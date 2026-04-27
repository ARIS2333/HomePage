---
title: Geth(12) Sync and the Ethereum Wire Protocol
published: 2026-04-21
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

Chapter 11 showed how geth establishes encrypted, multiplexed connections with peers. This chapter covers what travels over those connections: the **Ethereum wire protocol** (`eth`) that lets nodes exchange blocks, transactions, and chain state. It also covers how geth **synchronizes** with the network — downloading the blockchain from scratch or catching up after being offline.

---

## Overview: How Data Flows Between Peers

Data propagation in geth involves three main flows, each with its own subsystem:

```
 Transaction enters local pool
      |
      v
 handler.txBroadcastLoop()     -- subscribes to NewTxsEvent
      |
      +-- Direct broadcast to ~sqrt(N) peers  (TransactionsMsg)
      +-- Hash announcement to all other peers (NewPooledTransactionHashesMsg)
                                                      |
                                                      v
                                               Remote peer's TxFetcher
                                               requests full tx via
                                               GetPooledTransactionsMsg

 Consensus layer announces new head
      |
      v
 downloader.skeleton.Sync()    -- downloads header chain backwards
      |
      v
 downloader.syncToHead()       -- fetches bodies, receipts, state
      |
      v
 blockchain.InsertChain()      -- executes and validates blocks
```

The key components:

| Component | Location | Role |
|-----------|----------|------|
| `handler` | `eth/handler.go` | Orchestrates all protocol activity |
| `eth.Peer` | `eth/protocols/eth/peer.go` | Per-peer message queues and known-item tracking |
| `TxFetcher` | `eth/fetcher/tx_fetcher.go` | Retrieves transactions announced by peers |
| `Downloader` | `eth/downloader/downloader.go` | Synchronizes the blockchain |
| `skeleton` | `eth/downloader/skeleton.go` | Downloads the header chain (post-Merge) |

---

## The Ethereum Wire Protocol

The `eth` protocol is a devp2p sub-protocol (see Chapter 11) that defines how Ethereum nodes exchange chain data. Geth supports two versions:

```go
// eth/protocols/eth/protocol.go

const (
    ETH68 = 68
    ETH69 = 69
)

var ProtocolVersions = []uint{ETH69, ETH68}
var protocolLengths = map[uint]uint64{ETH68: 17, ETH69: 18}

const ProtocolName = "eth"
const maxMessageSize = 10 * 1024 * 1024  // 10 MB
```

### Message Types

The protocol defines 15 message codes. Each message is RLP-encoded and carries a `RequestId` for request-response correlation:

```go
// eth/protocols/eth/protocol.go

const (
    StatusMsg                     = 0x00
    NewBlockHashesMsg             = 0x01  // ETH68 only (disabled post-Merge)
    TransactionsMsg               = 0x02
    GetBlockHeadersMsg            = 0x03
    BlockHeadersMsg               = 0x04
    GetBlockBodiesMsg             = 0x05
    BlockBodiesMsg                = 0x06
    NewBlockMsg                   = 0x07  // ETH68 only (disabled post-Merge)
    NewPooledTransactionHashesMsg = 0x08
    GetPooledTransactionsMsg      = 0x09
    PooledTransactionsMsg         = 0x0a
    GetReceiptsMsg                = 0x0f
    ReceiptsMsg                   = 0x10
    BlockRangeUpdateMsg           = 0x11  // ETH69 only
)
```

The messages group into three categories:

| Category | Messages | Purpose |
|----------|----------|---------|
| **Handshake** | `StatusMsg` | Exchange chain identity on connection |
| **Transaction gossip** | `TransactionsMsg`, `NewPooledTransactionHashesMsg`, `GetPooledTransactionsMsg`, `PooledTransactionsMsg` | Propagate pending transactions |
| **Chain sync** | `GetBlockHeadersMsg`/`BlockHeadersMsg`, `GetBlockBodiesMsg`/`BlockBodiesMsg`, `GetReceiptsMsg`/`ReceiptsMsg`, `BlockRangeUpdateMsg` | Download chain data |

ETH69 drops `NewBlockHashesMsg` and `NewBlockMsg` (block announcements are no longer needed post-Merge since the consensus layer drives block progression) and adds `BlockRangeUpdateMsg` so peers can advertise which block range they serve.

