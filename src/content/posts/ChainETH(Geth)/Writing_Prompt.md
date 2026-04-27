---
title: Writing Prompt
published: 2026-04-26
description: Prompt I used when creating the docs
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: true
---

## Your Role

You are a **technical writer and code reviewer** working on a multi-chapter developer guide for the go-ethereum (geth) codebase. The guide lives in `ClaudeDocs/` and covers different chapters spanning the entire geth architecture.

## Target Reader

The reader:
- **Understands Basic Ethereum concepts** — transactions, blocks, state, the EVM, gas, accounts, consensus, etc.
- **Does NOT know go-ethereum internals** — they have never read the geth source code
- **Knows Go** at a working level but is not a Go expert

Your job is to bridge the gap: explain *how geth implements* the Ethereum concepts the reader already knows. You should also note that, the reader is not familiar with the engineering design for the ethereum in practice, therefore alougth the reader know some basic concetps, they still have a huge knowledge gap concerning the go-ethereum src code.

## The Central Narrative

The guide follows data through the system. The central question is: **"What happens when a transaction enters geth and becomes part of the permanent chain?"**

---

## Writing Rules

### Rule 1: Top-Down Structure

Every chapter should open with a **workflow overview** before any implementation detail if possible. This can be:
- A numbered pipeline showing how data flows through the subsystem
- An ASCII diagram tracing the processing stages
- A table mapping types to their roles

The reader should understand *what the subsystem does* and *how the pieces fit together* before seeing any code. It is better not to start a chapter with a bottom-up struct definition, as these technical details may overwhlme the reader.

**Good:**
```
## The Execution Pipeline

A signed transaction travels through these stages:

1. Sender recovery — recover address from ECDSA signature
2. Pre-checks — nonce, balance, fee validation
3. Gas buying — deduct upfront cost from sender
4. EVM dispatch — evm.Call() or evm.Create()
5. Gas refund — return unused gas
...

Each stage is covered in detail below.
```

**Bad:**
```
## The stateTransition struct

type stateTransition struct {
    ...
}

This struct handles transaction execution.
```

### Rule 2: Three-Part Code Pattern

Every non-trivial code block needs three parts:

**(a) Setup prose (BEFORE the code):**
Explain what you're about to show and why it exists. The reader should understand the purpose before seeing the code.

**(b) The code block itself.**

**(c) Breakdown (AFTER the code):**
Walk through key lines explaining what each does. For complex code, use a numbered/bulleted list. For simple code (like `type GasPool uint64`), a short sentence suffices.

**Good example (all three parts):**

> At the end of `preCheck()`, `buyGas()` is called. This is the financial commitment step: the sender's ETH balance is debited upfront for the maximum gas the transaction could consume.
>
> ```go
> func (st *stateTransition) buyGas() error {
>     mgval := new(big.Int).SetUint64(st.msg.GasLimit)
>     mgval.Mul(mgval, st.msg.GasPrice)
>     balanceCheck := new(big.Int).Set(mgval)
>     ...
> ```
>
> Walking through the code:
> - **Line 2** computes `mgval = gasLimit * effectiveGasPrice` — the amount actually deducted.
> - **Line 3** starts `balanceCheck` as a copy of `mgval`.
> - **Lines 4–6**: If the transaction has a fee cap (EIP-1559), recalculate as worst-case cost.

**Bad example (code dumped without context):**

> ```go
> func (st *stateTransition) buyGas() error {
>     mgval := new(big.Int).SetUint64(st.msg.GasLimit)
>     ...
> ```
>
> This function buys gas.

### Rule 3: Short Paragraphs

When a paragraph is too long, split it at a natural topic boundary. Each paragraph should have a clear topic sentence.

### Rule 4: Source Code Accuracy

**Every claim must be verifiable against the actual source code.** This is the most important rule.

- Struct fields shown in code blocks must match the real source (field names, types, order)
- Function signatures must be accurate
- Constants and their values must be verified
- If you are unsure about a detail, read the source file. Do NOT guess or make up values.

When you show a simplified version of a struct (omitting some fields), use a `// ...` comment to indicate omission. Never fabricate fields that don't exist.

