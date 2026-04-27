---
title: Geth(4) Account and State
published: 2026-04-13
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

Every Ethereum account — externally-owned accounts and contracts alike — lives in the **world state**. This chapter explains how geth represents accounts, organizes them in tries, caches reads and writes through `StateDB`, tracks changes with an undo log, and ultimately commits everything back to disk.

---

## How State Flows Through a Block

When geth processes a block, state changes follow this pipeline:

```
 1. StateDB is created from the parent block's state root
       │
       ▼
 2. For each transaction:
    a. EVM reads state    →  StateDB.GetBalance(), GetState(), ...
    b. EVM writes state   →  StateDB.AddBalance(), SetState(), ...
    c. Changes land in dirty maps inside stateObject (not in the trie yet)
    d. If the tx reverts   →  journal replays undo entries
       │
       ▼
 3. After each transaction:
    StateDB.Finalise()  →  moves dirty storage to pending,
                            deletes empty/self-destructed accounts
       │
       ▼
 4. IntermediateRoot()  →  flushes pending storage into per-account tries,
                            updates account trie, returns new state root
       │
       ▼
 5. StateDB.Commit()   →  commits all tries, writes trie nodes + code + 
                            snapshot updates to the database
```

The key insight: geth defers trie writes as long as possible. During execution, all mutations live in fast Go maps. Only at commit time do they flow into the Merkle Patricia Tries covered in Chapter 03.

---

## The Two-Trie Model

Ethereum's state is organized as a trie-of-tries:

```
                    Account Trie
                  (world state root)
                   /     |     \
                  /      |      \
          Account A  Account B  Account C
          (EOA)      (contract)  (contract)
                        |           |
                   Storage Trie  Storage Trie
                    /    |         /    \
                 slot0 slot1    slot0  slot1
```

- The **account trie** maps `keccak256(address)` → RLP-encoded account data. Its root hash is the `Root` field in every block header (see Chapter 02).
- Each contract account has its own **storage trie** that maps `keccak256(storageSlot)` → RLP-encoded value. The storage trie's root hash is stored inside the account data.

Both tries are `StateTrie` instances (key-hashing tries covered in Chapter 03).

---

## The StateAccount Struct

Each account is represented on disk by a four-field struct in `core/types/state_account.go`:

```go
// core/types/state_account.go

type StateAccount struct {
    Nonce    uint64
    Balance  *uint256.Int
    Root     common.Hash // merkle root of the storage trie
    CodeHash []byte
}
```

- **`Nonce`** — the transaction count (EOAs) or contract creation count (contracts). Incremented with each transaction sent from the account.
- **`Balance`** — the account's ETH balance in wei. Uses `uint256.Int` (256-bit integer) rather than `big.Int` for performance.
- **`Root`** — the root hash of this account's storage trie. For accounts with no storage, this is `types.EmptyRootHash`.
- **`CodeHash`** — the Keccak256 hash of the account's contract bytecode. For EOAs (non-contract accounts), this is `types.EmptyCodeHash`.

The `NewEmptyStateAccount()` constructor shows the defaults:

```go
// core/types/state_account.go

func NewEmptyStateAccount() *StateAccount {
    return &StateAccount{
        Balance:  new(uint256.Int),
        Root:     EmptyRootHash,
        CodeHash: EmptyCodeHash.Bytes(),
    }
}
```

A new account starts with zero balance, the empty root hash, and the empty code hash. An account is considered "empty" (eligible for deletion under EIP-161) when all three of nonce, balance, and code hash equal their zero/empty values.

For storage in the snapshot layer, geth uses a **slim RLP** encoding (`SlimAccount`) that replaces the empty root and empty code hash with nil bytes, saving space for the common case of simple EOAs.

---

## stateObject: The In-Memory Account

While `StateAccount` is the on-disk format, `stateObject` (in `core/state/state_object.go`) is the in-memory working copy. It wraps a `StateAccount` with caches, dirty tracking, and a reference back to its parent `StateDB`:

```go
// core/state/state_object.go

type stateObject struct {
    db       *StateDB
    address  common.Address
    addrHash common.Hash         // keccak256(address)
    origin   *types.StateAccount // original data before any changes, nil if new
    data     types.StateAccount  // current data with all mutations applied

    trie Trie   // storage trie, opened lazily on first access
    code []byte // contract bytecode, loaded on demand

    originStorage      Storage // storage values read from disk/trie
    dirtyStorage       Storage // storage modified in the current transaction
    pendingStorage     Storage // storage modified in the current block (across txs)
    uncommittedStorage Storage // storage modified since last commit, with original values

    dirtyCode      bool // true if code was updated
    selfDestructed bool // true if account was self-destructed
    newContract    bool // true if created in current tx (EIP-6780)
}
```

