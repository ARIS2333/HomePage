---
title: Geth(9) Block Production and Consensus
published: 2026-04-18
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

After the Merge, Ethereum's execution layer (geth) no longer decides *when* to produce a block or *which* fork is canonical. That authority moved to the consensus layer (the beacon chain client). Geth's role is to **build block payloads on demand** and **validate them against execution rules**. This chapter traces how a block is built — from the consensus layer's request through transaction selection, execution, and final assembly — and how the consensus engine interface abstracts the rules that govern block validity.

---

## The Engine API: CL Drives, EL Builds

Block production in post-Merge Ethereum is a two-phase handshake between the consensus layer (CL) and execution layer (EL), communicated via the **Engine API** (`eth/catalyst/api.go`):

```
 Consensus Layer (beacon client)              Execution Layer (geth)
        |                                              |
        |  1. engine_forkchoiceUpdatedV3               |
        |     { headHash, safeHash, finalHash,         |
        |       payloadAttributes }                    |
        | -------------------------------------------> |
        |                                              |  - Update canonical head
        |                                              |  - Start building payload
        |     { payloadStatus: VALID,                  |    (empty block immediately,
        |       payloadId: 0x... }                     |     then fill with txs)
        | <------------------------------------------- |
        |                                              |
        |  ... time passes (CL waits for slot) ...     |
        |                                              |
        |  2. engine_getPayloadV4                      |
        |     { payloadId: 0x... }                     |
        | -------------------------------------------> |
        |                                              |  - Stop building, return
        |     { executionPayload, blockValue,          |    best payload so far
        |       blobsBundle, ... }                     |
        | <------------------------------------------- |
        |                                              |
        |  3. (CL signs and broadcasts block)          |
        |                                              |
        |  4. engine_newPayloadV4                      |
        |     { executionPayload }                     |
        | -------------------------------------------> |
        |                                              |  - Validate and insert block
        |     { payloadStatus: VALID }                 |
        | <------------------------------------------- |
```

The versioning differs per endpoint. `ForkchoiceUpdated` has three versions (V1 for Paris, V2 for Shanghai, V3 for Cancun and later). `GetPayload` and `NewPayload` track fork boundaries more granularly: V1 for Paris, V2 for Shanghai, V3 for Cancun, V4 for Prague (adds requests), and `GetPayload` extends to V5 for Osaka (cell proofs in blobs bundle).

### Step 1: ForkchoiceUpdated

When the CL calls `ForkchoiceUpdatedV3`, geth's `ConsensusAPI.forkchoiceUpdated()` does two things:

1. **Updates the canonical head** — sets the head, safe, and finalized block pointers on the blockchain.
2. **Starts payload building** — if `payloadAttributes` is non-nil (meaning the CL wants this validator to propose), it calls `Miner.BuildPayload()`:

```go
// eth/catalyst/api.go  (inside forkchoiceUpdated)

if payloadAttributes != nil {
    args := &miner.BuildPayloadArgs{
        Parent:       update.HeadBlockHash,
        Timestamp:    payloadAttributes.Timestamp,
        FeeRecipient: payloadAttributes.SuggestedFeeRecipient,
        Random:       payloadAttributes.Random,
        Withdrawals:  payloadAttributes.Withdrawals,
        BeaconRoot:   payloadAttributes.BeaconRoot,
        Version:      payloadVersion,
    }
    id := args.Id()
    payload, err := api.eth.Miner().BuildPayload(args, payloadWitness)
    if err != nil {
        return valid(nil), engine.InvalidPayloadAttributes.With(err)
    }
    api.localBlocks.put(id, payload)
    return valid(&id), nil
}
```

The `BuildPayloadArgs` struct carries everything the CL provides: parent hash, timestamp, fee recipient (coinbase), RANDAO randomness, withdrawals, and beacon root.

### Step 2: GetPayload

Later, the CL calls `GetPayloadV4` with the payload ID. Geth returns the best block built so far (the one with the highest total fees), along with the blob sidecar bundle and consensus-layer requests.

---

## The Miner and Payload Building

The `Miner` struct in `miner/miner.go` is geth's block builder. Despite the name (a legacy from proof-of-work), it no longer mines — it assembles payloads on demand.

