---
title: Geth(15) Tying It All Together
published: 2026-04-24
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

This guide has been driven by a single question: **"What happens when a transaction enters geth and becomes part of the permanent chain?"** Over fourteen chapters, we examined every subsystem that participates in that journey. This final chapter traces the complete path in one continuous narrative — from the moment a user submits a transaction to the moment it is sealed in a finalized block — then provides a reference map of the entire architecture.

---

## The Life of a Transaction

The diagram below shows the full path. Each numbered stage is explained in detail afterwards, with references to the chapter where the subsystem was covered.

```
  User / DApp
       |
       |  eth_sendRawTransaction (JSON-RPC)
       v
  +------------------+
  | RPC Server       |  <-- Ch 13: transport, dispatch, method reflection
  | (rpc/server.go)  |
  +--------+---------+
           |
           v
  +------------------+
  | TransactionAPI   |  <-- Ch 13: SendRawTransaction, SubmitTransaction
  | (ethapi/api.go)  |
  +--------+---------+
           |  tx.UnmarshalBinary -> types.Transaction
           |  txPool.Add()
           v
  +------------------+
  | Transaction Pool |  <-- Ch 08: validation, pending/queued maps, blob pool
  | (core/txpool/)   |
  +--------+---------+
           |
           |  NewTxsEvent via event.Feed
           v
  +-------------------+
  | Handler           |  <-- Ch 12: txBroadcastLoop, txAnnounceLoop
  | (eth/handler.go)  |---> broadcast to peers (eth/68 protocol)
  +-------------------+
           |
           |  Consensus client calls engine_forkchoiceUpdatedV*
           |  with payloadAttributes (triggers block building)
           v
  +-------------------+
  | Miner             |  <-- Ch 09: BuildPayload, generateWork
  | (miner/worker.go) |
  +--------+----------+
           |  fillTransactions: pull from pool, sort by tip, execute each
           |
           v
  +--------------------+
  | State Transition   |  <-- Ch 06: preCheck, buyGas, EVM dispatch, gas refund
  | (core/             |
  |  state_transition) |
  +--------+-----------+
           |
           |  evm.Call() or evm.Create()
           v
  +--------------------+
  | EVM                |  <-- Ch 07: Run loop, opcodes, gas accounting
  | (core/vm/)         |
  +--------+-----------+
           |
           |  state mutations (SSTORE, CREATE, SELFDESTRUCT, etc.)
           v
  +--------------------+
  | StateDB            |  <-- Ch 04: stateObject, dirty maps, journal, snapshots
  | (core/state/)      |
  +--------+-----------+
           |
           |  Miner calls FinalizeAndAssemble
           v
  +--------------------+
  | Block Assembly     |  <-- Ch 09: header finalization, receipt root, state root
  | (consensus/beacon) |
  +--------+-----------+
           |
           |  CL calls engine_getPayloadV* -> engine_newPayloadV*
           |  then engine_forkchoiceUpdatedV* (sets new head)
           v
  +--------------------+
  | BlockChain         |  <-- Ch 10: InsertChain, processBlock, writeBlockWithState
  | (core/blockchain)  |
  +--------+-----------+
           |
           |  writeBlockAndSetHead -> ChainHeadEvent
           v
  +--------------------+
  | Storage Stack      |  <-- Ch 03, 04, 05: trie commit, triedb, ethdb (LevelDB/Pebble)
  | (trie/ + ethdb/)   |
  +--------------------+
           |
           |  Block propagated to peers
           v
  +--------------------+
  | P2P Network        |  <-- Ch 11, 12: NewBlockMsg broadcast, snap sync for lagging peers
  | (p2p/ + eth/)      |
  +--------------------+
```

### Stage 1: RPC Arrival

A user (or DApp) submits a signed transaction by calling `eth_sendRawTransaction` over HTTP, WebSocket, or IPC. The RPC server (Chapter 13) receives the request, splits the method name on `_` to find the `eth` namespace, and dispatches to `TransactionAPI.SendRawTransaction()`.

`SendRawTransaction` binary-decodes the raw bytes via `tx.UnmarshalBinary()` into a `types.Transaction` (Chapter 02) and calls `SubmitTransaction()`, which adds the transaction to the pool.

### Stage 2: Transaction Pool

The transaction enters the `TxPool` coordinator (Chapter 08), which routes it to the appropriate sub-pool — `LegacyPool` for regular transactions, `BlobPool` for EIP-4844 blob transactions. The sub-pool validates the transaction: signature recovery, nonce check, balance check, gas limit check, and pool-level limits (account slots, global slots, price floor).