The `Storage` type is simply `map[common.Hash]common.Hash`.

The four storage maps form a layered cache:

| Map | Scope | Purpose |
|-----|-------|---------|
| `originStorage` | block | Values as read from disk. The "clean" baseline for comparison. |
| `dirtyStorage` | transaction | Values modified in the current transaction. Cleared after each `Finalise()`. |
| `pendingStorage` | block | Accumulated modifications across all transactions in the block. Used for trie updates. |
| `uncommittedStorage` | since last commit | Tracks which slots changed since the last trie commit, along with their original values. |

This layering lets geth handle mid-transaction reverts (clear `dirtyStorage` entries) without losing cross-transaction state (`pendingStorage`).

---

## StateDB: The Main API

`StateDB` (in `core/state/statedb.go`) is the central interface that the EVM and all state-touching code use. It manages a collection of `stateObject`s and provides the public API for reading and writing accounts:

```go
// core/state/statedb.go

type StateDB struct {
    db         Database
    prefetcher *triePrefetcher
    reader     Reader
    trie       Trie              // the account trie, resolved on first access

    originalRoot common.Hash     // state root before any changes

    stateObjects         map[common.Address]*stateObject // live account objects
    stateObjectsDestruct map[common.Address]*stateObject // accounts deleted in this block
    mutations            map[common.Address]*mutation     // pending mutations per account

    dbErr  error   // first database error encountered
    refund uint64  // gas refund counter

    // Per-transaction state
    thash   common.Hash
    txIndex int
    logs    map[common.Hash][]*types.Log
    logSize uint

    // Per-block state
    preimages        map[common.Hash][]byte
    accessList       *accessList
    transientStorage transientStorage

    // Undo log
    journal *journal
    // ...
}
```

Key fields:

- **`db`** — the `Database` interface that provides access to tries and the snapshot layer. It bridges StateDB to the trie database from Chapter 03 and the storage stack from Chapter 05.
- **`reader`** — a `Reader` interface with `Account(addr)` and `Storage(addr, slot)` methods for loading state from disk.
- **`stateObjects`** — the live cache of all accounts accessed during this block. Once an account is loaded, it stays here for the duration of the block.
- **`stateObjectsDestruct`** — accounts that were self-destructed or deleted (EIP-161 empty accounts). Stored separately so that storage lookups for destructed accounts return empty.
- **`mutations`** — tracks which accounts have been modified and whether they were updated or deleted. Used during commit to know what to flush.
- **`journal`** — the undo log that enables `Snapshot()` / `RevertToSnapshot()`.

### Creating a StateDB

```go
// core/state/statedb.go

func New(root common.Hash, db Database) (*StateDB, error) {
    reader, err := db.Reader(root)
    if err != nil {
        return nil, err
    }
    return NewWithReader(root, db, reader)
}
```