```go
// miner/miner.go

type Miner struct {
    confMu      sync.RWMutex
    config      *Config
    chainConfig *params.ChainConfig
    engine      consensus.Engine
    txpool      *txpool.TxPool
    prio        []common.Address // Addresses to prioritize for tx inclusion
    chain       *core.BlockChain
    pending     *pending
    pendingMu   sync.Mutex
}
```

The `Config` controls block-building behaviour:

```go
// miner/miner.go

var DefaultConfig = Config{
    GasCeil:  60_000_000,
    GasPrice: big.NewInt(params.GWei / 1000),
    Recommit: 2 * time.Second,
}
```

- **`GasCeil`** — target gas limit for produced blocks (60M by default). The actual limit is computed by `CalcGasLimit()` which adjusts the parent's gas limit toward this target by at most 1/1024 per block.
- **`GasPrice`** — minimum gas tip for including a transaction.
- **`Recommit`** — how often to rebuild the payload with new transactions (2 seconds). Since the CL waits ~6 seconds (half a slot) before calling `GetPayload`, geth typically runs 3 build rounds.

### The buildPayload Strategy

`BuildPayload()` uses an **empty-then-fill** strategy:

```go
// miner/payload_building.go  (simplified)

func (miner *Miner) buildPayload(args *BuildPayloadArgs, witness bool) (*Payload, error) {
    // 1. Build an empty block immediately (no transactions)
    emptyParams := &generateParams{
        timestamp:   args.Timestamp,
        forceTime:   true,
        parentHash:  args.Parent,
        coinbase:    args.FeeRecipient,
        random:      args.Random,
        withdrawals: args.Withdrawals,
        beaconRoot:  args.BeaconRoot,
        noTxs:       true,
    }
    empty := miner.generateWork(emptyParams, witness)
    if empty.err != nil {
        return nil, empty.err
    }
    payload := newPayload(empty.block, empty.requests, empty.witness, args.Id())

    // 2. In a background goroutine, repeatedly rebuild with transactions
    go func() {
        timer := time.NewTimer(0) // fire immediately for first build
        endTimer := time.NewTimer(time.Second * 12) // slot timeout

        fullParams := &generateParams{ /* same as above, but noTxs: false */ }

        for {
            select {
            case <-timer.C:
                r := miner.generateWork(fullParams, witness)
                if r.err == nil {
                    payload.update(r, ...)
                }
                timer.Reset(miner.config.Recommit)
            case <-payload.stop:
                return // CL called GetPayload
            case <-endTimer.C:
                return // slot expired (12s)
            }
        }
    }()
    return payload, nil
}
```

The `Payload` struct holds both the empty block (always available) and the best full block (updated as better versions are built). When `Resolve()` is called (by `GetPayload`), it returns the full block if available, otherwise the empty one. This guarantees the validator never misses a slot — even if transaction execution is slow, there is always an empty block to propose.

Each `update()` call only replaces the current best if the new block has higher total fees.

---

## Block Assembly: generateWork

The core of block building is `generateWork()` in `miner/worker.go`. It follows a clear pipeline:

```
 prepareWork()                    ── build header, create state, run system calls
      |
      v
 fillTransactions()               ── pull from pool, order by tip, execute
      |
      v
 Collect consensus-layer requests ── deposits, withdrawal requests, consolidations
      |
      v
 engine.FinalizeAndAssemble()     ── apply withdrawals, compute state root, build block
```

### Step 1: prepareWork — Header and State

`prepareWork()` constructs the block header and initialises the execution environment:

```go
// miner/worker.go  (simplified)

func (miner *Miner) prepareWork(genParams *generateParams, witness bool) (*environment, error) {
    parent := miner.chain.CurrentBlock()
    if genParams.parentHash != (common.Hash{}) {
        block := miner.chain.GetBlockByHash(genParams.parentHash)
        parent = block.Header()
    }
    // Construct block header
    header := &types.Header{
        ParentHash: parent.Hash(),
        Number:     new(big.Int).Add(parent.Number, common.Big1),
        GasLimit:   core.CalcGasLimit(parent.GasLimit, miner.config.GasCeil),
        Time:       timestamp,
        Coinbase:   genParams.coinbase,
    }
    // Set EIP-1559 base fee
    if miner.chainConfig.IsLondon(header.Number) {
        header.BaseFee = eip1559.CalcBaseFee(miner.chainConfig, parent)
    }
    // Let the consensus engine set its fields (e.g., difficulty = 0 for PoS)
    miner.engine.Prepare(miner.chain, header)

    // Set Cancun fields (blob gas, beacon root)
    if miner.chainConfig.IsCancun(header.Number, header.Time) {
        excessBlobGas := eip4844.CalcExcessBlobGas(miner.chainConfig, parent, timestamp)
        header.BlobGasUsed = new(uint64)
        header.ExcessBlobGas = &excessBlobGas
        header.ParentBeaconRoot = genParams.beaconRoot
    }
    // Create execution environment: parent state + EVM instance
    env, err := miner.makeEnv(parent, header, genParams.coinbase, witness)

    // Run pre-block system calls
    if header.ParentBeaconRoot != nil {
        core.ProcessBeaconBlockRoot(*header.ParentBeaconRoot, env.evm) // EIP-4788
    }
    if miner.chainConfig.IsPrague(header.Number, header.Time) {
        core.ProcessParentBlockHash(header.ParentHash, env.evm)        // EIP-2935
    }
    return env, nil
}
```

Key points:

- The **gas limit** is adjusted toward `GasCeil` using `CalcGasLimit()`, which moves the parent's limit by at most 1/1024 per block.
- The **base fee** is computed from the parent's gas usage (see Chapter 06).
- `engine.Prepare()` sets consensus-specific header fields. For the beacon engine, this just sets `Difficulty = 0`.
- **System calls** are executed before any user transactions. `ProcessBeaconBlockRoot` (EIP-4788) stores the parent beacon block root in a system contract. `ProcessParentBlockHash` (EIP-2935) stores the parent block hash in a system contract for historical access.

The `environment` struct holds the in-progress block state:

```go
// miner/worker.go

type environment struct {
    signer   types.Signer
    state    *state.StateDB
    tcount   int            // tx count in cycle
    size     uint64         // block size so far
    gasPool  *core.GasPool  // remaining gas budget
    coinbase common.Address
    evm      *vm.EVM

    header   *types.Header
    txs      []*types.Transaction
    receipts []*types.Receipt
    sidecars []*types.BlobTxSidecar
    blobs    int
    witness  *stateless.Witness
}
```

### Step 2: fillTransactions — Transaction Selection

`fillTransactions()` pulls pending transactions from the pool and feeds them into the block:

```go
// miner/worker.go  (simplified)

func (miner *Miner) fillTransactions(interrupt *atomic.Int32, env *environment) error {
    tip := miner.config.GasPrice

    // Build a filter for the pool query
    filter := txpool.PendingFilter{
        MinTip: uint256.MustFromBig(tip),
    }
    if env.header.BaseFee != nil {
        filter.BaseFee = uint256.MustFromBig(env.header.BaseFee)
    }
    if env.header.ExcessBlobGas != nil {
        filter.BlobFee = uint256.MustFromBig(eip4844.CalcBlobFee(miner.chainConfig, env.header))
    }
    // Osaka adds per-tx gas cap and blob version filtering
    if miner.chainConfig.IsOsaka(env.header.Number, env.header.Time) {
        filter.GasLimitCap = params.MaxTxGas
        filter.BlobVersion = types.BlobSidecarVersion1
    }
    // Fetch plain and blob transactions separately
    filter.BlobTxs = false
    pendingPlainTxs := miner.txpool.Pending(filter)

    filter.BlobTxs = true
    pendingBlobTxs := miner.txpool.Pending(filter)

    // Split priority addresses out of the normal maps
    prioPlainTxs := make(map[common.Address][]*txpool.LazyTransaction)
    prioBlobTxs  := make(map[common.Address][]*txpool.LazyTransaction)
    for _, account := range miner.prio {
        if txs := pendingPlainTxs[account]; len(txs) > 0 {
            delete(pendingPlainTxs, account)
            prioPlainTxs[account] = txs
        }
        if txs := pendingBlobTxs[account]; len(txs) > 0 {
            delete(pendingBlobTxs, account)
            prioBlobTxs[account] = txs
        }
    }
    // Commit prioritized transactions first, then normal ones
    if len(prioPlainTxs) > 0 || len(prioBlobTxs) > 0 {
        plainTxs := newTransactionsByPriceAndNonce(env.signer, prioPlainTxs, env.header.BaseFee)
        blobTxs := newTransactionsByPriceAndNonce(env.signer, prioBlobTxs, env.header.BaseFee)
        miner.commitTransactions(env, plainTxs, blobTxs, interrupt)
    }
    if len(pendingPlainTxs) > 0 || len(pendingBlobTxs) > 0 {
        plainTxs := newTransactionsByPriceAndNonce(env.signer, pendingPlainTxs, env.header.BaseFee)
        blobTxs := newTransactionsByPriceAndNonce(env.signer, pendingBlobTxs, env.header.BaseFee)
        miner.commitTransactions(env, plainTxs, blobTxs, interrupt)
    }
    return nil
}
```