### The Status Handshake

The first message on every `eth` connection is the status handshake, exchanged in both directions simultaneously:

```go
// eth/protocols/eth/protocol.go

type StatusPacket68 struct {
    ProtocolVersion uint32
    NetworkID       uint64
    TD              *big.Int       // Total difficulty
    Head            common.Hash    // Head block hash
    Genesis         common.Hash    // Genesis hash
    ForkID          forkid.ID      // Fork identifier (EIP-2124)
}

type StatusPacket69 struct {
    ProtocolVersion uint32
    NetworkID       uint64
    Genesis         common.Hash
    ForkID          forkid.ID
    EarliestBlock   uint64         // Earliest available block
    LatestBlock     uint64         // Latest available block
    LatestBlockHash common.Hash
}
```

ETH69 removes `TD` (total difficulty is meaningless post-Merge) and `Head` (replaced by the explicit block range), and adds `EarliestBlock`/`LatestBlock` so peers know which data the remote node can serve.

After exchanging status messages, each side validates:

1. **NetworkID** must match (mainnet = 1, sepolia = 11155111, etc.)
2. **Genesis hash** must match
3. **ForkID** must be compatible (see the Fork ID section below)
4. **Protocol version** must match

If any check fails, the peer is disconnected.

---

## The Handler: Orchestrating Protocol Activity

The `handler` struct in `eth/handler.go` ties together all protocol subsystems:

```go
// eth/handler.go

type handler struct {
    nodeID    enode.ID
    networkID uint64

    snapSync atomic.Bool  // Whether snap sync is enabled
    synced   atomic.Bool  // Whether we're considered synchronised

    database ethdb.Database
    txpool   txPool
    chain    *core.BlockChain
    maxPeers int

    downloader     *downloader.Downloader
    txFetcher      *fetcher.TxFetcher
    peers          *peerSet
    txBroadcastKey [16]byte

    eventMux   *event.TypeMux
    txsCh      chan core.NewTxsEvent
    txsSub     event.Subscription
    blockRange *blockRangeState

    requiredBlocks map[uint64]common.Hash
    quitSync       chan struct{}
    wg             sync.WaitGroup
    // ...
}
```

Key fields:

- **`snapSync`** — true during snap sync, false during full sync. Snap sync downloads state snapshots instead of re-executing all transactions from genesis.
- **`synced`** — set to true once initial sync completes. While false, the node rejects incoming transactions to avoid filling the pool with potentially stale data.
- **`downloader`** — the sync engine that fetches headers, bodies, and receipts.
- **`txFetcher`** — retrieves transactions that peers have announced.
- **`peers`** — the set of currently connected `eth` peers.
- **`txBroadcastKey`** — a random key used for deterministic peer selection during transaction broadcast.
- **`blockRange`** — tracks the local node's available block range and broadcasts updates to ETH69 peers.

### Startup

`Start()` launches the background goroutines:

```go
// eth/handler.go

func (h *handler) Start(maxPeers int) {
    h.maxPeers = maxPeers

    // Subscribe to new transactions and start broadcast loop
    h.txsCh = make(chan core.NewTxsEvent, txChanSize)      // buffer: 4096
    h.txsSub = h.txpool.SubscribeTransactions(h.txsCh, false)
    go h.txBroadcastLoop()

    // Start block range state broadcaster (ETH69)
    h.blockRange = newBlockRangeState(h.chain, h.eventMux)
    go h.blockRangeLoop(h.blockRange)

    // Start the transaction fetcher
    h.txFetcher.Start()

    // Start peer handler tracker
    go h.protoTracker()
}
```

### Peer Lifecycle

When an `eth` peer connects, `runEthPeer()` handles the full lifecycle:

```go
// eth/handler.go  (simplified)

func (h *handler) runEthPeer(peer *eth.Peer, handler eth.Handler) error {
    // 1. Wait for snap extension (if peer supports snap protocol)
    snap, err := h.peers.waitSnapExtension(peer)

    // 2. Execute the Ethereum status handshake
    peer.Handshake(h.networkID, h.chain, h.blockRange.currentRange())

    // 3. Enforce peer limits (reserve slots for snap-capable peers during snap sync)
    if h.snapSync.Load() && snap == nil {
        if all, snp := h.peers.len(), h.peers.snapLen(); all-snp > snp+5 {
            return p2p.DiscTooManyPeers
        }
    }

    // 4. Register peer with the handler, downloader, and snap syncer
    h.peers.registerPeer(peer, snap)
    h.downloader.RegisterPeer(peer.ID(), peer.Version(), peer)
    if snap != nil {
        h.downloader.SnapSyncer.Register(snap)
    }

    // 5. Send existing pending transactions
    h.syncTransactions(peer)

    // 6. Validate required blocks (configurable checkpoint hashes)
    for number, hash := range h.requiredBlocks {
        // Request the header and verify it matches
        // ...
    }

    // 7. Enter message handling loop (blocks until disconnect)
    return handler(peer)
}
```

Step 6 is a security feature: operators can configure `RequiredBlocks` — a map of block number to hash — and any peer that cannot produce matching headers within 15 seconds is disconnected. This defends against peers serving a different chain.

---

## Transaction Propagation

Transaction propagation is the most active part of the protocol — new transactions need to reach every node quickly, but without flooding the network with duplicate data.

### The Broadcast Strategy

When new transactions enter the local pool, the handler distributes them to peers using a two-tier strategy:

```go
// eth/handler.go  (simplified)

func (h *handler) BroadcastTransactions(txs types.Transactions) {
    var (
        txset = make(map[*ethPeer][]common.Hash) // peers to send full txs
        annos = make(map[*ethPeer][]common.Hash) // peers to send announcements
        choice = newBroadcastChoice(h.nodeID, h.txBroadcastKey)
        peers  = h.peers.all()
    )

    for _, tx := range txs {
        var directSet map[*ethPeer]struct{}
        switch {
        case tx.Type() == types.BlobTxType:
            // Blob txs: announce only (too large to broadcast)
        case tx.Size() > txMaxBroadcastSize:
            // Large txs (>4KB): announce only
        default:
            txSender, _ := types.Sender(signer, tx)
            directSet = choice.choosePeers(peers, txSender)
        }

        for _, peer := range peers {
            if peer.KnownTransaction(tx.Hash()) {
                continue
            }
            if _, ok := directSet[peer]; ok {
                txset[peer] = append(txset[peer], tx.Hash())
            } else {
                annos[peer] = append(annos[peer], tx.Hash())
            }
        }
    }

    for peer, hashes := range txset {
        peer.AsyncSendTransactions(hashes)
    }
    for peer, hashes := range annos {
        peer.AsyncSendPooledTransactionHashes(hashes)
    }
}
```

The routing rules:

- **Blob transactions** — always announced, never directly broadcast. Blob txs carry ~128KB of blob data, so sending them to all peers would be extremely wasteful.
- **Large transactions** (>4096 bytes) — also announce-only.
- **Regular transactions** — sent directly to approximately `sqrt(N)` peers (chosen deterministically using `siphash(key, self_id, peer_id, tx_sender)`), and announced to all remaining peers.

The deterministic peer selection ensures that different nodes in the network choose different subsets to broadcast to, providing good coverage without every node sending to every peer.

### Per-Peer Send Queues

Each peer has two async channels for transaction distribution:

```go
// eth/protocols/eth/peer.go

type Peer struct {
    // ...
    knownTxs    *knownCache          // Set of tx hashes known to this peer (max 32768)
    txBroadcast chan []common.Hash    // Channel for direct tx broadcasts
    txAnnounce  chan []common.Hash    // Channel for tx announcements
    // ...
}
```

The `broadcastTransactions()` goroutine drains `txBroadcast`, batching hashes into packets up to 100KB:

```go
// eth/protocols/eth/broadcast.go  (simplified)

func (p *Peer) broadcastTransactions() {
    for {
        // Accumulate hashes, then:
        // 1. Gather full tx data from txpool for each hash
        // 2. Build packet up to maxTxPacketSize (100KB)
        // 3. Send TransactionsMsg asynchronously
    }
}
```