If validation passes, the transaction lands in the pool's **pending** map — it is ready for inclusion in a block. The pool publishes a `NewTxsEvent` via an `event.Feed` (Chapter 14), notifying the handler.

### Stage 3: Peer Broadcast

The `handler` in `eth/handler.go` (Chapter 12) picks up the `NewTxsEvent` and broadcasts the transaction to connected peers. For each peer, the handler decides between two strategies:

- **Direct broadcast** — send the full transaction to a small fraction of peers (square root of total).
- **Hash announcement** — send only the transaction hash to the remaining peers, who can request the full transaction if they don't have it.

This is the `eth/68` wire protocol at work. The transaction propagates across the network, landing in the pools of other nodes.

### Stage 4: Block Building

Block building is triggered by the consensus client. When the CL calls `engine_forkchoiceUpdatedV*` with `payloadAttributes`, geth's Engine API (Chapter 09) tells the miner to start building a payload.

The miner's `generateWork()` method:

1. Prepares a block header with the parent hash, timestamp, and other consensus fields.
2. Creates a `StateDB` snapshot at the parent block's state root.
3. Calls `fillTransactions()` — pulls pending transactions from the pool, sorts them by effective tip (highest-paying first), and executes each one.

### Stage 5: Transaction Execution

For each transaction, the miner calls `core.ApplyTransaction()`, which creates a `stateTransition` and runs its `execute()` method (Chapter 06). The execution pipeline:

1. **`preCheck()`** — verify the sender's nonce, balance, and call `buyGas()` to deduct the upfront gas cost and reduce the block's `GasPool`.
2. **EVM dispatch** — call `evm.Call()` for message calls or `evm.Create()` for contract creation.
3. **Gas refund** — `calcRefund()` computes the refund (capped at 1/5 of gas used since EIP-3529), then `returnGas()` credits the remaining gas back to the sender.
4. **Fee distribution** — the priority fee (tip) goes to the coinbase; the base fee is burned.

### Stage 6: EVM Execution

Inside `evm.Call()`, the `EVM.Run()` loop (Chapter 07) takes over:

1. Fetch the next opcode from the contract's bytecode.
2. Look up the opcode in the fork-specific jump table to get its gas cost and handler function.
3. Deduct gas. If insufficient, abort with an out-of-gas error.
4. Execute the handler — arithmetic (`opAdd`), storage (`opSstore`, `opSload`), calls (`opCall`, `opDelegateCall`), or any other EVM operation.
5. Repeat until `STOP`, `RETURN`, `REVERT`, or an error.

State-mutating opcodes like `SSTORE` write to the `StateDB` (Chapter 04), which records changes in dirty maps on the corresponding `stateObject`. These changes are not yet committed to the trie — they exist only in memory, protected by the journal's undo log so they can be reverted if the transaction fails.

### Stage 7: State and Trie

After all transactions are executed, the miner calls the consensus engine's `FinalizeAndAssemble()`, which internally invokes `Finalize()` to apply block-level state changes (beacon withdrawals, system-level operations), then computes the final state root by committing dirty state through the trie (Chapter 03):

- Each modified account's storage trie is updated and hashed.
- The account trie is updated with new account states and hashed.
- The resulting 32-byte Merkle root becomes the block header's `StateRoot`.

The assembled block — header, transaction list, receipts, and withdrawals — is the **payload** returned to the consensus client via `engine_getPayloadV*`.

### Stage 8: Block Insertion

The consensus client validates the block on the beacon chain, then sends it back to geth via `engine_newPayloadV*`. Geth runs `InsertBlockWithoutSetHead()` (Chapter 10):

1. **Header verification** — check timestamp, gas limit, extra data, and other consensus constraints.
2. **Body validation** — verify transaction root, uncles hash, withdrawals hash against the header.
3. **State processing** — re-execute all transactions (identical to step 5-6 above) and compare the resulting state root against the header's `StateRoot`. If they don't match, the block is rejected.
4. **Write to database** — block header, body, receipts, and state trie nodes are persisted to the key-value store (LevelDB or Pebble, Chapter 05).

When the CL calls `engine_forkchoiceUpdatedV*` designating this block as the new head, geth updates its canonical chain pointers and emits a `ChainHeadEvent`.

### Stage 9: Persistence and Propagation

The committed state flows down the storage stack:

- **StateDB** flushes dirty state objects to the **trie** (Chapter 03).
- The trie commits new nodes to **TrieDB** — either the path-based or hash-based scheme (Chapter 05).
- TrieDB eventually flushes to **ethdb** (LevelDB or Pebble) on disk.
- Old blocks are migrated to the **freezer** (append-only ancient storage) to keep the active database lean.

Simultaneously, the handler broadcasts the new block to peers (Chapter 12) — full block to a few, block hash to the rest. Peers that are behind can use snap sync or full sync to catch up.