The `PendingFilter` pre-filters transactions in the pool (see Chapter 08) so only transactions that meet the minimum tip, base fee, and blob fee thresholds are returned. Post-Osaka, the filter also enforces a per-transaction gas cap (`MaxTxGas`) and requires blob sidecar version 1 (cell proofs). This avoids pulling thousands of transactions that cannot be included.

**Priority addresses** (`miner.prio`) are committed first. These are configured via `SetPrioAddresses()` and allow operators to guarantee inclusion for specific senders (e.g., MEV builders, protocol operations).

### Transaction Ordering

The `transactionsByPriceAndNonce` struct in `miner/ordering.go` implements the ordering strategy:

```go
// miner/ordering.go

type transactionsByPriceAndNonce struct {
    txs     map[common.Address][]*txpool.LazyTransaction // per-account nonce-sorted lists
    heads   txByPriceAndTime                             // max-heap of next-tx-per-account
    signer  types.Signer
    baseFee *uint256.Int
}
```

For each account, the pool provides a nonce-sorted list of pending transactions. The ordering struct creates a **max-heap** of the *first* (lowest-nonce) transaction from each account, sorted by effective tip:

```go
// miner/ordering.go

func newTxWithMinerFee(tx *txpool.LazyTransaction, from common.Address,
    baseFee *uint256.Int) (*txWithMinerFee, error) {
    tip := new(uint256.Int).Set(tx.GasTipCap)
    if baseFee != nil {
        if tx.GasFeeCap.Cmp(baseFee) < 0 {
            return nil, types.ErrGasFeeCapTooLow
        }
        tip = new(uint256.Int).Sub(tx.GasFeeCap, baseFee)
        if tip.Gt(tx.GasTipCap) {
            tip = tx.GasTipCap
        }
    }
    return &txWithMinerFee{tx: tx, from: from, fees: tip}, nil
}
```

The effective tip is `min(gasTipCap, gasFeeCap - baseFee)` — the actual amount the miner receives per unit of gas. Transactions with equal tips are ordered by time (first-seen wins).

The heap provides two operations:

- **`Shift()`** — the current best transaction succeeded. Replace it with the next transaction from the same account (maintaining nonce order) and re-heapify.
- **`Pop()`** — the current best transaction failed. Remove the entire account from consideration (since all subsequent nonces depend on this one).

### The commitTransactions Loop

`commitTransactions()` is the inner loop that picks transactions from the heap and executes them:

```go
// miner/worker.go  (simplified)

func (miner *Miner) commitTransactions(env *environment,
    plainTxs, blobTxs *transactionsByPriceAndNonce, interrupt *atomic.Int32) error {
    if env.gasPool == nil {
        env.gasPool = new(core.GasPool).AddGas(env.header.GasLimit)
    }
    for {
        // Check for interrupts (new head, timeout, recommit)
        if signal := interrupt.Load(); signal != commitInterruptNone {
            return signalToErr(signal)
        }
        // Stop if not enough gas for any transaction
        if env.gasPool.Gas() < params.TxGas {
            break
        }
        // Pick the higher-tipping transaction between plain and blob heaps
        pltx, ptip := plainTxs.Peek()
        bltx, btip := blobTxs.Peek()

        var ltx *txpool.LazyTransaction
        var txs *transactionsByPriceAndNonce
        // ... select whichever has the higher tip ...

        // Check gas budget, blob budget, block size
        if env.gasPool.Gas() < ltx.Gas { txs.Pop(); continue }
        if !env.txFitsSize(tx)          { break }

        // Execute the transaction
        env.state.SetTxContext(tx.Hash(), env.tcount)
        err := miner.commitTransaction(env, tx)

        switch {
        case errors.Is(err, core.ErrNonceTooLow):
            txs.Shift() // stale nonce, try next from same account
        case err == nil:
            txs.Shift() // success, advance to next from same account
        default:
            txs.Pop()   // fatal error, skip entire account
        }
    }
    return nil
}
```

