---
title: Geth(10) The Blockchain
published: 2026-04-19
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

The previous chapters showed how blocks are built (Chapter 09) and how transactions are executed (Chapter 06). This chapter covers what happens *after* a block is ready: how geth inserts it into the chain, decides which fork is canonical, handles reorganizations, and persists everything to disk. The central type is `BlockChain` in `core/blockchain.go` — the orchestrator that ties together the consensus engine, the state database, the trie layer, and the on-disk storage.

---

## The Block Insertion Pipeline

When a new block arrives — whether from the network, the consensus layer's `NewPayload`, or local block building — it travels through this pipeline:

```
 InsertChain(blocks)
      |
      v
 1. Sanity checks          ── contiguous? linked? chain not stopped?
      |
      v
 2. Parallel header verify ── engine.VerifyHeaders() runs concurrently
      |
      v
 3. Skip known blocks      ── already in DB and behind current head?
      |
      v
 4. For each new block:
      |
      +-- ProcessBlock()
      |     |
      |     +-- a. Create StateDB from parent root
      |     +-- b. Prefetch state in background goroutine
      |     +-- c. processor.Process()     ── execute all transactions
      |     +-- d. validator.ValidateState() ── check roots, gas, bloom
      |     |
      |     +-- e. writeBlockAndSetHead()
      |           |
      |           +-- writeBlockWithState() ── persist block + state
      |           +-- reorg() (if needed)   ── switch canonical chain
      |           +-- writeHeadBlock()      ── update head pointers
      |           +-- emit events           ── ChainEvent, logs, ChainHeadEvent
      |
      v
 5. Fire accumulated ChainHeadEvent
```

Each stage is covered in detail below.

---

## The BlockChain Struct

The `BlockChain` struct is the central manager for the canonical chain. It holds references to every subsystem that touches block storage and validation:

```go
// core/blockchain.go

type BlockChain struct {
    chainConfig *params.ChainConfig // Chain & network configuration
    cfg         *BlockChainConfig   // Blockchain configuration

    db            ethdb.Database                   // Low level persistent database
    snaps         *snapshot.Tree                   // Snapshot tree for fast trie leaf access
    triegc        *prque.Prque[int64, common.Hash] // Priority queue mapping block numbers to tries to gc
    gcproc        time.Duration                    // Accumulates canonical block processing for trie dumping
    lastWrite     uint64                           // Last block when the state was flushed
    flushInterval atomic.Int64                     // Time interval after which to flush a state
    triedb        *triedb.Database                 // Trie node database handler
    statedb       *state.CachingDB                 // State database with caching
    txIndexer     *txIndexer                       // Transaction indexer (optional)

    hc               *HeaderChain
    rmLogsFeed       event.Feed
    chainFeed        event.Feed
    chainHeadFeed    event.Feed
    logsFeed         event.Feed
    blockProcFeed    event.Feed
    // ...
    genesisBlock     *types.Block

    chainmu *syncx.ClosableMutex // Synchronizes chain write operations

    currentBlock      atomic.Pointer[types.Header] // Current head of the chain
    currentSnapBlock  atomic.Pointer[types.Header] // Current head of snap-sync
    currentFinalBlock atomic.Pointer[types.Header] // Latest (consensus) finalized block
    currentSafeBlock  atomic.Pointer[types.Header] // Latest (consensus) safe block

    bodyCache     *lru.Cache[common.Hash, *types.Body]
    bodyRLPCache  *lru.Cache[common.Hash, rlp.RawValue]
    receiptsCache *lru.Cache[common.Hash, []*types.Receipt]
    blockCache    *lru.Cache[common.Hash, *types.Block]
    txLookupCache *lru.Cache[common.Hash, txLookup]

    stopping      atomic.Bool // true when chain is stopped
    procInterrupt atomic.Bool // interrupt signaler for block processing

    engine     consensus.Engine
    validator  Validator
    prefetcher Prefetcher
    processor  Processor
    // ...
}
```

The fields group into several roles:

| Role | Key Fields |
|------|-----------|
| **Persistence** | `db` (key-value store), `triedb` (trie nodes), `statedb` (state cache) |
| **Chain heads** | `currentBlock`, `currentSnapBlock`, `currentFinalBlock`, `currentSafeBlock` — all `atomic.Pointer[types.Header]` |
| **Concurrency** | `chainmu` (write lock), `stopping` / `procInterrupt` (shutdown signals) |
| **Caches** | LRU caches for bodies, receipts, blocks, tx lookups |
| **Events** | `chainFeed`, `chainHeadFeed`, `logsFeed`, `rmLogsFeed` — pub-sub feeds |
| **Validation** | `engine` (consensus), `validator` (block/state), `processor` (tx execution) |
| **GC** | `triegc` (priority queue), `gcproc`, `lastWrite`, `flushInterval` |