### Stage 10: Finality

The consensus client eventually marks the block (or an ancestor) as **finalized** — meaning it will never be reverted. Geth updates its `currentFinalBlock` pointer (Chapter 10). The transaction is now permanently part of the canonical chain.

---

## Architecture Reference

The table below maps every major subsystem to its chapter, primary source files, and the key structs or interfaces that define it.

### Foundation Layer

| Chapter | Title | Key Files | Key Types |
|---------|-------|-----------|-----------|
| 00 | Codebase Overview | `cmd/geth/main.go`, `node/node.go`, `eth/backend.go` | — |
| 01 | Primitives, Configuration, and Encoding | `common/common.go`, `params/config.go`, `rlp/encode.go`, `crypto/crypto.go` | `Address`, `Hash`, `ChainConfig` |
| 02 | Core Data Types | `core/types/block.go`, `core/types/transaction.go`, `core/types/receipt.go` | `Block`, `Header`, `Transaction`, `Receipt`, `Log` |

### State Architecture

| Chapter | Title | Key Files | Key Types |
|---------|-------|-----------|-----------|
| 03 | Merkle Patricia Trie | `trie/trie.go`, `trie/node.go`, `trie/hasher.go`, `trie/stacktrie.go` | `Trie`, `fullNode`, `shortNode`, `StackTrie` |
| 04 | Account and State | `core/state/statedb.go`, `core/state/state_object.go`, `core/state/journal.go` | `StateDB`, `stateObject`, `journal` |
| 05 | The Storage Stack | `ethdb/database.go`, `ethdb/leveldb/`, `ethdb/pebble/`, `core/rawdb/schema.go` | `KeyValueStore`, `Database`, `Freezer` |

### Execution

| Chapter | Title | Key Files | Key Types |
|---------|-------|-----------|-----------|
| 06 | Transaction Execution | `core/state_transition.go`, `core/state_processor.go`, `core/gaspool.go` | `stateTransition`, `GasPool`, `ExecutionResult` |
| 07 | The EVM Deep Dive | `core/vm/evm.go`, `core/vm/interpreter.go`, `core/vm/jump_table.go` | `EVM`, `Contract`, `JumpTable` |

### Chain Operations

| Chapter | Title | Key Files | Key Types |
|---------|-------|-----------|-----------|
| 08 | The Transaction Pool | `core/txpool/txpool.go`, `core/txpool/legacypool/`, `core/txpool/blobpool/` | `TxPool`, `LegacyPool`, `BlobPool` |
| 09 | Block Production and Consensus | `miner/worker.go`, `miner/payload_building.go`, `consensus/consensus.go`, `eth/catalyst/api.go` | `Miner`, `Engine`, `Payload` |
| 10 | The Blockchain | `core/blockchain.go`, `core/headerchain.go`, `core/genesis.go` | `BlockChain`, `HeaderChain` |

### Networking

| Chapter | Title | Key Files | Key Types |
|---------|-------|-----------|-----------|
| 11 | P2P Networking and Discovery | `p2p/server.go`, `p2p/peer.go`, `p2p/rlpx/rlpx.go`, `p2p/discover/` | `Server`, `Peer`, `UDPv4`, `UDPv5` |
| 12 | Sync and the Ethereum Wire Protocol | `eth/handler.go`, `eth/downloader/`, `eth/fetcher/`, `eth/protocols/eth/` | `handler`, `Downloader`, `TxFetcher` |

### Interface and Lifecycle

| Chapter | Title | Key Files | Key Types |
|---------|-------|-----------|-----------|
| 13 | JSON-RPC and Accounts | `rpc/server.go`, `rpc/handler.go`, `internal/ethapi/api.go`, `accounts/` | `Server`, `Backend`, `EthAPIBackend`, `KeyStore` |
| 14 | Node Lifecycle | `cmd/geth/main.go`, `node/node.go`, `eth/backend.go`, `event/feed.go` | `Node`, `Lifecycle`, `Ethereum`, `Feed` |

---

## Cross-Cutting Patterns

Several design patterns appear across all subsystems. Recognizing them makes the codebase easier to navigate.

### The Lifecycle Pattern

Many long-lived components follow a similar contract — setup methods that launch goroutines, and teardown methods that shut them down. The `Node` orchestrates the formal `Lifecycle` interface (`Start()`/`Stop()`) for registered services (Chapter 14), starting them in registration order and stopping them in reverse order. Components that implement this interface include:

- `Ethereum` (eth/backend.go) — the core service
- `handler` (eth/handler.go) — sync and broadcast
- `Server` (p2p/server.go) — P2P networking