Three kinds of budget are tracked:

1. **Gas budget** — `GasPool` starts at `header.GasLimit` and decreases with each transaction. When less than `TxGas` (21000) remains, no more transactions fit.
2. **Blob budget** — `env.blobs` tracks blob count. When it reaches `MaxBlobsPerBlock`, blob transactions are cleared from the heap.
3. **Block size** — `env.size` tracks the RLP-encoded block size. Transactions that would push the block over `MaxBlockSize - 1MB` (buffer zone) are rejected.

The **interrupt** mechanism allows the loop to be cut short. A timer set at `Recommit` (2s) fires `commitInterruptTimeout`, causing the loop to stop. The next build round starts fresh with potentially new transactions from the pool.

### Transaction Execution

Each transaction is executed via `commitTransaction()`:

```go
// miner/worker.go

func (miner *Miner) commitTransaction(env *environment, tx *types.Transaction) error {
    if tx.Type() == types.BlobTxType {
        return miner.commitBlobTransaction(env, tx)
    }
    receipt, err := miner.applyTransaction(env, tx)
    if err != nil {
        return err
    }
    env.txs = append(env.txs, tx)
    env.receipts = append(env.receipts, receipt)
    env.size += tx.Size()
    env.tcount++
    return nil
}

func (miner *Miner) applyTransaction(env *environment, tx *types.Transaction) (*types.Receipt, error) {
    snap := env.state.Snapshot()
    gp   := env.gasPool.Gas()

    receipt, err := core.ApplyTransaction(env.evm, env.gasPool, env.state, env.header, tx, &env.header.GasUsed)
    if err != nil {
        env.state.RevertToSnapshot(snap)
        env.gasPool.SetGas(gp)
    }
    return receipt, err
}
```

If execution fails, the state is reverted to the pre-transaction snapshot and the gas pool is restored — the failed transaction leaves no trace in the block. `core.ApplyTransaction()` is the same execution pipeline covered in Chapter 06.

For blob transactions, `commitBlobTransaction()` additionally checks the per-block blob limit, strips the sidecar from the transaction (blobs are not stored in the execution chain), and accumulates `BlobGasUsed` in the header.

### Step 3: Consensus-Layer Requests (Prague)

After all transactions are executed, `generateWork()` collects consensus-layer requests if the Prague fork is active:

```go
// miner/worker.go  (inside generateWork)

if miner.chainConfig.IsPrague(work.header.Number, work.header.Time) {
    requests = [][]byte{}
    // EIP-6110: parse deposit logs from executed transactions
    core.ParseDepositLogs(&requests, allLogs, miner.chainConfig)
    // EIP-7002: process withdrawal requests from the system contract
    core.ProcessWithdrawalQueue(&requests, work.evm)
    // EIP-7251: process consolidation requests from the system contract
    core.ProcessConsolidationQueue(&requests, work.evm)
}
if requests != nil {
    reqHash := types.CalcRequestsHash(requests)
    work.header.RequestsHash = &reqHash
}
```

These requests are derived from the execution results and included in the payload envelope for the CL:

- **EIP-6110 deposits** — validator deposit events are parsed from transaction logs (the deposit contract emits them).
- **EIP-7002 withdrawal requests** — a system contract call retrieves queued validator withdrawal requests.
- **EIP-7251 consolidation requests** — a system contract call retrieves queued validator consolidation requests.

### Step 4: FinalizeAndAssemble

The final step calls the consensus engine to apply post-transaction state changes and assemble the block:

```go
// miner/worker.go  (inside generateWork)

body := types.Body{Transactions: work.txs, Withdrawals: genParam.withdrawals}
block, err := miner.engine.FinalizeAndAssemble(miner.chain, work.header, work.state, &body, work.receipts)
```