The four head pointers deserve special attention. They are all `atomic.Pointer` so readers can access them lock-free:

- **`currentBlock`** — the latest fully-validated block. This is the canonical chain tip.
- **`currentSnapBlock`** — the latest block whose state was downloaded during snap sync (may be ahead of `currentBlock` during sync).
- **`currentFinalBlock`** — the latest block marked as finalized by the consensus layer. Once finalized, this block and its ancestors will never be reverted.
- **`currentSafeBlock`** — the latest block the consensus layer considers safe (very unlikely to be reverted, but not yet finalized).

---

## Initialization: NewBlockChain

The constructor wires together every component:

```go
// core/blockchain.go

func NewBlockChain(db ethdb.Database, genesis *Genesis, engine consensus.Engine,
    cfg *BlockChainConfig) (*BlockChain, error) {
    // ...
    // 1. Open trie database
    triedb := triedb.NewDatabase(db, cfg.triedbConfig(enableVerkle))

    // 2. Write or verify genesis block
    chainConfig, genesisHash, compatErr, err := SetupGenesisBlockWithOverride(
        db, triedb, genesis, cfg.Overrides)

    // 3. Allocate the BlockChain struct with LRU caches
    bc := &BlockChain{
        chainConfig:   chainConfig,
        db:            db,
        triedb:        triedb,
        triegc:        prque.New[int64, common.Hash](nil),
        chainmu:       syncx.NewClosableMutex(),
        bodyCache:     lru.NewCache[common.Hash, *types.Body](bodyCacheLimit),
        receiptsCache: lru.NewCache[common.Hash, []*types.Receipt](receiptsCacheLimit),
        blockCache:    lru.NewCache[common.Hash, *types.Block](blockCacheLimit),
        txLookupCache: lru.NewCache[common.Hash, txLookup](txLookupCacheLimit),
        engine:        engine,
        // ...
    }

    // 4. Create the header chain
    bc.hc, _ = NewHeaderChain(db, chainConfig, engine, bc.insertStopped)

    // 5. Set up state database, validator, prefetcher, processor
    bc.statedb = state.NewDatabase(bc.triedb, nil)
    bc.validator = NewBlockValidator(chainConfig, bc)
    bc.prefetcher = newStatePrefetcher(chainConfig, bc.hc)
    bc.processor = NewStateProcessor(bc.hc)

    // 6. Restore chain state from disk
    bc.loadLastState()

    // 7. Recover if head state is missing
    // 8. Set up snapshots
    // 9. Start optional services (tx indexer, state size tracker)
    // ...
}
```

Walking through the key steps:

- **Step 1** opens the trie database with the configured scheme (hash-based or path-based — see Chapter 05).
- **Step 2** calls `SetupGenesisBlockWithOverride()`, which writes the genesis block if the database is empty or verifies it matches the stored one.
- **Steps 3–5** create the struct and wire in the header chain, state database, validator, and processor. The `HeaderChain` (covered later in this chapter) manages header storage and is embedded within `BlockChain`.
- **Step 6** calls `loadLastState()` which reads the persisted head markers from the database and restores `currentBlock`, `currentSnapBlock`, `currentFinalBlock`, and `currentSafeBlock`.
- **Step 7** handles the case where the head block's state is missing (e.g., after a crash). It rewinds the chain to a block whose state is available.

---

## InsertChain: The Public Entry Point

`InsertChain()` is the main entry point for adding blocks to the chain. Both the downloader (during sync) and the Engine API (for new payloads) ultimately call this:

```go
// core/blockchain.go

func (bc *BlockChain) InsertChain(chain types.Blocks) (int, error) {
    if len(chain) == 0 {
        return 0, nil
    }
    // Verify the chain is contiguous and properly linked
    for i := 1; i < len(chain); i++ {
        block, prev := chain[i], chain[i-1]
        if block.NumberU64() != prev.NumberU64()+1 || block.ParentHash() != prev.Hash() {
            return 0, fmt.Errorf("non contiguous insert: ...")
        }
    }
    // Acquire the chain write lock
    if !bc.chainmu.TryLock() {
        return 0, errChainStopped
    }
    defer bc.chainmu.Unlock()

    _, n, err := bc.insertChain(chain, true, false)
    return n, err
}
```