Other components use variant naming but follow the same pattern — `LegacyPool` and `BlobPool` use `Init()`/`Close()`, and `BlockChain` is initialized via `NewBlockChain()` and torn down via `Stop()`.

### The Event Feed Pattern

Decoupled communication between subsystems uses `event.Feed` (Chapter 14). A producer calls `Send()`, and all subscribers receive the value on their channels. Key feeds in geth:

| Feed | Producer | Consumers | Event Type |
|------|----------|-----------|------------|
| `chainHeadFeed` | `BlockChain` | miner, handler, filter system | `ChainHeadEvent` |
| (via `SubscribeTransactions`) | `TxPool` | handler (for broadcast) | `NewTxsEvent` |
| `walletEvent` | account backends | `startNode()` wallet listener | `WalletEvent` |

### The Backend Interface Pattern

API layers are decoupled from implementation through interfaces. The RPC methods in `internal/ethapi/` talk to a `Backend` interface (Chapter 13), which `EthAPIBackend` implements by delegating to `BlockChain`, `TxPool`, `Miner`, and other concrete types. This allows the same API code to work in full node, light client, and test contexts.

### The Config Struct Pattern

Every major subsystem has a `Config` struct that aggregates its tunable parameters:

| Config | Package | Controls |
|--------|---------|----------|
| `node.Config` | `node/config.go` | Data directory, P2P settings, RPC endpoints |
| `ethconfig.Config` | `eth/ethconfig/config.go` | Sync mode, cache sizes, gas price, tx pool limits |
| `p2p.Config` | `p2p/config.go` | Max peers, listen address, NAT, bootnodes |
| `ChainConfig` | `params/config.go` | Fork activation timestamps, chain ID, consensus rules |
| `TxPool.Config` | `core/txpool/legacypool/` | Price limits, slot limits, journal settings |

CLI flags map to these config structs during startup (Chapter 14): `utils.SetNodeConfig()` writes to `node.Config`, `utils.SetEthConfig()` writes to `ethconfig.Config`, and so on.

### The Four-Layer Storage Model

Data flows through four layers on its way to disk (Chapters 03, 04, 05):

```
  StateDB         in-memory dirty maps, journal for rollback
     |
     v
  Trie            Merkle Patricia Trie nodes (fullNode, shortNode)
     |
     v
  TrieDB          caching layer (path-based or hash-based scheme)
     |
     v
  ethdb           key-value store (LevelDB or Pebble) + freezer for ancient data
```

Reads traverse this stack top-down: `StateDB` checks its dirty cache, falls through to the trie, which falls through to TrieDB, which falls through to disk. Writes accumulate at the top and flush downward on commit.

---

## Reading Paths

Depending on your interest, here are suggested paths through the chapters:

**"I want to understand transaction execution"**
Ch 01 (primitives) -> Ch 02 (data types) -> Ch 06 (execution pipeline) -> Ch 07 (EVM) -> Ch 04 (state)

**"I want to understand block production"**
Ch 02 (data types) -> Ch 08 (tx pool) -> Ch 09 (block production) -> Ch 06 (execution) -> Ch 10 (blockchain)

**"I want to understand state storage"**
Ch 01 (primitives) -> Ch 03 (trie) -> Ch 04 (state) -> Ch 05 (storage stack)

**"I want to understand networking and sync"**
Ch 11 (P2P) -> Ch 12 (sync protocol) -> Ch 10 (blockchain) -> Ch 08 (tx pool)

**"I want to understand the RPC interface"**
Ch 13 (JSON-RPC) -> Ch 04 (state, for eth_call) -> Ch 06 (execution, for eth_call internals)

**"I want to understand how geth starts up"**
Ch 14 (node lifecycle) -> Ch 00 (overview) -> then any subsystem chapter

---

## The Central Answer

The guide's central question was: *What happens when a transaction enters geth and becomes part of the permanent chain?*

The answer, compressed to one paragraph:

A signed transaction arrives via JSON-RPC (Ch 13), is validated and stored in the transaction pool (Ch 08), broadcast to peers over the eth/68 wire protocol (Ch 12), pulled by the miner when the consensus client requests a payload (Ch 09), executed through the state transition pipeline (Ch 06) and the EVM (Ch 07), mutating the world state held in StateDB (Ch 04) and committed through the Merkle Patricia Trie (Ch 03). The resulting block is inserted into the blockchain (Ch 10), persisted through the storage stack to LevelDB or Pebble (Ch 05), propagated to peers (Ch 11, Ch 12), and eventually marked as finalized — all within a process orchestrated by the Node lifecycle (Ch 14) using the primitives and configuration established at the foundation (Ch 01, Ch 02).