Similarly, `announceTransactions()` drains `txAnnounce` and sends `NewPooledTransactionHashesMsg` packets containing the hash, type, and size of each announced transaction. The type and size metadata lets the recipient decide whether to fetch the transaction without needing a round trip.

### The Transaction Fetcher

When a peer announces transaction hashes, the `TxFetcher` manages retrieval through a three-stage pipeline:

```
 Stage 1: Waitlist          Stage 2: Queue           Stage 3: Fetching
 (wait 500ms for broadcast)  (ready to request)       (request in flight)

 Announced hash             Not arrived?              Pick peer, send
 -----> waitlist            -----> announces           GetPooledTransactionsMsg
                                                       |
                                                       v
                                                    PooledTransactionsMsg
                                                       |
                                                       v
                                                    Add to txpool
```

Key constants that control the pipeline:

```go
// eth/fetcher/tx_fetcher.go

const (
    maxTxAnnounces     = 4096          // Max announcements per peer
    maxTxRetrievals    = 256           // Max txs per request
    maxTxRetrievalSize = 128 * 1024    // 128KB max per request
    txArriveTimeout    = 500 * time.Millisecond
    txFetchTimeout     = 5 * time.Second
)
```

- **Stage 1 (Waitlist):** When a hash is first announced, it waits 500ms. Many transactions arrive via direct broadcast from other peers within this window, avoiding an unnecessary fetch request.
- **Stage 2 (Queue):** If the transaction hasn't arrived after the wait, it moves to the request queue.
- **Stage 3 (Fetching):** The fetcher sends `GetPooledTransactionsMsg` to a peer that announced the hash. If the peer doesn't respond within 5 seconds, the request is retried with a different peer.

The fetcher also tracks **underpriced** transactions — those that the txpool rejected as too cheap. These are cached for 5 minutes so the fetcher doesn't repeatedly request them from different peers.

---

## Message Handling

When a message arrives from a peer, it is dispatched to a handler function via a version-specific handler map:

```go
// eth/protocols/eth/handlers.go

var eth68 = map[uint64]msgHandler{
    NewBlockHashesMsg:             handleNewBlockhashes,
    NewBlockMsg:                   handleNewBlock,
    TransactionsMsg:               handleTransactions,
    NewPooledTransactionHashesMsg: handleNewPooledTransactionHashes,
    GetBlockHeadersMsg:            handleGetBlockHeaders,
    BlockHeadersMsg:               handleBlockHeaders,
    GetBlockBodiesMsg:             handleGetBlockBodies,
    BlockBodiesMsg:                handleBlockBodies,
    GetReceiptsMsg:                handleGetReceipts68,
    ReceiptsMsg:                   handleReceipts[*ReceiptList68],
    GetPooledTransactionsMsg:      handleGetPooledTransactions,
    PooledTransactionsMsg:         handlePooledTransactions,
}
```

ETH69 drops the block announcement handlers (`NewBlockHashes` and `NewBlock` return errors) and adds `BlockRangeUpdateMsg`.

### Serving Data

Request handlers serve chain data to remote peers. For example, header requests:

```go
// eth/protocols/eth/handlers.go

func handleGetBlockHeaders(backend Backend, msg Decoder, peer *Peer) error {
    var query GetBlockHeadersPacket
    if err := msg.Decode(&query); err != nil {
        return err
    }
    response := ServiceGetBlockHeadersQuery(backend.Chain(), query.GetBlockHeadersRequest, peer)
    return peer.ReplyBlockHeadersRLP(query.RequestId, response)
}
```

`ServiceGetBlockHeadersQuery()` handles two modes: **number mode** (contiguous segment from a block number) and **hash mode** (starting from a specific hash). It respects response limits — at most 1024 headers and 2MB total.

### Receiving Transactions

Directly broadcast transactions arrive via `TransactionsMsg`:

```go
// eth/protocols/eth/handlers.go  (simplified)

func handleTransactions(backend Backend, msg Decoder, peer *Peer) error {
    var txs TransactionsPacket
    if err := msg.Decode(&txs); err != nil {
        return err
    }
    // Mark each tx as known by this peer
    for _, tx := range txs {
        peer.markTransaction(tx.Hash())
    }
    // Inject into local txpool
    return backend.Handle(peer, &txs)
}
```