The method does two things before delegating to the internal `insertChain()`:

1. **Contiguity check** — every block must be the direct child of the previous one (consecutive numbers, matching parent hash).
2. **Lock acquisition** — `chainmu.TryLock()` ensures only one goroutine writes to the chain at a time. If the chain is stopped, `TryLock()` returns false immediately rather than blocking forever.

The `setHead` parameter passed as `true` means `insertChain` will update the canonical head. There is also `InsertBlockWithoutSetHead()` which passes `false` — this is used by the Engine API to insert a block without changing the canonical tip, letting `SetCanonical()` do that separately.

---

## insertChain: The Internal Implementation

The internal `insertChain()` method does the heavy lifting. It processes blocks one by one, verifying headers in parallel:

```go
// core/blockchain.go  (simplified)

func (bc *BlockChain) insertChain(chain types.Blocks, setHead bool, makeWitness bool) (
    *stateless.Witness, int, error) {

    // Start parallel signature recovery for all blocks
    SenderCacher().RecoverFromBlocks(types.MakeSigner(bc.chainConfig, ...), chain)

    var lastCanon *types.Block
    defer func() {
        if lastCanon != nil && bc.CurrentBlock().Hash() == lastCanon.Hash() {
            bc.chainHeadFeed.Send(ChainHeadEvent{Header: lastCanon.Header()})
        }
    }()

    // Start the parallel header verifier
    headers := make([]*types.Header, len(chain))
    for i, block := range chain {
        headers[i] = block.Header()
    }
    abort, results := bc.engine.VerifyHeaders(bc, headers)
    defer close(abort)

    // Create an iterator that pairs blocks with verification results
    it := newInsertIterator(chain, results, bc.validator)
    block, err := it.next()

    // Skip known blocks that are behind the current head
    // ...

    // Main processing loop
    for ; block != nil && (err == nil || errors.Is(err, ErrKnownBlock)); block, err = it.next() {
        if bc.insertStopped() {
            break
        }
        parent := it.previous()
        if parent == nil {
            parent = bc.GetHeader(block.ParentHash(), block.NumberU64()-1)
        }

        // Execute and validate the block
        res, err := bc.ProcessBlock(parent.Root, block, setHead, makeWitness)
        if err != nil {
            return nil, it.index, err
        }
        // Track stats and report progress
        // ...
    }
    return witness, it.index, err
}
```

Three things happen concurrently to maximize throughput:

1. **Sender recovery** — `SenderCacher().RecoverFromBlocks()` starts ECDSA signature recovery for all transactions across all blocks in background goroutines. This is the most CPU-intensive part of validation.
2. **Header verification** — `engine.VerifyHeaders()` launches a goroutine that checks consensus rules for all headers. Results arrive via a channel, consumed by the `insertIterator`.
3. **Sequential block execution** — the main loop processes blocks one at a time (state execution cannot be parallelized since each block depends on the previous state).

The deferred function at the end fires a single `ChainHeadEvent` if the chain progressed — this avoids emitting one event per block during batch imports.

### Handling Edge Cases

Before the main loop, `insertChain` handles several edge cases:

- **Known blocks behind the head** are skipped — they are already in the database and do not change the canonical chain.
- **Pruned ancestor** (`ErrPrunedAncestor`) — the parent block's state has been garbage-collected. If `setHead` is true, the block is inserted as a sidechain; otherwise `recoverAncestors()` re-executes ancestors to rebuild the state.
- **Clique blocks** get special handling because Clique's proof-of-authority mechanism allows blocks to share state, so a known block might still need re-import to fill in snapshots.

---

## ProcessBlock: Execute and Persist

`ProcessBlock()` is where a single block is executed and, if valid, written to storage:

```go
// core/blockchain.go  (simplified)

func (bc *BlockChain) ProcessBlock(parentRoot common.Hash, block *types.Block,
    setHead bool, makeWitness bool) (*blockProcessingResult, error) {

    // 1. Create state database from parent root
    statedb, err := state.New(parentRoot, bc.statedb)

    // (If prefetching is enabled, a throwaway state runs transactions
    //  in parallel to warm up the trie cache)

    // 2. Execute all transactions in the block
    res, err := bc.processor.Process(block, statedb, bc.cfg.VmConfig)

    // 3. Validate the execution results against the header
    err = bc.validator.ValidateState(block, statedb, res, false)

    // 4. Write block and state to disk
    if !setHead {
        err = bc.writeBlockWithState(block, res.Receipts, statedb)
    } else {
        status, err = bc.writeBlockAndSetHead(block, res.Receipts, res.Logs, statedb, false)
    }
    return &blockProcessingResult{usedGas: res.GasUsed, procTime: ..., status: status}, nil
}
```