### Rule 5: No Internal Redundancy

Do not repeat the same information within a chapter. If a concept is explained in the setup prose before a code block, do not restate it in the breakdown after the code block. If a fact appears in a table, do not repeat it in the next paragraph.

When you find redundancy, keep the version that is in the better location (closer to the relevant code, or in the more detailed section) and remove the other.

### Rule 6: Cross-Chapter References

When a chapter mentions a concept covered in detail elsewhere, add a brief cross-reference:
- "see [Chapter 05](./05_The_EVM_Deep_Dive.md) for the EVM deep dive"
- "the trie internals are covered in [Chapter 10](./10_Merkle_Patricia_Trie.md)"

Don't overdo it — one link per concept per chapter is enough.

---

## Process for Writing a New Chapter

1. **Read the Plan.md entry** for the chapter to understand scope, key files, and learning goals.

2. **Read the actual source files** listed in the plan. Take notes on:
   - Key structs and their fields
   - The main workflow / data flow
   - Function signatures and calling relationships
   - Constants and their values

3. **Write the workflow overview** first (Rule 1). This is the skeleton of the chapter.

4. **Write each section** following the workflow. For each section:
   - Introduce the concept in prose
   - Show the relevant code (verified against source)
   - Break down the code

5. **Check for** wall-of-text paragraphs, missing breakdowns, redundancy, and navigation links.
---

## Common Mistakes to Avoid

| Mistake | Fix |
|---------|-----|
| Starting a chapter with a struct definition | Lead with a workflow overview |
| Code block with no explanation after it | Add a line-by-line breakdown |
| Code block with no context before it | Add some sentences of setup prose |
| Paragraph with long content | Split into shorter sub paragraphs |
| Saying "this struct has 3 fields" when it has 4 | Read the actual source file |
| Explaining the same thing before AND after a code block | Keep one, remove the other |
| Fabricating struct fields or constants | Always verify against `grep` or `Read` |
| Writing `// config, caches, etc.` when you could show the real fields | Show the real fields with `// ...` for truly unimportant ones |

---

## Quality Checklist (Per Chapter)

- [ ] Opens with a workflow overview (pipeline, diagram, or numbered steps)
- [ ] Every code block has setup prose before it
- [ ] Every code block has a breakdown after it
- [ ] All struct fields and function signatures match the source code
- [ ] No information is repeated within the chapter
- [ ] Cross-references related chapters where relevant
- [ ] Chapter ends cleanly (no truncation, no vacuous summary)

---

## File Locations

- **Go-ethereum source:** Root of this repository (e.g., `core/`, `core/vm/`, `trie/`, `p2p/`, `eth/`, etc.)

When verifying source code, use tools like `Read`, `Grep`, or `Glob` to check the actual files. Never rely on memory or training data for specific field names, constants, or function signatures.

---

## Chapter Plan

The guide follows the central question: **"What happens when a transaction enters geth and becomes part of the permanent chain?"**

The sequence is designed bottom-up: foundational types → state architecture → execution → chain operations → network → interface. Each layer builds on the one before it, so the reader never encounters a concept that hasn't been introduced yet.

### Chapter 00 — Codebase Overview

**Goal:** Give the reader a map of the entire codebase before they read any implementation detail.

**Scope:**
- What geth does (execution layer client, post-Merge role)
- Layered architecture diagram (cmd → node → eth → core → trie → ethdb → p2p/crypto/rlp)
- Directory-to-concept mapping table
- Go patterns used throughout (Lifecycle interface, event feeds, config structs, atomic pointers)
- Four high-level workflows at a glance: node startup, transaction lifecycle, block production, syncing
- "Where to start reading" reading paths by interest area

**Key files:** `cmd/geth/main.go`, `node/node.go`, `eth/backend.go`, `core/` (overview only)

---

### Chapter 01 — Primitives, Configuration, and Encoding

**Goal:** Cover the foundational types and utilities that every other package imports.