Transaction announcements arrive via `NewPooledTransactionHashesMsg`:

```go
// eth/protocols/eth/handlers.go  (simplified)

func handleNewPooledTransactionHashes(backend Backend, msg Decoder, peer *Peer) error {
    var ann NewPooledTransactionHashesPacket
    if err := msg.Decode(&ann); err != nil {
        return err
    }
    // Validate: hashes, types, and sizes arrays must have equal length
    // Mark hashes as known, notify the tx fetcher
    return backend.Handle(peer, &ann)
}
```

The `backend.Handle()` call routes to the handler, which feeds the announcement to `txFetcher.Notify()`, triggering the three-stage retrieval pipeline.

### Request/Response Dispatch

Outgoing requests and incoming responses are correlated via the `dispatcher` goroutine in each peer. Each request carries a `RequestId` and a response sink channel:

```go
// eth/protocols/eth/dispatcher.go

type Request struct {
    peer *Peer
    id   uint64            // RequestId for correlation
    sink chan *Response     // Where the response will be delivered
    code uint64            // Request message code
    want uint64            // Expected response code
    data interface{}       // Request payload
}

type Response struct {
    Req  *Request
    Res  interface{}       // Response data
    Meta interface{}       // Computed metadata
    Time time.Duration     // Round-trip time
    Done chan error         // Signal to reader
}
```

When a response message arrives (e.g., `BlockHeadersMsg`), the handler dispatches it to the matching request's sink channel based on `RequestId`. The requester reads from the sink and signals completion via `Done`.

---

## Chain Synchronization

When a node starts (or falls behind), it needs to download the blockchain from peers. Post-Merge, synchronization is driven by the consensus layer (beacon client) which tells geth where the chain head is.

### The Downloader

The `Downloader` struct in `eth/downloader/downloader.go` orchestrates the entire sync process:

```go
// eth/downloader/downloader.go

type Downloader struct {
    mode atomic.Uint32    // FullSync or SnapSync

    queue *queue           // Scheduler for block fetching
    peers *peerSet         // Active peers

    stateDB    ethdb.Database
    blockchain BlockChain

    // Skeleton sync (post-merge)
    skeleton *skeleton

    // State sync
    pivotHeader *types.Header      // Snap sync pivot block
    SnapSyncer  *snap.Syncer       // State snapshot downloader

    // Cancellation
    cancelCh chan struct{}
    quitCh   chan struct{}
    // ...
}
```

Two sync modes are supported:

| Mode | Strategy |
|------|----------|
| **Full sync** | Download all headers, bodies, and receipts. Execute every transaction to reconstruct state. |
| **Snap sync** | Download headers, bodies, and receipts. Download a state snapshot at a recent "pivot" block instead of re-executing the full history. Only execute the last ~64 blocks. |

Snap sync is much faster (hours instead of days) because it avoids re-executing millions of historical transactions. However, it requires peers that support the `snap` protocol and have a recent state snapshot available.

### Key Limits

```go
// eth/downloader/downloader.go

var (
    MaxBlockFetch   = 128   // Blocks per request
    MaxHeaderFetch  = 192   // Headers per request
    MaxReceiptFetch = 256   // Receipts per request
    maxResultsProcess = 2048   // Results to import at once
    fsMinFullBlocks   = 64     // Min full blocks in snap sync
)
```

### The Sync Pipeline

When the consensus layer signals a new head, the sync pipeline activates:

```
 Consensus layer: "New head at block N"
      |
      v
 skeleton.Sync(head, finalized)    -- Download header skeleton backwards
      |
      v
 syncToHead()                       -- Main sync orchestrator
      |
      +-- Determine sync mode (full/snap) and pivot block
      |
      +-- Spawn concurrent fetchers:
      |     fetchHeaders()           -- from skeleton chain
      |     fetchBodies()            -- block contents
      |     fetchReceipts()          -- tx receipts (snap sync only)
      |
      +-- Spawn processors:
      |     processHeaders()         -- validate and queue headers
      |     processFullSyncContent() -- execute blocks (full sync)
      |     processSnapSyncContent() -- import blocks + state (snap sync)
      |
      +-- State sync (snap sync only):
            SnapSyncer downloads account trie, storage tries, bytecode
```