`New` takes a state root (typically the parent block's state root) and a `Database`. It creates a `Reader` bound to that root, which will be used for all subsequent state lookups. The account trie itself is not opened yet — it's resolved lazily on first write.

---

## Reading State

All read operations follow the same pattern: look up the `stateObject` for the address, then read the field. For example:

```go
// core/state/statedb.go

func (s *StateDB) GetBalance(addr common.Address) *uint256.Int {
    stateObject := s.getStateObject(addr)
    if stateObject != nil {
        return stateObject.Balance()
    }
    return common.U2560
}
```

The interesting work is in `getStateObject`, which implements a multi-layer lookup:

```go
// core/state/statedb.go

func (s *StateDB) getStateObject(addr common.Address) *stateObject {
    // 1. Check the in-memory cache first
    if obj := s.stateObjects[addr]; obj != nil {
        return obj
    }
    // 2. If destructed in this block, return nil
    if _, ok := s.stateObjectsDestruct[addr]; ok {
        return nil
    }
    // 3. Load from the reader (snapshot or trie)
    acct, err := s.reader.Account(addr)
    if err != nil {
        s.setError(...)
        return nil
    }
    if acct == nil {
        return nil
    }
    // 4. Wrap in stateObject and cache
    obj := newObject(s, addr, acct)
    s.setStateObject(obj)
    return obj
}
```

The lookup chain:

1. **In-memory cache** (`stateObjects` map) — O(1) hash map lookup. Once loaded, accounts stay cached for the entire block.
2. **Destruction check** — if the account was deleted in this block, return nil immediately. This prevents reading stale disk data for a destroyed account.
3. **Reader** — calls `s.reader.Account(addr)`, which reads from the snapshot layer (if available) or falls back to the trie. The `Reader` interface abstracts this.
4. **Cache and return** — the loaded account is wrapped in a `stateObject` and inserted into the cache.

### Reading Storage

Storage reads follow a similar layered pattern inside `stateObject`:

```go
// core/state/state_object.go

func (s *stateObject) GetCommittedState(key common.Hash) common.Hash {
    if value, pending := s.pendingStorage[key]; pending {
        return value
    }
    if value, cached := s.originStorage[key]; cached {
        return value
    }
    if _, destructed := s.db.stateObjectsDestruct[s.address]; destructed {
        s.originStorage[key] = common.Hash{}
        return common.Hash{}
    }
    // Load from reader (snapshot/trie)
    value, err := s.db.reader.Storage(s.address, key)
    // ...
    s.originStorage[key] = value
    return value
}

func (s *stateObject) getState(key common.Hash) (common.Hash, common.Hash) {
    origin := s.GetCommittedState(key)
    value, dirty := s.dirtyStorage[key]
    if dirty {
        return value, origin
    }
    return origin, origin
}
```

`getState` returns two values: the **current** value and the **committed** (pre-transaction) value. Both are always needed — the committed value is used for gas metering (EIP-2200). So `GetCommittedState` always runs, even when a dirty value exists.

The committed value is resolved through these layers:

1. **`pendingStorage`** — values written in earlier transactions within this block.
2. **`originStorage`** — values previously loaded from disk (a read cache).
3. **`reader.Storage()`** — loads from the snapshot layer or trie on disk.

Then `getState` checks **`dirtyStorage`** — if a value was written in the current transaction, it overrides the committed value as the "current" return. Otherwise, the committed value is returned for both.

Values loaded from disk are cached in `originStorage` for future reads.

---

## Writing State

Write operations also go through `StateDB`, which delegates to `stateObject`:

```go
// core/state/statedb.go

func (s *StateDB) AddBalance(addr common.Address, amount *uint256.Int, reason tracing.BalanceChangeReason) uint256.Int {
    stateObject := s.getOrNewStateObject(addr)
    if stateObject == nil {
        return uint256.Int{}
    }
    return stateObject.AddBalance(amount)
}

func (s *StateDB) SetState(addr common.Address, key, value common.Hash) common.Hash {
    if stateObject := s.getOrNewStateObject(addr); stateObject != nil {
        return stateObject.SetState(key, value)
    }
    return common.Hash{}
}
```

`getOrNewStateObject` loads an existing account or creates a new empty one. The actual mutation happens inside `stateObject`:

```go
// core/state/state_object.go

func (s *stateObject) SetBalance(amount *uint256.Int) uint256.Int {
    prev := *s.data.Balance
    s.db.journal.balanceChange(s.address, s.data.Balance)
    s.setBalance(amount)
    return prev
}

func (s *stateObject) SetState(key, value common.Hash) common.Hash {
    prev, origin := s.getState(key)
    if prev == value {
        return prev
    }
    s.db.journal.storageChange(s.address, key, prev, origin)
    s.setState(key, value, origin)
    return prev
}
```

Every write follows the same two-step pattern:

1. **Journal the change** — record the previous value in the journal so it can be undone on revert.
2. **Apply the mutation** — update the in-memory field (`data.Balance`) or dirty map (`dirtyStorage`).

The `setState` helper has a subtle optimization: if the new value equals the original (pre-transaction) value, the key is removed from `dirtyStorage` entirely. This means "set back to original" is a no-op from the trie's perspective.

```go
// core/state/state_object.go

func (s *stateObject) setState(key common.Hash, value common.Hash, origin common.Hash) {
    if value == origin {
        delete(s.dirtyStorage, key)
        return
    }
    s.dirtyStorage[key] = value
}
```

---

## The Journal: Snapshot and Revert

The EVM needs to undo state changes when a transaction reverts (out of gas, `REVERT` opcode, etc.). Geth handles this with a journal — an append-only log of undo entries.

The journal is defined in `core/state/journal.go`:

```go
// core/state/journal.go

type journalEntry interface {
    revert(*StateDB)
    dirtied() *common.Address
    copy() journalEntry
}

type journal struct {
    entries []journalEntry         // list of undo entries
    dirties map[common.Address]int // dirty accounts and their change count

    validRevisions []revision
    nextRevisionId int
}

type revision struct {
    id           int
    journalIndex int
}
```

Each state mutation (balance change, storage write, nonce update, etc.) appends a `journalEntry` that knows how to undo itself. The concrete entry types are defined in the same file:

```go
// core/state/journal.go

type balanceChange struct {
    account common.Address
    prev    *uint256.Int
}

type storageChange struct {
    account   common.Address
    key       common.Hash
    prevvalue common.Hash
    origvalue common.Hash
}

type nonceChange struct {
    account common.Address
    prev    uint64
}
// ... plus codeChange, createObjectChange, selfDestructChange, etc.
```

Each entry stores just enough data to undo the change — typically the previous value.

### Snapshot and RevertToSnapshot

The EVM takes a snapshot before each internal call. If the call fails, it reverts to the snapshot:

```go
// core/state/statedb.go

func (s *StateDB) Snapshot() int {
    return s.journal.snapshot()
}

func (s *StateDB) RevertToSnapshot(revid int) {
    s.journal.revertToSnapshot(revid, s)
}
```

`snapshot()` records the current journal length and returns an ID. `revertToSnapshot()` replays all journal entries from the current end back to the recorded length, calling `revert()` on each:

```go
// core/state/journal.go

func (j *journal) revert(statedb *StateDB, snapshot int) {
    for i := len(j.entries) - 1; i >= snapshot; i-- {
        j.entries[i].revert(statedb)

        if addr := j.entries[i].dirtied(); addr != nil {
            if j.dirties[*addr]--; j.dirties[*addr] == 0 {
                delete(j.dirties, *addr)
            }
        }
    }
    j.entries = j.entries[:snapshot]
}
```

The reversal walks backward through the journal entries, undoing each change. The `dirties` map is also adjusted — if an account's change count drops to zero, it's removed from the dirty set entirely.

---

## Finalise and IntermediateRoot

After each transaction, `Finalise()` promotes dirty storage to pending and cleans up:

```go
// core/state/statedb.go

func (s *StateDB) Finalise(deleteEmptyObjects bool) {
    for addr := range s.journal.dirties {
        obj, exist := s.stateObjects[addr]
        if !exist {
            continue
        }
        if obj.selfDestructed || (deleteEmptyObjects && obj.empty()) {
            delete(s.stateObjects, obj.address)
            s.markDelete(addr)
            if _, ok := s.stateObjectsDestruct[obj.address]; !ok {
                s.stateObjectsDestruct[obj.address] = obj
            }
        } else {
            obj.finalise()
            s.markUpdate(addr)
        }
        // ...
    }
    // ...
}
```

For each account that was dirtied during the transaction:

- If self-destructed or empty (EIP-161): delete it from the live set and record it in `stateObjectsDestruct`.
- Otherwise: call `obj.finalise()`, which moves `dirtyStorage` entries into `pendingStorage` and clears the dirty map.

`IntermediateRoot()` goes one step further — it flushes pending storage into the actual tries and updates the account trie:

```go
// core/state/statedb.go (simplified)

func (s *StateDB) IntermediateRoot(deleteEmptyObjects bool) common.Hash {
    s.Finalise(deleteEmptyObjects)

    // Open the account trie if not yet loaded
    if s.trie == nil {
        tr, err := s.db.OpenTrie(s.originalRoot)
        // ...
        s.trie = tr
    }
    // Phase 1: Update each account's storage trie (concurrently)
    for addr, op := range s.mutations {
        if op.applied || op.isDelete() {
            continue
        }
        obj := s.stateObjects[addr]
        workers.Go(func() error {
            obj.updateRoot()
            return nil
        })
    }
    workers.Wait()

    // Phase 2: Write account data into the account trie
    for addr, op := range s.mutations {
        // ...
        if op.isDelete() {
            deletedAddrs = append(deletedAddrs, addr)
        } else {
            s.updateStateObject(s.stateObjects[addr])
        }
    }
    for _, deletedAddr := range deletedAddrs {
        s.deleteStateObject(deletedAddr)
    }
    return s.trie.Hash()
}
```

The function has two phases:

1. **Storage tries** — `updateRoot()` on each stateObject flushes `uncommittedStorage` into the storage trie via `Trie.UpdateStorage()` / `Trie.DeleteStorage()`, then calls `trie.Hash()` to compute the new storage root. This runs concurrently for all mutated accounts.
2. **Account trie** — after all storage roots are updated, each mutated account's data (including the new `Root`) is written into the account trie via `updateStateObject()`. Deleted accounts are removed via `deleteStateObject()`. Updates are applied before deletions to avoid unnecessary trie node resolution.

The result is a new state root hash without committing to disk — this is used to set the block header's `Root` field during block processing.

---

## Commit: Flushing to Disk

`Commit()` writes all state changes to the database. It's called once per block after all transactions are processed:

```go
// core/state/statedb.go

func (s *StateDB) Commit(block uint64, deleteEmptyObjects bool, noStorageWiping bool) (common.Hash, error) {
    ret, err := s.commitAndFlush(block, deleteEmptyObjects, noStorageWiping)
    // ...
    return ret.root, nil
}
```

The inner `commit()` method orchestrates the work:

1. **`IntermediateRoot()`** — finalise all pending changes and flush into tries.
2. **Handle destructions** — process account deletions first (storage trie cleanup).
3. **Commit account trie** — `s.trie.Commit(true)` is scheduled first since the account trie is the largest.
4. **Commit storage tries** — for each mutated account, `obj.commit()` commits the storage trie and returns a `NodeSet` of dirty trie nodes. These run concurrently with each other and with the account trie commit via `errgroup`.
5. **Merge all NodeSets** — all dirty trie nodes (account + storage) are merged into a single `MergedNodeSet`.

The `commitAndFlush()` wrapper then persists the results:

```go
// core/state/statedb.go (commitAndFlush, simplified)

func (s *StateDB) commitAndFlush(block uint64, ...) (*stateUpdate, error) {
    ret, err := s.commit(deleteEmptyObjects, noStorageWiping, block)
    // ...
    // Write contract code to disk
    if len(ret.codes) > 0 {
        batch := db.NewBatch()
        for _, code := range ret.codes {
            rawdb.WriteCode(batch, code.hash, code.blob)
        }
        batch.Write()
    }
    // Update snapshot tree
    if snap := s.db.Snapshot(); snap != nil {
        snap.Update(ret.root, ret.originRoot, ret.accounts, ret.storages)
        snap.Cap(ret.root, TriesInMemory)
    }
    // Write trie nodes to the trie database
    if db := s.db.TrieDB(); db != nil {
        db.Update(ret.root, ret.originRoot, block, ret.nodes, ret.stateSet())
    }
    return ret, nil
}
```

Three things are written:

1. **Contract code** — new or modified bytecode is written via `rawdb.WriteCode()` in a batch.
2. **Snapshot layer** — the flat state snapshot is updated with the account and storage diffs. `Cap()` keeps at most `TriesInMemory` (128) diff layers in memory.
3. **Trie database** — all dirty trie nodes are submitted to `triedb.Database.Update()`, which either adds them to the hashdb cache or creates a new pathdb diff layer (see Chapter 03).

After commit, the `StateDB` is effectively consumed — its tries are committed and no longer usable. A new `StateDB` must be created from the new root for subsequent blocks.

---

## The Read Path: Reader Interface

The `Reader` interface abstracts how state is loaded from disk. It combines two sub-interfaces:

```go
// core/state/reader.go

type StateReader interface {
    Account(addr common.Address) (*types.StateAccount, error)
    Storage(addr common.Address, slot common.Hash) (common.Hash, error)
}

type ContractCodeReader interface {
    Code(addr common.Address, codeHash common.Hash) ([]byte, error)
    CodeSize(addr common.Address, codeHash common.Hash) (int, error)
}

type Reader interface {
    ContractCodeReader
    StateReader
}
```

The `Reader` is created by `db.Reader(root)` when `StateDB` is constructed. Depending on the configuration, the implementation may read from the snapshot layer first (O(1) flat lookups) and fall back to the trie (O(log n) path traversal) only for misses. This is why snapshot lookups appear in the read path before trie lookups.

---

## What's Next

With accounts and state covered, we now understand how geth manages the in-memory representation of Ethereum's world state. Chapter 05 — The Storage Stack completes the picture by tracing the full path from `StateDB` through the trie database down to bytes on disk.