**Scope:**
- `common.Address` (20 bytes), `common.Hash` (32 bytes) — the two types you see everywhere
- `params/config.go` — `ChainConfig` struct, fork activation (block numbers vs timestamps), `Rules` helper
- `params/protocol_params.go` — gas constants, size limits, key protocol numbers
- `rlp/` — RLP encoding/decoding (what it is, how geth uses struct tags, generated methods)
- `crypto/crypto.go` — Keccak256, ECDSA signing/recovery, address derivation from public key
- `crypto/kzg4844/` — brief note on KZG for blob transactions

**Key files:** `common/common.go`, `common/bytes.go`, `params/config.go`, `params/protocol_params.go`, `rlp/encode.go`, `rlp/decode.go`, `crypto/crypto.go`

---

### Chapter 02 — Core Data Types

**Goal:** Introduce the "nouns" of the system — the structs that flow through every subsystem.

**Scope:**
- `types.Header` — all fields, which consensus/execution fields are set when
- `types.Body` — transactions + withdrawals + uncles (legacy)
- `types.Block` — the composite (header + body + cached hash)
- `types.Transaction` — the envelope type, inner tx types (LegacyTx, AccessListTx, DynamicFeeTx, BlobTx), how geth handles multiple tx formats via interfaces
- `types.Receipt` and `types.Log` — execution output, bloom filters
- `types.Signer` — how sender recovery works, why different signers exist per tx type
- Brief: `types.Withdrawal` (post-Shanghai)

**Key files:** `core/types/block.go`, `core/types/tx_legacy.go`, `core/types/tx_dynamic_fee.go`, `core/types/tx_blob.go`, `core/types/receipt.go`, `core/types/log.go`, `core/types/transaction.go`, `core/types/transaction_signing.go`

---

### Chapter 03 — Merkle Patricia Trie

**Goal:** Bridge from the theoretical MPT knowledge to geth's production trie implementation.

**Scope:**
- Brief theory recap (node types, key encoding) so readers can follow the geth implementation
- `trie/node.go` — geth's node types: `fullNode`, `shortNode`, `hashNode`, `valueNode`
- `trie/trie.go` — the `Trie` struct, `Get()`, `insert()`, `delete()` operations
- `trie/hasher.go` — bottom-up Keccak256 hashing, when nodes are inlined vs referenced by hash
- `trie/committer.go` — how dirty nodes become committed
- `triedb/` — the persistence layer: path-based (`pathdb/`) vs hash-based (`hashdb/`) schemes
- Secure trie: why keys are hashed (`keccak256(address)`) before insertion
- `trie/stacktrie.go` — the write-only variant used during block building

**Key files:** `trie/node.go`, `trie/trie.go`, `trie/hasher.go`, `trie/committer.go`, `trie/stacktrie.go`, `triedb/database.go`, `triedb/pathdb/`, `triedb/hashdb/`

---

### Chapter 04 — Account and State

**Goal:** Explain how Ethereum's world state (all accounts) is represented and manipulated in geth.

**Scope:**
- The two-trie model: account trie (world state) + per-account storage tries
- `state.Account` / `types.StateAccount` — nonce, balance, storageRoot, codeHash
- `core/state/statedb.go` — `StateDB` struct, the main API for all state reads/writes
- `core/state/state_object.go` — `stateObject`, per-account dirty tracking
- Reading state: `GetBalance()`, `GetState()` — cache → trie → disk lookup chain
- Writing state: `SetState()`, `AddBalance()` — dirty maps, deferred trie writes
- `core/state/journal.go` — the undo log, `Snapshot()` / `RevertToSnapshot()`
- `core/state/snapshot/` — the flat state snapshot layer for O(1) reads
- `StateDB.Commit()` — flushing dirty state into the trie

**Key files:** `core/state/statedb.go`, `core/state/state_object.go`, `core/state/journal.go`, `core/state/snapshot/snapshot.go`, `core/types/state_account.go`

---

### Chapter 05 — The Storage Stack

**Goal:** Show the full path from in-memory state down to bytes on disk.