For the beacon consensus engine, `FinalizeAndAssemble()` does:

1. **Process withdrawals** — credit each withdrawal's ETH amount to its recipient address.
2. **Compute state root** — `state.IntermediateRoot(true)` flushes all pending state changes into the trie and returns the root hash, which is set as `header.Root`.
3. **Assemble the block** — `types.NewBlock()` creates the final block with the header, body, and receipts.

---

## The Consensus Engine Interface

The `consensus.Engine` interface in `consensus/consensus.go` abstracts the rules for block validity:

```go
// consensus/consensus.go

type Engine interface {
    Author(header *types.Header) (common.Address, error)
    VerifyHeader(chain ChainHeaderReader, header *types.Header) error
    VerifyHeaders(chain ChainHeaderReader, headers []*types.Header) (chan<- struct{}, <-chan error)
    VerifyUncles(chain ChainReader, block *types.Block) error
    Prepare(chain ChainHeaderReader, header *types.Header) error
    Finalize(chain ChainHeaderReader, header *types.Header, state vm.StateDB, body *types.Body)
    FinalizeAndAssemble(chain ChainHeaderReader, header *types.Header, state *state.StateDB,
        body *types.Body, receipts []*types.Receipt) (*types.Block, error)
    Seal(chain ChainHeaderReader, block *types.Block, results chan<- *types.Block, stop <-chan struct{}) error
    SealHash(header *types.Header) common.Hash
    CalcDifficulty(chain ChainHeaderReader, time uint64, parent *types.Header) *big.Int
    Close() error
}
```

The methods split into block **production** and **validation**:

| Method | Role | Called by |
|--------|------|----------|
| `Prepare` | Set consensus fields on a new header | Miner (block building) |
| `Finalize` | Apply post-tx state changes (withdrawals) | State processor (block importing) |
| `FinalizeAndAssemble` | Finalize + build the block object | Miner (block building) |
| `VerifyHeader` | Check one header against consensus rules | Block importer |
| `VerifyHeaders` | Check a batch of headers concurrently | Block importer (sync) |
| `VerifyUncles` | Validate uncle blocks | Block importer |
| `Seal` | Produce a sealed block (PoW-era, no-op for PoS) | Legacy mining |
| `CalcDifficulty` | Compute the expected difficulty | Header verification |

### The Beacon Consensus Engine

The `Beacon` struct in `consensus/beacon/consensus.go` implements the `Engine` interface for post-Merge Ethereum:

```go
// consensus/beacon/consensus.go

type Beacon struct {
    ethone consensus.Engine // wrapped pre-merge engine (ethash or clique)
}
```

The beacon engine wraps a legacy engine for historical block support. Every method checks whether the block is pre-merge (difficulty > 0) or post-merge (difficulty == 0) and delegates accordingly:

**`Prepare()`** — for post-merge blocks, simply sets `header.Difficulty = 0`. Nothing else needs to be set — the CL provides timestamp, coinbase, random, etc.

**`Finalize()`** — processes beacon chain withdrawals by crediting each recipient's balance:

```go
// consensus/beacon/consensus.go

func (beacon *Beacon) Finalize(chain consensus.ChainHeaderReader, header *types.Header,
    state vm.StateDB, body *types.Body) {
    if !beacon.IsPoSHeader(header) {
        beacon.ethone.Finalize(chain, header, state, body)
        return
    }
    for _, w := range body.Withdrawals {
        amount := new(uint256.Int).SetUint64(w.Amount)
        amount = amount.Mul(amount, uint256.NewInt(params.GWei))
        state.AddBalance(w.Address, amount, tracing.BalanceIncreaseWithdrawal)
    }
}
```

Withdrawal amounts are denominated in Gwei in the beacon chain, so they are multiplied by `params.GWei` (10^9) to convert to Wei before crediting. There is no block reward in PoS — validators are rewarded by the consensus layer.

**`verifyHeader()`** — validates a post-merge header:

- Extra data must be <= 32 bytes
- Nonce must be zero, uncle hash must be the empty uncle hash
- Timestamp must be strictly greater than parent's
- Difficulty must be zero
- Gas limit <= 2^63-1, gas used <= gas limit
- Block number == parent number + 1
- EIP-1559 base fee is correctly derived from parent
- Shanghai: `WithdrawalsHash` must be present
- Cancun: `ExcessBlobGas`, `BlobGasUsed`, `ParentBeaconRoot` must be present; blob gas rules verified