Walking through each step:

- **Step 1** creates a `StateDB` from the parent block's state root. If prefetching is enabled (the default), two readers are created — one for a throwaway prefetch execution and one for the real execution — sharing a cache so the real execution benefits from the prefetcher's trie node lookups.
- **Step 2** calls `processor.Process()` which executes every transaction in the block (see Chapter 06 for the full execution pipeline).
- **Step 3** calls `validator.ValidateState()` which checks that `GasUsed`, the bloom filter, the receipt root, the requests hash (Prague), and the state root all match the block header (see Chapter 09 for validation details).
- **Step 4** persists the block. The `setHead` flag controls whether the canonical head is updated. When called from `InsertChain`, `setHead` is `true`. When called from the Engine API's `InsertBlockWithoutSetHead`, it is `false` — the block is stored but the head is not moved until the consensus layer explicitly calls `SetCanonical()`.

---

## writeBlockWithState: Persisting Block and State

Once a block passes validation, it must be durably stored:

```go
// core/blockchain.go

func (bc *BlockChain) writeBlockWithState(block *types.Block, receipts []*types.Receipt,
    statedb *state.StateDB) error {

    // 1. Write block data atomically
    blockBatch := bc.db.NewBatch()
    rawdb.WriteBlock(blockBatch, block)
    rawdb.WriteReceipts(blockBatch, block.Hash(), block.NumberU64(), receipts)
    rawdb.WritePreimages(blockBatch, statedb.Preimages())
    blockBatch.Write()

    // 2. Commit state changes from memory into the trie database
    root, stateUpdate, err := statedb.CommitWithUpdate(block.NumberU64(), ...)

    // 3. Trie garbage collection (hash-based scheme only)
    if bc.triedb.Scheme() == rawdb.PathScheme {
        return nil // path-based scheme handles GC internally
    }
    if bc.cfg.ArchiveMode {
        return bc.triedb.Commit(root, false) // archive: always flush
    }
    // Full node: selective garbage collection
    bc.triedb.Reference(root, common.Hash{})
    bc.triegc.Push(root, -int64(block.NumberU64()))
    // ...
}
```

The persistence splits into three layers:

**Layer 1 — Block data** is written in an atomic batch: the block (header + body), receipts, and any preimage mappings. The batch ensures all components are either fully written or not written at all.

**Layer 2 — State commit** flushes dirty state from `StateDB`'s in-memory maps into the trie database (see Chapter 04). `CommitWithUpdate()` returns the state root (which should match the block header's `Root` field) and a state update record for the size tracker.

**Layer 3 — Trie garbage collection** manages which trie nodes stay in memory and which get flushed to disk. This only applies to the hash-based trie scheme (the path-based scheme handles GC internally):

- **Archive nodes** flush every block's trie to disk immediately.
- **Full nodes** keep recent tries in memory and use a priority queue (`triegc`) to track which tries can be garbage-collected. Tries are flushed when either the memory limit (`TrieDirtyLimit`) is exceeded or enough processing time has accumulated (`flushInterval`). The constant `TriesInMemory` (128) controls how far back tries are retained — tries older than `HEAD - 128` are eligible for dereference.

---

## writeBlockAndSetHead: Updating the Canonical Chain

When a block should become the new canonical head, `writeBlockAndSetHead()` handles both persistence and head updates:

```go
// core/blockchain.go

func (bc *BlockChain) writeBlockAndSetHead(block *types.Block, receipts []*types.Receipt,
    logs []*types.Log, state *state.StateDB, emitHeadEvent bool) (WriteStatus, error) {

    // 1. Persist block and state
    if err := bc.writeBlockWithState(block, receipts, state); err != nil {
        return NonStatTy, err
    }

    // 2. Reorganise if the parent is not the current head
    currentBlock := bc.CurrentBlock()
    if block.ParentHash() != currentBlock.Hash() {
        if err := bc.reorg(currentBlock, block.Header()); err != nil {
            return NonStatTy, err
        }
    }

    // 3. Update head pointers
    bc.writeHeadBlock(block)

    // 4. Emit events
    bc.chainFeed.Send(ChainEvent{
        Header: block.Header(), Receipts: receipts, Transactions: block.Transactions(),
    })
    if len(logs) > 0 {
        bc.logsFeed.Send(logs)
    }
    if emitHeadEvent {
        bc.chainHeadFeed.Send(ChainHeadEvent{Header: block.Header()})
    }
    return CanonStatTy, nil
}
```