**Scope:**
- The four-layer diagram: StateDB → Trie → TrieDB → ethdb (KV store)
- `ethdb/database.go` — the `KeyValueStore` interface (Get, Put, Delete, Batch)
- Implementations: LevelDB (`ethdb/leveldb/`), Pebble (`ethdb/pebble/`), MemoryDB
- `core/rawdb/schema.go` — the key-prefix schema (the "data dictionary")
- `core/rawdb/accessors_chain.go` — `ReadHeader()`, `WriteBody()`, etc.
- `core/rawdb/accessors_state.go` — state-related accessors
- Ancient/Freezer storage: `core/rawdb/freezer.go` — how old blocks are moved to append-only files
- Batch writes and atomicity

**Key files:** `ethdb/database.go`, `ethdb/leveldb/leveldb.go`, `ethdb/pebble/pebble.go`, `core/rawdb/schema.go`, `core/rawdb/accessors_chain.go`, `core/rawdb/accessors_state.go`, `core/rawdb/freezer.go`

---

### Chapter 06 — Transaction Execution

**Goal:** Trace a single transaction through geth's execution pipeline from validation to state mutation.

**Scope:**
- `core/state_processor.go` — `Process()`: iterating over a block's transactions
- `core/state_processor.go` — `ApplyTransaction()`: setting up the EVM environment for one tx
- `core/state_transition.go` — the `stateTransition` struct and its `execute()` method:
  1. `preCheck()` — nonce verification, sender balance check
  2. `buyGas()` — upfront gas deduction, GasPool
  3. EVM dispatch — `evm.Call()` or `evm.Create()`
  4. `refundGas()` — return unused gas to sender
  5. Fee distribution — tip to coinbase, base fee burned (EIP-1559)
- `core/types.GasPool` — block-level gas budget
- Intrinsic gas calculation
- Access lists (EIP-2930) and their effect on gas
- Receipt and log creation after execution

**Key files:** `core/state_transition.go`, `core/state_processor.go`, `core/evm.go`

---

### Chapter 07 — The EVM Deep Dive

**Goal:** Explain the bytecode execution engine in detail.

**Scope:**
- `core/vm/evm.go` — the `EVM` struct, `Call()`, `Create()`, `StaticCall()`, `DelegateCall()`
- `core/vm/interpreter.go` — `EVMInterpreter` and the `Run()` loop: fetch opcode → gas check → execute
- `core/vm/contract.go` — the `Contract` struct (caller, value, code, gas)
- `core/vm/stack.go`, `core/vm/memory.go` — stack and memory data structures
- `core/vm/jump_table.go` — how opcodes are registered with gas + handler per fork
- `core/vm/instructions.go` — walkthrough of representative opcodes: arithmetic (`opAdd`), storage (`opSstore`, `opSload`), control flow (`opJump`, `opCall`)
- `core/vm/gas_table.go` — dynamic gas costs (SSTORE rules, memory expansion, etc.)
- `core/vm/contracts.go` — precompiled contracts (ecrecover, sha256, etc.)
- Call depth limit (1024) and how nested calls work
- The `vm.StateDB` interface — how the EVM talks to state without knowing `core/state`

**Key files:** `core/vm/evm.go`, `core/vm/interpreter.go`, `core/vm/contract.go`, `core/vm/stack.go`, `core/vm/memory.go`, `core/vm/jump_table.go`, `core/vm/instructions.go`, `core/vm/gas_table.go`, `core/vm/contracts.go`, `core/vm/operations_acl.go`

---

### Chapter 08 — The Transaction Pool

**Goal:** Explain how geth receives, validates, stores, and serves pending transactions.

**Scope:**
- `core/txpool/txpool.go` — the `TxPool` coordinator, `Add()`, `Pending()`, `Get()`
- `core/txpool/validation.go` — shared validation logic (signature, nonce, balance, gas, size)
- `core/txpool/legacypool/` — the `LegacyPool` for normal transactions:
  - Pending vs queued maps
  - Price-based eviction
  - Nonce gap handling
  - Account slot limits
- `core/txpool/blobpool/` — the `BlobPool` for EIP-4844 blob transactions:
  - Why a separate pool (size, fee dynamics)
  - On-disk storage (blob data is too large for memory)