### The Skeleton Syncer

The skeleton syncer (`eth/downloader/skeleton.go`) handles the first phase: downloading the header chain. Post-Merge, this is the primary mechanism for syncing headers since the consensus layer provides the chain head.

```go
// eth/downloader/skeleton.go

type skeleton struct {
    db       ethdb.Database
    filler   backfiller          // Callback to trigger body/receipt download
    peers    *peerSet
    progress *skeletonProgress   // Persistent sync state
    // ...
}

type skeletonProgress struct {
    Subchains []*subchain
    Finalized *uint64
}

type subchain struct {
    Head uint64           // Newest header block number
    Tail uint64           // Oldest header block number
    Next common.Hash      // Hash of the next oldest header
}
```

The skeleton works by downloading headers **backwards** — from the consensus-provided head toward genesis. Because the sync might be interrupted and restarted, the progress is tracked as a list of **subchains** — disjoint segments of downloaded headers. As headers are downloaded, subchains grow and eventually merge:

```
 Initial state (new head announced):

   Subchain 1:  [Head: 1000, Tail: 1000]
                 (just the tip)

 After some downloading:

   Subchain 1:  [Head: 1000, Tail: 800]
                 (200 headers downloaded backwards)

 After restart + new head:

   Subchain 1:  [Head: 1050, Tail: 1050]    (new tip)
   Subchain 2:  [Head: 1000, Tail: 800]     (previous progress)

 After gap is filled:

   Subchain 1:  [Head: 1050, Tail: 800]     (merged)

 Eventually links to local chain:

   Subchain 1:  [Head: 1050, Tail: 0]       (complete)
```

Headers are downloaded in batches of 512 (`requestHeaders`) and stored in a scratch space of up to 131072 headers (`scratchHeaders`, ~64MB). Once the skeleton chain links to the local chain (or genesis), the downloader starts backfilling bodies, receipts, and state.

### Full Sync vs. Snap Sync

After the skeleton chain is complete, the downloader fills in the remaining data:

**Full Sync:**
1. `fetchBodies()` downloads block bodies (transactions, uncles, withdrawals) for all headers.
2. `processFullSyncContent()` calls `blockchain.InsertChain()` for each batch, which executes every transaction and builds the state trie from scratch (see Chapter 10).

**Snap Sync:**
1. A **pivot block** is chosen near the chain head (at least 64 blocks from the tip). Everything at or below the pivot uses snap sync; everything above uses full execution.
2. `fetchBodies()` and `fetchReceipts()` download bodies and receipts for all blocks.
3. The `SnapSyncer` downloads the state trie at the pivot block using the `snap` protocol — fetching account ranges, storage ranges, and bytecode in parallel from multiple peers.
4. `processSnapSyncContent()` imports blocks below the pivot using the downloaded receipts (no re-execution needed), then switches to full execution for the last ~64 blocks.

---

## Fork ID: Chain Compatibility (EIP-2124)

Before exchanging any chain data, peers need to know they're on the same chain. The **fork ID** is a compact identifier that encodes which forks a chain has activated:

```go
// core/forkid/forkid.go

type ID struct {
    Hash [4]byte  // CRC32 checksum of genesis + passed forks
    Next uint64   // Block/timestamp of next upcoming fork (0 if none)
}
```

The fork ID is calculated by hashing the genesis block and all activated fork block numbers:

```go
// core/forkid/forkid.go

func NewID(config *params.ChainConfig, genesis *types.Block, head, time uint64) ID {
    hash := crc32.ChecksumIEEE(genesis.Hash().Bytes())

    forksByBlock, forksByTime := gatherForks(config, genesis.Time())
    for _, fork := range forksByBlock {
        if fork <= head {
            hash = checksumUpdate(hash, fork)
            continue
        }
        return ID{Hash: checksumToBytes(hash), Next: fork}
    }
    for _, fork := range forksByTime {
        if fork <= time {
            hash = checksumUpdate(hash, fork)
            continue
        }
        return ID{Hash: checksumToBytes(hash), Next: fork}
    }
    return ID{Hash: checksumToBytes(hash), Next: 0}
}
```