The critical decision is in step 2: if the new block's parent is *not* the current head, a reorganization is needed. This happens when a fork produces a block that becomes the preferred chain tip. The `reorg()` function (covered in the next section) switches the canonical chain from the old fork to the new one.

Step 3 calls `writeHeadBlock()`, which updates both on-disk markers and in-memory atomic pointers:

```go
// core/blockchain.go

func (bc *BlockChain) writeHeadBlock(block *types.Block) {
    batch := bc.db.NewBatch()
    rawdb.WriteHeadHeaderHash(batch, block.Hash())
    rawdb.WriteHeadFastBlockHash(batch, block.Hash())
    rawdb.WriteCanonicalHash(batch, block.Hash(), block.NumberU64())
    rawdb.WriteTxLookupEntriesByBlock(batch, block)
    rawdb.WriteHeadBlockHash(batch, block.Hash())
    batch.Write()

    bc.hc.SetCurrentHeader(block.Header())
    bc.currentSnapBlock.Store(block.Header())
    bc.currentBlock.Store(block.Header())
}
```

The database batch writes five things atomically: the head header hash, the head fast (snap) block hash, the canonical number-to-hash mapping, transaction lookup entries (hash → block number), and the head block hash. Then the in-memory pointers are updated. Because the pointers are `atomic.Pointer`, readers see the new head immediately without needing the chain lock.

---

## Chain Reorganization

A reorg occurs when a new block's parent is not the current chain tip — meaning the chain must switch from one fork to another. The `reorg()` function handles this:

```
 Old canonical chain          New canonical chain
       |                            |
   block A5                     block B5  <-- new head
       |                            |
   block A4                     block B4
       |                            |
   block A3  (current head)     block B3
       \                           /
        +--- common ancestor -----+
                  block 2
```

The algorithm:

```go
// core/blockchain.go  (simplified)

func (bc *BlockChain) reorg(oldHead *types.Header, newHead *types.Header) error {
    var (
        newChain    []*types.Header
        oldChain    []*types.Header
        commonBlock *types.Header
    )
    // Step 1: Reduce the longer chain to match the shorter one's height
    if oldHead.Number.Uint64() > newHead.Number.Uint64() {
        for ; oldHead.Number.Uint64() != newHead.Number.Uint64(); oldHead = bc.GetHeader(...) {
            oldChain = append(oldChain, oldHead)
        }
    } else {
        for ; newHead.Number.Uint64() != oldHead.Number.Uint64(); newHead = bc.GetHeader(...) {
            newChain = append(newChain, newHead)
        }
    }
    // Step 2: Walk both chains back until hashes match
    for {
        if oldHead.Hash() == newHead.Hash() {
            commonBlock = oldHead
            break
        }
        oldChain = append(oldChain, oldHead)
        newChain = append(newChain, newHead)
        oldHead = bc.GetHeader(oldHead.ParentHash, ...)
        newHead = bc.GetHeader(newHead.ParentHash, ...)
    }
    // Step 3: Undo old blocks, apply new blocks
    // ...
}
```

After finding the common ancestor, the function performs the actual switch:

1. **Emit removed logs** — iterates through the old chain in forward order, collects logs from removed blocks, and sends them via `rmLogsFeed` as `RemovedLogsEvent`.
2. **Collect deleted transactions** — iterates through the old chain, gathering all transaction hashes that will be removed from the canonical index.
3. **Apply new blocks** — iterates through the new chain in forward order, calling `writeHeadBlock()` for each block to update canonical hash mappings and tx lookup entries. Collects reborn logs and emits them via `logsFeed`.
4. **Clean up indexes** — deletes tx lookup entries for transactions that were in the old chain but not the new one (using `HashDifference(deletedTxs, rebirthTxs)`). Removes stale canonical hash mappings above the new head.
5. **Purge tx lookup cache** — the LRU cache may hold stale entries from the old chain.

The entire tx-lookup mutation is protected by `txLookupLock`, a separate `sync.RWMutex` that ensures API readers see a consistent view of the transaction index during the reorg.

Large reorgs (more than 63 blocks) trigger a `log.Warn("Large chain reorg detected")` to alert operators.

---

## Head Management: Finalized and Safe Blocks

Post-Merge, the consensus layer manages two additional head markers beyond the canonical tip:

```go
// core/blockchain.go

func (bc *BlockChain) SetFinalized(header *types.Header) {
    bc.currentFinalBlock.Store(header)
    if header != nil {
        rawdb.WriteFinalizedBlockHash(bc.db, header.Hash())
    } else {
        rawdb.WriteFinalizedBlockHash(bc.db, common.Hash{})
    }
}

func (bc *BlockChain) SetSafe(header *types.Header) {
    bc.currentSafeBlock.Store(header)
    // ...
}
```

- **`SetFinalized()`** is called by the Engine API's `ForkchoiceUpdated`. A finalized block is guaranteed to never be reverted by the consensus protocol. It is both stored to the atomic pointer (for fast reads) and persisted to disk (so it survives restarts).
- **`SetSafe()`** marks a block as safe — very unlikely to be reverted but not yet finalized. Unlike finalized, the safe block is **not** persisted to disk. On restart, `loadLastState()` sets safe equal to finalized.

The reader methods are lock-free:

```go
// core/blockchain_reader.go

func (bc *BlockChain) CurrentBlock() *types.Header      { return bc.currentBlock.Load() }
func (bc *BlockChain) CurrentSnapBlock() *types.Header   { return bc.currentSnapBlock.Load() }
func (bc *BlockChain) CurrentFinalBlock() *types.Header  { return bc.currentFinalBlock.Load() }
func (bc *BlockChain) CurrentSafeBlock() *types.Header   { return bc.currentSafeBlock.Load() }
```

### SetHead: Rewinding the Chain

`SetHead()` rewinds the chain to a specific block number, deleting blocks and state above that point:

```go
// core/blockchain.go

func (bc *BlockChain) SetHead(head uint64) error {
    if _, err := bc.setHeadBeyondRoot(head, 0, common.Hash{}, false); err != nil {
        return err
    }
    bc.chainHeadFeed.Send(ChainHeadEvent{Header: bc.CurrentBlock()})
    return nil
}
```

There is also `SetHeadWithTimestamp()` which rewinds to the latest block at or before a given timestamp. Both delegate to `setHeadBeyondRoot()`, which handles the complexity of deleting data while preserving consistency between the key-value store and the freezer (ancient) database.

---

## The Engine API Path: InsertBlockWithoutSetHead

The Engine API uses a two-step approach to inserting blocks:

```go
// core/blockchain.go

func (bc *BlockChain) InsertBlockWithoutSetHead(block *types.Block, makeWitness bool) (
    *stateless.Witness, error) {
    // ...
    witness, _, err := bc.insertChain(types.Blocks{block}, false, makeWitness)
    return witness, err
}

func (bc *BlockChain) SetCanonical(head *types.Block) (common.Hash, error) {
    // Recover state if missing
    if !bc.HasState(head.Root()) {
        bc.recoverAncestors(head, false)
    }
    // Run reorg if needed and update head
    if head.ParentHash() != bc.CurrentBlock().Hash() {
        bc.reorg(bc.CurrentBlock(), head.Header())
    }
    bc.writeHeadBlock(head)
    // ...
}
```

This split exists because in post-Merge Ethereum, the consensus layer tells geth *which* block to build on via `ForkchoiceUpdated`, separately from *which* blocks to validate via `NewPayload`. A block might be validated (inserted without setting head) long before the consensus layer decides it should be canonical.

---

## The HeaderChain

`HeaderChain` manages header-only storage and is embedded within `BlockChain`. During snap sync, geth downloads headers first (via the skeleton syncer) before fetching bodies and state, so the header chain can be ahead of the full block chain.

```go
// core/headerchain.go

type HeaderChain struct {
    config        *params.ChainConfig
    chainDb       ethdb.Database
    genesisHeader *types.Header

    currentHeader     atomic.Pointer[types.Header]
    currentHeaderHash common.Hash

    headerCache *lru.Cache[common.Hash, *types.Header]
    numberCache *lru.Cache[common.Hash, uint64]

    procInterrupt func() bool
    engine        consensus.Engine
}
```

`HeaderChain` provides:

- **`GetHeader(hash, number)`** / **`GetHeaderByNumber(n)`** — read headers from cache or database.
- **`InsertHeaderChain(headers)`** — validates and inserts a batch of headers. Used during snap sync's skeleton phase.
- **`Reorg(headers)`** — reorganizes the header chain to point to a new set of canonical headers.
- **`SetCurrentHeader(header)`** — updates the in-memory head header pointer.

During normal block insertion, `BlockChain` calls `hc.SetCurrentHeader()` as part of `writeHeadBlock()`. During snap sync, the header chain can advance independently via `InsertHeaderChain()`.