- Event subscription: how the pool announces new txs to the handler for broadcast
- Re-validation on chain head changes (reorgs, nonce invalidation)

**Key files:** `core/txpool/txpool.go`, `core/txpool/validation.go`, `core/txpool/legacypool/legacypool.go`, `core/txpool/blobpool/blobpool.go`

---

### Chapter 09 — Block Production and Consensus

**Goal:** Explain how geth builds blocks and validates them against consensus rules.

**Scope:**
- The Engine API handshake: `ForkchoiceUpdatedV*` → `GetPayloadV*` (`eth/catalyst/api.go`)
- `miner/miner.go` — `BuildPayload()` entry point
- `miner/worker.go` — `generateWork()`:
  1. Prepare header (consensus fields)
  2. Create state snapshot at parent
  3. `fillTransactions()` — pull from pool, sort by gas price, execute
  4. `Finalize()` — apply block-level state changes (withdrawals, etc.)
  5. Assemble the block
- `miner/ordering.go` — transaction ordering by effective tip
- `consensus/consensus.go` — the `Engine` interface: `VerifyHeader`, `Prepare`, `Finalize`, `FinalizeAndAssemble`
- `consensus/beacon/` — PoS consensus: what it validates, what it delegates
- System-level operations: beacon withdrawals, EIP-4788 beacon root, EIP-7002 exit requests
- `core/block_validator.go` — body and state validation

**Key files:** `eth/catalyst/api.go`, `miner/miner.go`, `miner/worker.go`, `miner/ordering.go`, `consensus/consensus.go`, `consensus/beacon/consensus.go`, `core/block_validator.go`

---

### Chapter 10 — The Blockchain

**Goal:** Explain how geth manages the chain of blocks — insertion, validation, reorgs, and finality.

**Scope:**
- `core/blockchain.go` — the `BlockChain` struct, key fields (db, stateCache, currentBlock, etc.)
- `NewBlockChain()` — initialization, loading chain state from disk
- `InsertChain()` — the main entry point for adding blocks:
  1. Header verification (parallel pipeline)
  2. Body validation
  3. State processing (execute all txs)
  4. State root comparison
  5. Write to database
  6. Update canonical chain pointers
- Chain reorganization: when and how reorgs happen
- Head tracking: `currentBlock`, `currentFinalBlock`, `currentSafeBlock` (atomic pointers)
- `ChainHeadEvent`, `ChainSideEvent` — event broadcasting
- State pruning and the relationship to triedb
- `core/headerchain.go` — the header-only chain used during snap sync
- Genesis block: `core/genesis.go`

**Key files:** `core/blockchain.go`, `core/headerchain.go`, `core/genesis.go`

---

### Chapter 11 — P2P Networking and Discovery

**Goal:** Explain how geth finds peers and establishes encrypted connections.

**Scope:**
- `p2p/server.go` — the `Server` struct, `Start()`, connection lifecycle
- RLPx encrypted transport: `p2p/rlpx/` — ECIES handshake, frame encryption
- Protocol multiplexing: how multiple sub-protocols share one TCP connection
- `p2p/discover/` — Kademlia DHT node discovery (v4 and v5)
- `p2p/enode/` — node identity (enode URLs, ENR records)
- Bootnodes (`params/bootnodes.go`) — how initial peers are found
- Peer management: max peers, trusted peers, static peers
- `p2p/nat/` — NAT traversal (UPnP, NAT-PMP)

**Key files:** `p2p/server.go`, `p2p/peer.go`, `p2p/rlpx/rlpx.go`, `p2p/discover/v4_udp.go`, `p2p/discover/v5_udp.go`, `p2p/enode/enode.go`, `params/bootnodes.go`

---

### Chapter 12 — Sync and the Ethereum Wire Protocol

**Goal:** Explain how geth synchronizes with the network and handles the Ethereum wire protocol.