The `gatherForks()` function reflects on the `ChainConfig` struct to collect all fork activation points (both block-based and timestamp-based), sorts them, and removes duplicates.

The fork ID enables four validation rules during the handshake:

1. **Same fork state** — both nodes have activated the same forks. Compatible.
2. **Remote is subset** — the remote node has activated fewer forks but knows the correct next fork. It's still syncing. Compatible.
3. **Remote is superset** — the remote node has activated more forks. The local node might be behind. Compatible (it may catch up).
4. **No match** — different fork history. Incompatible chains — disconnect.

This prevents nodes on different networks (or different hard-fork choices) from wasting time trying to sync with each other.

---

## DoS Protection

Several mechanisms protect against misbehaving peers:

**Response limits** — every serving handler caps its response size. Header requests return at most 1024 headers or 2MB. Body and receipt requests have similar limits. This prevents a single request from consuming excessive bandwidth.

**Known-item tracking** — each peer tracks up to 32768 known transaction hashes via `knownCache`. Transactions already known to a peer are not re-sent, reducing redundant traffic:

```go
// eth/protocols/eth/peer.go

const maxKnownTxs = 32768

type knownCache struct {
    hashes mapset.Set[common.Hash]
    max    int
}
```

**Announcement limits** — the `TxFetcher` caps announcements at 4096 per peer. If a peer exceeds this, the excess is silently dropped rather than queued.

**Request timeouts** — the request tracker (`requestTracker`) monitors pending requests and drops peers that fail to respond within 5 minutes.

**Peer scoring** — the downloader tracks which peers return invalid data. A peer that serves bad headers, bodies, or receipts is banned and disconnected via the `dropPeer` callback.

**Required block challenges** — operators can configure `RequiredBlocks` to verify that peers serve the expected chain. Peers that fail the challenge within 15 seconds are dropped.

---

## ETH68 vs. ETH69

ETH69 is the primary protocol version and introduces several changes:

| Feature | ETH68 | ETH69 |
|---------|-------|-------|
| Block announcements (`NewBlockMsg`, `NewBlockHashesMsg`) | Supported (disabled post-Merge) | Removed |
| Status packet | Includes `TD` and `Head` | Replaces with `EarliestBlock`/`LatestBlock` block range |
| `BlockRangeUpdateMsg` | Not present | Peers broadcast their available block range |
| Receipt format | Includes bloom filters | Omits bloom filters (recipients recompute them) |

The block range update mechanism in ETH69 lets peers advertise which blocks they can serve. This is particularly useful for pruned nodes that don't retain the full chain history — peers know not to request blocks outside the advertised range. The handler sends updates whenever the range changes by more than 32 blocks.

---

## Putting It All Together

Here is the lifecycle of a transaction from submission to network-wide propagation:

1. A user submits a transaction via `eth_sendRawTransaction` (JSON-RPC).
2. The transaction enters the local txpool, which emits a `NewTxsEvent`.
3. `handler.txBroadcastLoop()` receives the event and calls `BroadcastTransactions()`.
4. The transaction is sent directly to `~sqrt(N)` deterministically chosen peers via `TransactionsMsg`, and announced to all other peers via `NewPooledTransactionHashesMsg`.
5. Peers receiving the announcement feed it to their `TxFetcher`, which waits 500ms for a possible direct broadcast, then sends `GetPooledTransactionsMsg` to retrieve the full transaction.
6. Within seconds, the transaction has propagated to effectively every node in the network.

And the lifecycle of a new block from the consensus layer to a fully synced node:

1. The beacon client calls `ForkchoiceUpdated` with a new head hash.
2. The Engine API calls `skeleton.Sync()` with the new head header.
3. The skeleton syncer downloads headers backwards, filling in gaps in its subchain list.
4. Once the skeleton links to the local chain, the downloader spawns fetchers for bodies and receipts.
5. `processFullSyncContent()` or `processSnapSyncContent()` inserts the downloaded blocks into the blockchain via `InsertChain()`.
6. The chain head advances, triggering a `ChainHeadEvent` that updates the txpool, broadcasts the new state to peers, and signals readiness to the consensus layer.