---

## The Genesis Block

Every chain starts with a genesis block — block number 0, with no parent, containing the initial state allocation (pre-funded accounts, contract code, etc.). The `Genesis` struct defines this initial state:

```go
// core/genesis.go

type Genesis struct {
    Config     *params.ChainConfig `json:"config"`
    Nonce      uint64              `json:"nonce"`
    Timestamp  uint64              `json:"timestamp"`
    ExtraData  []byte              `json:"extraData"`
    GasLimit   uint64              `json:"gasLimit"   gencodec:"required"`
    Difficulty *big.Int            `json:"difficulty" gencodec:"required"`
    Mixhash    common.Hash         `json:"mixHash"`
    Coinbase   common.Address      `json:"coinbase"`
    Alloc      types.GenesisAlloc  `json:"alloc"      gencodec:"required"`

    BaseFee       *big.Int        `json:"baseFeePerGas"` // EIP-1559
    ExcessBlobGas *uint64         `json:"excessBlobGas"` // EIP-4844
    BlobGasUsed   *uint64         `json:"blobGasUsed"`   // EIP-4844
    // ...
}
```

The `Alloc` field is a `GenesisAlloc` (a map of addresses to accounts), defining the initial balances, code, nonces, and storage for all pre-existing accounts. For mainnet, this includes the crowdsale allocation, system contracts, and other initial state.

### Genesis Commit

When a node starts for the first time, the genesis block is committed to the database:

```go
// core/genesis.go

func (g *Genesis) Commit(db ethdb.Database, triedb *triedb.Database) (*types.Block, error) {
    if g.Number != 0 {
        return nil, errors.New("can't commit genesis block with number > 0")
    }
    // Validate config and signers
    // ...

    // Flush genesis allocations into the trie → compute state root
    root, err := flushAlloc(&g.Alloc, triedb)

    // Build the genesis block with the computed state root
    block := g.toBlockWithRoot(root)

    // Write everything atomically
    batch := db.NewBatch()
    rawdb.WriteGenesisStateSpec(batch, block.Hash(), blob) // JSON alloc spec
    rawdb.WriteBlock(batch, block)                          // header + body
    rawdb.WriteReceipts(batch, block.Hash(), 0, nil)        // empty receipts
    rawdb.WriteCanonicalHash(batch, block.Hash(), 0)        // number 0 → hash
    rawdb.WriteHeadBlockHash(batch, block.Hash())           // head block
    rawdb.WriteHeadFastBlockHash(batch, block.Hash())       // head fast block
    rawdb.WriteHeadHeaderHash(batch, block.Hash())          // head header
    rawdb.WriteChainConfig(batch, block.Hash(), config)     // chain config
    return block, batch.Write()
}
```

The `flushAlloc()` function iterates over every account in `Alloc`, inserts them into a trie, and returns the state root. This root becomes the genesis block's `Root` field. The entire genesis state — block, receipts, canonical mappings, head markers, and chain config — is written in a single atomic batch.

### SetupGenesisBlock

`NewBlockChain` calls `SetupGenesisBlockWithOverride()` which decides what to do based on the database state:

|                      | `genesis == nil`     | `genesis != nil`          |
|----------------------|---------------------|--------------------------|
| **DB has no genesis** | Use mainnet default | Commit provided genesis  |
| **DB has genesis**    | Load from DB        | Verify compatible, use DB |

If the database already has a genesis block and the provided one conflicts, a `GenesisMismatchError` is returned. If the chain config is updated (e.g., a new fork is scheduled), the stored config is updated as long as the fork activation points are above the current chain head.

---

## Startup State Recovery

When `loadLastState()` restores the chain on startup, it must handle the case where the head block's state trie is not available (e.g., after a crash before the trie was flushed):

```go
// core/blockchain.go  (inside loadLastState, simplified)

func (bc *BlockChain) loadLastState() error {
    head := rawdb.ReadHeadBlockHash(bc.db)
    if head == (common.Hash{}) {
        return bc.Reset() // empty database
    }
    headBlock := bc.GetBlockByHash(head)
    if headBlock == nil {
        return bc.Reset() // corrupt database
    }
    bc.currentBlock.Store(headBlock.Header())

    // Restore snap sync head
    bc.currentSnapBlock.Store(headBlock.Header())
    if head := rawdb.ReadHeadFastBlockHash(bc.db); head != (common.Hash{}) {
        if block := bc.GetBlockByHash(head); block != nil {
            bc.currentSnapBlock.Store(block.Header())
        }
    }

    // Restore finalized and safe blocks
    if head := rawdb.ReadFinalizedBlockHash(bc.db); head != (common.Hash{}) {
        if block := bc.GetBlockByHash(head); block != nil {
            bc.currentFinalBlock.Store(block.Header())
            bc.currentSafeBlock.Store(block.Header()) // safe defaults to finalized
        }
    }
    return nil
}
```