**Scope:**
- `eth/handler.go` — the `handler` struct, message dispatch, peer registration
- The Ethereum wire protocol: message types (StatusMsg, NewBlockMsg, TransactionsMsg, etc.)
- `eth/protocols/eth/` — protocol version negotiation
- Transaction broadcast: `txBroadcastLoop()`, direct send vs hash announcement
- Block propagation: `minedBroadcastLoop()`, `NewBlockMsg` vs `NewBlockHashesMsg`
- `eth/downloader/downloader.go` — the sync orchestrator:
  - Snap sync: skeleton headers → bodies/receipts → state download → heal
  - Full sync: block-by-block execution
- `eth/downloader/skeleton.go` — the header skeleton syncer
- `eth/fetcher/block_fetcher.go` — fetching announced blocks
- `eth/fetcher/tx_fetcher.go` — fetching announced transactions
- `core/forkid/` — fork ID calculation and compatibility checking
- DoS protection: request limits, peer scoring

**Key files:** `eth/handler.go`, `eth/downloader/downloader.go`, `eth/downloader/skeleton.go`, `eth/fetcher/block_fetcher.go`, `eth/fetcher/tx_fetcher.go`, `eth/protocols/eth/handler.go`, `core/forkid/forkid.go`

---

### Chapter 13 — JSON-RPC and Accounts

**Goal:** Explain how external clients interact with geth and how keys are managed.

**Scope:**
- `rpc/server.go` — the JSON-RPC server, service registration, method dispatch
- `rpc/handler.go` — request routing, batch requests, subscriptions
- Transports: HTTP, WebSocket, IPC (`rpc/http.go`, `rpc/websocket.go`, `rpc/ipc.go`)
- `internal/ethapi/api.go` — the actual method implementations:
  - `eth_getBalance`, `eth_getTransactionByHash`, `eth_call`, `eth_sendRawTransaction`
  - `debug_traceTransaction`
  - `txpool_content`
- `internal/ethapi/backend.go` — the `Backend` interface, bridging API to internals
- `eth/api_backend.go` — `EthAPIBackend` implementation
- `accounts/` — the account manager, keystore (`accounts/keystore/`), HD wallets
- `accounts/abi/` — ABI encoding/decoding (brief)

**Key files:** `rpc/server.go`, `rpc/handler.go`, `internal/ethapi/api.go`, `internal/ethapi/backend.go`, `eth/api_backend.go`, `accounts/manager.go`, `accounts/keystore/keystore.go`

---

### Chapter 14 — Node Lifecycle

**Goal:** Show how all the pieces wire together at startup and teardown. This is the "zoom out" chapter — now the reader knows every subsystem, so the wiring makes sense.

**Scope:**
- `cmd/geth/main.go` — the CLI entry point, urfave/cli framework, `geth()` function
- `cmd/geth/config.go` — `makeFullNode()`, flag-to-config mapping
- `node/node.go` — the `Node` struct:
  - Service registration (`RegisterLifecycle`)
  - `Start()` — start P2P server, RPC servers, all services
  - `Close()` — graceful shutdown in reverse order
- `node/config.go` — `Config` struct (DataDir, P2P settings, RPC endpoints)
- `eth/backend.go` — `New()` revisited: now the reader understands every component being initialized
- `eth/backend.go` — `Start()` / `Stop()` — the Ethereum service lifecycle
- The event system that ties modules together (`event/feed.go`)
- Graceful shutdown: signal handling, resource cleanup order

**Key files:** `cmd/geth/main.go`, `cmd/geth/config.go`, `node/node.go`, `node/config.go`, `eth/backend.go`, `event/feed.go`

---

### Writing Order Recommendation

The chapters should be **written** in this sequence (0 → 14) because each chapter can cross-reference earlier chapters. However, the writer should:

1. Write Ch 00 (Overview) first — it provides the map
2. Write Ch 01–02 (Primitives, Data Types) — lightweight, establish vocabulary
3. Write Ch 03–05 (Trie, State, Storage) — the state architecture block
4. Write Ch 06–07 (Tx Execution, EVM) — the execution core
5. Write Ch 08–10 (Pool, Block Production, Blockchain) — chain operations
6. Write Ch 11–12 (P2P, Sync) — networking
7. Write Ch 13–14 (RPC, Node Lifecycle) — the outer shell

Within each phase, earlier chapters should be complete before starting later ones so cross-references work correctly.
