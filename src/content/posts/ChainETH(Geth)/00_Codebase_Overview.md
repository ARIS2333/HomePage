---
title: Geth(0) Codebase Overview
published: 2026-04-09
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

This chapter gives you a map of the entire go-ethereum (geth) codebase before you read any implementation detail. By the end, you will know what geth does, how its packages are layered, and where to find the code behind every major Ethereum concept.

---

## What Is Geth?

Geth is Ethereum's **execution layer** client. After the Merge (September 2022), Ethereum split into two layers:

- **Consensus layer (CL):** A separate client (e.g. Prysm, Lighthouse) that runs proof-of-stake, chooses the canonical chain, and tells the execution layer what to do.
- **Execution layer (EL):** Geth. It executes transactions, maintains the world state, stores blocks, and serves JSON-RPC queries.

The two layers communicate via the **Engine API** — a set of authenticated JSON-RPC methods (`engine_forkchoiceUpdatedV*`, `engine_newPayloadV*`, `engine_getPayloadV*`). The consensus client drives the execution client: it tells geth which block to build, which payload to accept, and which fork choice to follow.

Geth no longer decides consensus on its own. It validates and executes blocks, but the CL decides which blocks are canonical.

This version of geth is **v1.16.7**, which corresponds to the **Fusaka** hard fork (also known as **Fulu-Osaka** — "Fulu" is the consensus-layer fork name, "Osaka" is the execution-layer fork name).

---

## Layered Architecture

Geth is organized in layers. Outer layers depend on inner layers, never the reverse. Think of it as six layers, from the outermost shell down to disk:

```
+-----------------------------------------------------------------------+
|  Layer 6: Entry Point                                                 |
|  cmd/geth — CLI flags, config parsing, assembles the node             |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
|  Layer 5: Service Host                                                |
|  node — manages P2P server, RPC servers, lifecycle of all services    |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
|  Layer 4: Ethereum Service                                            |
|  eth — wires chain + pool + syncer + miner + Engine API together      |
|                                                                       |
|  key sub-packages:                                                    |
|    miner/         block builder (payload assembly)                    |
|    eth/handler    peer message dispatch, tx/block broadcast           |
|    eth/catalyst   Engine API (CL <-> EL bridge)                       |
|    eth/downloader sync orchestrator (snap sync, full sync)            |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
|  Layer 3: Chain Logic                                                 |
|  core — blockchain, state processor, tx pool, genesis                 |
|                                                                       |
|  key sub-packages:                                                    |
|    core/vm/       EVM interpreter, opcodes, gas tables, precompiles   |
|    core/state/    StateDB, state objects, journal, snapshots          |
|    core/types/    Block, Header, Transaction, Receipt, Log            |
|    core/txpool/   transaction pool (legacypool + blobpool)            |
|    core/rawdb/    low-level DB accessors, key schema, freezer         |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
|  Layer 2: Trie                                                        |
|  trie + triedb — Merkle Patricia Trie: nodes, hashing, persistence   |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
|  Layer 1: Disk                                                        |
|  ethdb — key-value store interface (LevelDB / Pebble backends)        |
+-----------------------------------------------------------------------+
```

**Cross-cutting packages** (imported by nearly every layer):