After `loadLastState()`, the constructor checks whether the head block's state is actually available. If not, it calls `setHeadBeyondRoot()` to rewind the chain until it finds a block with available state. This makes geth resilient to crashes — at worst, it re-executes a few recent blocks on the next startup.

---

## Events

The `BlockChain` emits events through `event.Feed` channels that other subsystems subscribe to:

```go
// core/events.go

type ChainEvent struct {
    Header       *types.Header
    Receipts     []*types.Receipt
    Transactions []*types.Transaction
}

type ChainHeadEvent struct {
    Header *types.Header
}

type RemovedLogsEvent struct {
    Logs []*types.Log
}
```

| Feed | Event Type | When Emitted |
|------|-----------|-------------|
| `chainFeed` | `ChainEvent` | Every new canonical block |
| `chainHeadFeed` | `ChainHeadEvent` | When the chain tip changes (batched per `insertChain` call) |
| `logsFeed` | `[]*types.Log` | New logs from canonical blocks or reborn blocks during reorg |
| `rmLogsFeed` | `RemovedLogsEvent` | Logs from blocks removed during a reorg |
| `blockProcFeed` | `bool` | `true` when block processing starts, `false` when it ends |

The transaction pool subscribes to `ChainHeadEvent` to re-validate pending transactions against the new state. The `eth` handler subscribes to `ChainHeadEvent` to broadcast new blocks to peers. The RPC layer uses `chainFeed` and `logsFeed` to serve `eth_subscribe` notifications.

---

## Graceful Shutdown

`Stop()` ensures all in-memory state is safely persisted before the node exits:

```go
// core/blockchain.go

func (bc *BlockChain) Stop() {
    bc.stopWithoutSaving()

    // Journal snapshots to disk
    if bc.snaps != nil {
        bc.snaps.Journal(bc.CurrentBlock().Root)
        bc.snaps.Release()
    }
    if bc.triedb.Scheme() == rawdb.PathScheme {
        bc.triedb.Journal(bc.CurrentBlock().Root)
    } else {
        // Hash-based scheme: commit recent tries
        if !bc.cfg.ArchiveMode {
            for _, offset := range []uint64{0, 1, state.TriesInMemory - 1} {
                if number := bc.CurrentBlock().Number.Uint64(); number > offset {
                    recent := bc.GetBlockByNumber(number - offset)
                    bc.triedb.Commit(recent.Root(), true)
                }
            }
            // Dereference all remaining GC entries
            for !bc.triegc.Empty() {
                bc.triedb.Dereference(bc.triegc.PopItem())
            }
        }
    }
    bc.triedb.Close()
}
```

The shutdown strategy depends on the trie scheme:

- **Path-based scheme** — journals the current trie state to disk. On restart, the journal is replayed to restore the in-memory state.
- **Hash-based scheme** — commits three specific tries to disk: `HEAD`, `HEAD-1`, and `HEAD-127`. This covers three restart scenarios: normal restart (HEAD state is there), uncle-reorg (HEAD-1 is there), and worst-case rewind (HEAD-127 limits re-execution to 127 blocks). The remaining GC queue entries are all dereferenced to free memory.

The `stopWithoutSaving()` helper handles the non-persistence part of shutdown: setting the `stopping` flag, closing the tx indexer, unsubscribing events, signaling `procInterrupt` to abort any in-progress block insertion, and closing the `chainmu` mutex.

---

## WriteStatus: Canonical vs. Sidechain

Every block insertion returns a `WriteStatus` indicating the block's relationship to the canonical chain:

```go
// core/blockchain.go

type WriteStatus byte

const (
    NonStatTy   WriteStatus = iota // Unknown or non-canonical
    CanonStatTy                     // Part of the canonical chain
    SideStatTy                      // Stored but not canonical (fork)
)
```

`CanonStatTy` means the block extended or replaced the canonical chain tip. `SideStatTy` means the block was stored (it might become canonical later during a reorg) but did not change the current head. The `insertChain` loop uses this status for logging — canonical blocks log at `Debug` level as "Inserted new block", while side chain blocks log as "Inserted forked block".