**`FinalizeAndAssemble()`** — calls `Finalize()`, computes the state root via `state.IntermediateRoot(true)`, and builds the final block with `types.NewBlock()`.

---

## Block Validation

When geth *receives* a block (rather than building one), it must validate it before inserting it into the chain. The `BlockValidator` in `core/block_validator.go` handles this.

### ValidateBody

`ValidateBody()` checks the block body against the header without executing any transactions:

```go
// core/block_validator.go  (simplified)

func (v *BlockValidator) ValidateBody(block *types.Block) error {
    // EIP-7934: block size cap (Osaka)
    if v.config.IsOsaka(block.Number(), block.Time()) && block.Size() > params.MaxBlockSize {
        return ErrBlockOversized
    }
    // Already imported?
    if v.bc.HasBlockAndState(block.Hash(), block.NumberU64()) {
        return ErrKnownBlock
    }
    // Uncle validation (must be empty post-merge)
    v.bc.engine.VerifyUncles(v.bc, block)

    // Transaction root matches header
    if hash := types.DeriveSha(block.Transactions(), ...); hash != header.TxHash { ... }

    // Withdrawals root matches header (post-Shanghai)
    if header.WithdrawalsHash != nil {
        if hash := types.DeriveSha(block.Withdrawals(), ...); hash != *header.WithdrawalsHash { ... }
    }
    // Blob gas used matches actual blob count
    if header.BlobGasUsed != nil {
        if blobs * BlobTxBlobGasPerBlob != *header.BlobGasUsed { ... }
    }
    // Blob txs must not have sidecars attached in-block
    for i, tx := range block.Transactions() {
        if tx.BlobTxSidecar() != nil { ... }
    }
    // Parent must exist
    if !v.bc.HasBlockAndState(block.ParentHash(), block.NumberU64()-1) { ... }

    return nil
}
```

### ValidateState

After the block's transactions are executed (by `StateProcessor.Process()`, see Chapter 06), `ValidateState()` checks the execution results:

```go
// core/block_validator.go  (simplified)

func (v *BlockValidator) ValidateState(block *types.Block, statedb *state.StateDB,
    res *ProcessResult, stateless bool) error {
    // Gas used must match header
    if block.GasUsed() != res.GasUsed { ... }

    // Bloom filter must match
    if types.MergeBloom(res.Receipts) != header.Bloom { ... }

    // Receipt root must match header
    if types.DeriveSha(res.Receipts, ...) != header.ReceiptHash { ... }

    // Requests hash must match header (Prague)
    if header.RequestsHash != nil {
        if types.CalcRequestsHash(res.Requests) != *header.RequestsHash { ... }
    }
    // State root must match header
    if statedb.IntermediateRoot(...) != header.Root { ... }

    return nil
}
```

The state root check is the most expensive — it requires flushing all state changes into the trie and computing the root hash. If it does not match the header's declared root, the block is invalid.

---

## Putting It All Together

Here is the complete flow from the CL's request to a sealed block:

1. **CL calls `ForkchoiceUpdatedV3`** with payload attributes. Geth updates its canonical head and starts building.
2. **Empty block** is built immediately via `generateWork(noTxs: true)` — header construction, system calls, finalize.
3. **Full block** building begins in a background goroutine. Every `Recommit` interval (2s), `generateWork` runs the full pipeline: `prepareWork` → `fillTransactions` → collect requests → `FinalizeAndAssemble`.
4. **`fillTransactions`** pulls pending transactions from the pool, splits by priority, orders by effective tip, and executes them one by one. Failed transactions are skipped; the state is reverted cleanly.
5. **CL calls `GetPayloadV4`** — geth returns the best block (highest fees) built so far.
6. **CL broadcasts** the block. Other nodes receive it via `NewPayloadV4` and validate it: `VerifyHeader` → `ValidateBody` → `Process` (execute all txs) → `ValidateState` (compare roots).

The miner is the point where the transaction pool (Chapter 08), the execution engine (Chapters 06–07), and the consensus rules all converge to produce the next block in the chain.