| Package | Purpose |
|---------|---------|
| `common/` | `Address` (20 bytes), `Hash` (32 bytes) |
| `params/` | Chain config, gas constants, fork schedule |
| `rlp/` | RLP encoding/decoding (Ethereum's serialization) |
| `crypto/` | Keccak256, ECDSA, KZG |
| `event/` | `Feed` pub/sub for inter-module communication |
| `p2p/` | Encrypted networking and peer discovery |
| `log/`, `metrics/` | Logging and monitoring |

### How the layers interact

There are two directions of communication between these layers:

**Downward: function calls.** When a higher layer needs work done, it calls into the layer below. For example, when a new block arrives, the call chain looks like this:

```
eth/handler receives block from a peer
  → core.BlockChain.InsertChain()        executes the block
    → core.StateProcessor.Process()      runs each transaction
      → core/vm.EVM.Call()               interprets bytecode
        → core/state.StateDB.SetState()  writes a storage slot
          → trie.Trie.Update()           updates the Merkle trie
            → ethdb.Put()                persists to LevelDB/Pebble
```

Each layer only knows about the layer directly below it. The EVM calls `StateDB` methods but has no idea that a Merkle trie or a disk database exists underneath.

**Upward: event feeds.** Lower layers never call upper layers directly. Instead, they broadcast events. Upper layers subscribe to the events they care about. For example, after `BlockChain` inserts a new block, it sends a `ChainHeadEvent`. Two independent subscribers react:

- The **transaction pool** re-validates pending transactions against the new state
- The **handler** broadcasts the new block hash to peers

This keeps the layers decoupled — `core/` doesn't import `eth/` or `miner/`, yet they all stay in sync.

---

## Directory-to-Concept Mapping

The table below maps every top-level directory to the Ethereum concept it implements. This is your lookup table for "I want to understand X — where do I look?"

| Directory | What It Does | Ethereum Concept |
|-----------|-------------|------------------|
| `cmd/geth/` | CLI entry point, flag parsing, node assembly | — |
| `node/` | Service host: manages lifecycle, P2P server, RPC endpoints | — |
| `eth/` | Ethereum service: wires together chain, pool, syncer, handler | — |
| `eth/catalyst/` | Engine API implementation | CL ↔ EL communication |
| `eth/downloader/` | Sync orchestrator (snap sync, full sync) | Chain synchronization |
| `eth/fetcher/` | Fetches announced blocks and transactions from peers | Block/tx propagation |
| `eth/protocols/eth/` | Ethereum wire protocol (message types, handlers) | devp2p sub-protocol |
| `eth/protocols/snap/` | Snap protocol for state download | Snap sync |
| `core/` | Blockchain, state processor, genesis, gas pool | Block execution |
| `core/types/` | Block, Transaction, Receipt, Header, Log | Core data structures |
| `core/vm/` | EVM interpreter, opcodes, gas tables, precompiles | EVM |
| `core/state/` | StateDB, state objects, journal, snapshots | World state |
| `core/txpool/` | Transaction pool coordinator | Mempool |
| `core/txpool/legacypool/` | Pool for regular transactions | Pending/queued txs |
| `core/txpool/blobpool/` | Pool for EIP-4844 blob transactions | Blob mempool |
| `core/rawdb/` | Low-level DB accessors, key schema, freezer | Chain storage |
| `trie/` | Merkle Patricia Trie implementation | State trie, storage tries |
| `triedb/` | Trie persistence layer (hash-based and path-based) | Trie-to-disk mapping |
| `ethdb/` | Key-value store interface + LevelDB/Pebble backends | Disk storage |
| `p2p/` | devp2p networking: encrypted transport, discovery, peer mgmt | Peer-to-peer network |
| `p2p/discover/` | Kademlia DHT (v4 and v5) | Node discovery |
| `p2p/rlpx/` | RLPx encrypted transport | Encrypted connections |
| `consensus/` | Consensus engine interface | Consensus rules |
| `consensus/beacon/` | Proof-of-Stake consensus (post-Merge) | PoS validation |
| `miner/` | Block builder (payload assembly) | Block production |
| `rpc/` | JSON-RPC server, transports (HTTP, WS, IPC) | Client API |
| `internal/ethapi/` | JSON-RPC method implementations | `eth_*`, `debug_*` APIs |
| `accounts/` | Account manager, keystore, HD wallets | Key management |
| `common/` | `Address` (20 bytes), `Hash` (32 bytes), utility types | Foundational types |
| `params/` | Chain config, fork schedule, gas constants | Protocol parameters |
| `rlp/` | RLP encoding/decoding | Serialization |
| `crypto/` | Keccak256, ECDSA sign/recover, KZG (EIP-4844) | Cryptography |
| `event/` | `Feed` (pub/sub), `TypeMux` (legacy event dispatch) | Internal event system |
| `log/` | Structured logging | — |
| `metrics/` | Prometheus-style metrics | — |
| `beacon/` | Light beacon chain client (experimental) | CL light client |
| `graphql/` | GraphQL API | Alternative query API |
| `ethclient/` | Go client library for JSON-RPC | Client SDK |
| `console/` | Interactive JavaScript console | REPL |
| `signer/` | External signer support (Clef) | Transaction signing |
| `tests/` | Ethereum test suite runners | — |

---

## Go Patterns Used Throughout

Four Go patterns appear repeatedly across the codebase. Recognizing them will make every chapter easier to follow.

### Pattern 1: The `Lifecycle` Interface

Geth manages dozens of subsystems (the Ethereum service, the transaction pool, the local tx tracker, etc.) that need to start and stop in order. The `node.Lifecycle` interface standardizes this:

```go
// node/lifecycle.go

type Lifecycle interface {
    Start() error
    Stop() error
}
```

Any component that has background goroutines implements `Lifecycle` and registers itself on the `Node` via `RegisterLifecycle()`. When `Node.Start()` is called, it starts all registered lifecycles in order. When `Node.Close()` is called, it stops them in **reverse order** — ensuring that components are torn down cleanly.

For example, in `eth/backend.go`, the Ethereum service registers itself as a lifecycle on the node:

```go
// eth/backend.go (inside the New function)

stack.RegisterAPIs(eth.APIs())
stack.RegisterProtocols(eth.Protocols())
stack.RegisterLifecycle(eth)
```

This means the `Node` controls when `Ethereum.Start()` and `Ethereum.Stop()` are called — the Ethereum service doesn't start itself.

### Pattern 2: Event Feeds

Modules communicate asynchronously through `event.Feed` — a typed pub/sub channel. A producer calls `Send()`, and all subscribers receive the value on their channel simultaneously.

```go
// event/feed.go

type Feed struct {
    // ...
}
```

`Feed.Subscribe(channel)` adds a listener. `Feed.Send(value)` delivers the value to all subscribed channels.

For example, when the blockchain imports a new block, it sends a `ChainHeadEvent`. The transaction pool subscribes to this event so it can re-validate pending transactions against the new state. The miner subscribes so it can start building the next block. The handler subscribes so it can broadcast the new head to peers.

```go
// core/events.go

type ChainHeadEvent struct {
    Header *types.Header
}
```

You will see this pattern in every chapter: feeds are the glue that lets loosely-coupled modules react to state changes.

### Pattern 3: Config Structs

Almost every subsystem takes a config struct as its first constructor argument. The pattern is always the same:

1. A `Config` struct with exported fields and sensible zero values
2. A package-level `Defaults` variable with production defaults
3. The CLI layer (in `cmd/`) maps command-line flags to config fields

For example, `eth/ethconfig/config.go` defines `ethconfig.Config` for the Ethereum service, `node/config.go` defines `node.Config` for the host, and `core/txpool/legacypool/legacypool.go` defines pool-specific config. At startup, the top-level `gethConfig` struct bundles them all:

```go
// cmd/geth/config.go

type gethConfig struct {
    Eth      ethconfig.Config
    Node     node.Config
    Ethstats ethstatsConfig
    Metrics  metrics.Config
}
```

### Pattern 4: Atomic Pointers for Hot State

For state that is read frequently from many goroutines but written rarely, geth uses `atomic.Pointer[T]` instead of a mutex. The primary example is in `BlockChain`:

```go
// core/blockchain.go

currentBlock      atomic.Pointer[types.Header] // Current head of the chain
currentSnapBlock  atomic.Pointer[types.Header] // Current head of snap-sync
currentFinalBlock atomic.Pointer[types.Header] // Latest (consensus) finalized block
currentSafeBlock  atomic.Pointer[types.Header] // Latest (consensus) safe block
```

Any goroutine can call `currentBlock.Load()` to read the current chain head without acquiring a lock. Only the chain insertion path calls `currentBlock.Store()` to update it. This avoids contention on the most frequently accessed piece of state in the entire system.

---

## Four Workflows at a Glance

The central question of this guide is: **"What happens when a transaction enters geth and becomes part of the permanent chain?"** Before diving into any subsystem, here is a bird's-eye view of the four major workflows that answer this question. Each workflow is covered in detail by the chapters listed.

### Workflow 1: Node Startup

When you run `geth`, the following happens:

```
main()                          cmd/geth/main.go
  └─ geth()
       ├─ prepare()             Set cache defaults for mainnet
       ├─ makeFullNode()        cmd/geth/config.go
       │    ├─ loadBaseConfig() Parse flags + TOML config file
       │    ├─ node.New()       Create the service host (P2P, RPC servers)
       │    ├─ eth.New()        Create the Ethereum service:
       │    │    ├─ Open chaindata DB
       │    │    ├─ core.NewBlockChain()
       │    │    ├─ txpool.New()
       │    │    ├─ newHandler() (sync + peer management)
       │    │    ├─ miner.New()
       │    │    └─ RegisterLifecycle(eth) on the node
       │    └─ catalyst.Register()  Register Engine API
       ├─ startNode()
       │    └─ node.Start()     Start P2P, RPC, then all lifecycles
       └─ stack.Wait()          Block until shutdown signal
```

The key file is `cmd/geth/config.go:makeFullNode()` — it's the assembly point where every subsystem is created and wired together. Chapters 13 and 14 cover this in detail.

### Workflow 2: Transaction Lifecycle

A transaction enters geth through the JSON-RPC (`eth_sendRawTransaction`) or from a peer over the P2P network. Its journey:

```
 Arrival              JSON-RPC eth_sendRawTransaction  OR  P2P TransactionsMsg
    │
    ▼
 Validation           txpool.Add() → validate signature, nonce, balance, gas
    │
    ▼
 Pool Storage         legacypool: pending map (executable) or queued map (future nonce)
    │                 blobpool: on-disk storage for blob txs
    ▼
 Broadcast            event.Feed sends NewTxsEvent → handler broadcasts to peers
    │
    ▼
 Block Inclusion      miner pulls from pool, orders by effective tip, executes via EVM
    │
    ▼
 Permanent Storage    block written to chaindata DB, tx index updated
```

Chapters 06, 07, and 08 cover this path step by step.

### Workflow 3: Block Production (Post-Merge)

After the Merge, blocks are produced on demand from the consensus layer:

```
 CL calls ForkchoiceUpdatedV*     Sets the head block + requests payload building
    │
    ▼
 miner.BuildPayload()             Starts an async payload builder
    │
    ▼
 worker.generateWork()            miner/worker.go
    ├─ Prepare header             Set timestamp, gas limit, base fee, etc.
    ├─ Create StateDB at parent   core/state
    ├─ fillTransactions()          Pull txs from pool, sort by tip, execute each
    └─ engine.FinalizeAndAssemble() Apply withdrawals, system txs, assemble block
    │
    ▼
 CL calls GetPayloadV*            Retrieves the built payload
    │
    ▼
 CL calls NewPayloadV*            Sends the payload to other EL clients for validation
    │
    ▼
 InsertChain()                    Validate + execute + write to DB
```

Chapter 09 covers the miner and consensus engine. Chapter 10 covers `InsertChain()`.

### Workflow 4: Chain Synchronization

A new node must download the existing chain from peers:

```
 Peer connection          P2P handshake, exchange status (genesis hash, head, fork ID)
    │
    ▼
 Sync mode selection      Snap sync (default) or full sync
    │
    ▼
 Snap sync pipeline       eth/downloader
    ├─ Skeleton sync      Download headers in reverse (anchor → pivot → genesis)
    ├─ Body download      Fetch block bodies for skeleton headers
    ├─ Receipt download   Fetch receipts (skip execution for pre-pivot blocks)
    ├─ State download     Snap protocol: download flat account + storage data
    └─ State healing      Fill in any missing trie nodes
    │
    ▼
 Switch to full sync      After pivot: execute every block normally
    │
    ▼
 Caught up                Node is in sync, processes new blocks as they arrive
```

Chapters 11 and 12 cover networking and sync.

---

## Where to Start Reading

Depending on your interest, here are recommended reading paths through the guide:

**"I want to understand how transactions are executed"**
→ Ch 01 (Primitives) → Ch 02 (Data Types) → Ch 06 (Tx Execution) → Ch 07 (EVM)

**"I want to understand state and storage"**
→ Ch 01 (Primitives) → Ch 03 (Merkle Patricia Trie) → Ch 04 (Account & State) → Ch 05 (Storage Stack)

**"I want to understand block production and the chain"**
→ Ch 02 (Data Types) → Ch 09 (Block Production) → Ch 10 (The Blockchain)

**"I want to understand networking and sync"**
→ Ch 11 (P2P) → Ch 12 (Sync & Wire Protocol)

**"I want to understand the full picture, top to bottom"**
→ Read chapters 00 through 14 in order. Each chapter builds on the previous ones.

---

## What's Next

With this map in hand, we begin the bottom-up tour. Chapter 01 — Primitives, Configuration, and Encoding introduces the foundational types (`Address`, `Hash`), the chain configuration system, RLP encoding, and the cryptographic primitives that every other package depends on.
