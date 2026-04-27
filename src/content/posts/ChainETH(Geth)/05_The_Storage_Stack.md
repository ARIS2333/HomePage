---
title: Geth(5) The Storage Stack
published: 2026-04-14
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

The previous chapters introduced the **Merkle Patricia Trie** (how data is authenticated) and the **Account & State layer** (how state is organized and mutated). But neither chapter answered a practical question: *where do all these bytes actually end up?*

This chapter traces the full path from an in-memory state mutation down to bytes on disk. It covers four things:

1. **The interface hierarchy** — how geth defines a storage contract that every backend must implement
2. **The key-value store** — how Pebble (the default engine) turns `Put`/`Get` calls into disk I/O
3. **The key schema and accessor layer** — how `core/rawdb/` organizes all of Ethereum's data into a single flat key space
4. **The freezer** — how ancient, finalized blocks are moved out of the key-value store into append-only flat files

---

## The Four-Layer Diagram

When `StateDB.Commit()` finishes (covered in Chapter 04), the trie nodes and account data need to reach disk. They travel through four layers:

```
+-----------------------------------------------------------+
|  Layer 4: StateDB                                         |
|  In-memory dirty state, journal, snapshots                |
|  (core/state/)                                            |
+---------------------------+-------------------------------+
                            |  Commit()
                            v
+-----------------------------------------------------------+
|  Layer 3: Trie + TrieDB                                   |
|  Merkle Patricia Trie nodes, path-based or hash-based     |
|  persistence (trie/, triedb/)                              |
+---------------------------+-------------------------------+
                            |  triedb.Commit() → batch writes
                            v
+-----------------------------------------------------------+
|  Layer 2: rawdb accessor layer                            |
|  Key-prefix schema, Read/Write functions                  |
|  (core/rawdb/)                                            |
+---------------------------+-------------------------------+
                            |  ethdb.Put(), ethdb.Batch.Write()
                            v
+-----------------------------------------------------------+
|  Layer 1: Key-Value Store + Freezer                       |
|  Pebble (default) or LevelDB for live data                |
|  Freezer for ancient chain segments                       |
|  (ethdb/pebble/, core/rawdb/freezer.go)                   |
+-----------------------------------------------------------+
```

Layers 3 and 4 were covered in previous chapters. This chapter focuses on **Layers 1 and 2** — the bottom half of the stack.

---

## The Interface Hierarchy

Geth defines all storage contracts in a single file: `ethdb/database.go`. Everything above this boundary — trie code, rawdb accessors, the Ethereum service — programs against these interfaces. The actual storage engine (Pebble, LevelDB, or an in-memory map) is invisible to them.

### The Key-Value Side

The core building blocks are two tiny interfaces — one for reading, one for writing:

```go
// ethdb/database.go

type KeyValueReader interface {
    Has(key []byte) (bool, error)
    Get(key []byte) ([]byte, error)
}

type KeyValueWriter interface {
    Put(key []byte, value []byte) error
    Delete(key []byte) error
}
```

These are combined into `KeyValueStore`, which adds batch support, iteration, compaction, and statistics:

```go
// ethdb/database.go

type KeyValueStore interface {
    KeyValueReader
    KeyValueWriter
    KeyValueStater
    KeyValueSyncer
    KeyValueRangeDeleter
    Batcher
    Iteratee
    Compacter
    io.Closer
}
```

- **`Batcher`** provides `NewBatch()` for atomic multi-key writes (covered below).
- **`Iteratee`** provides `NewIterator(prefix, start)` for ordered key scans.
- **`Compacter`** provides `Compact(start, limit)` for triggering LSM-tree compaction.
- **`KeyValueSyncer`** provides `SyncKeyValue()` to force-flush the write-ahead log.

### The Ancient Side

Old, finalized blocks rarely change and are better stored in flat, append-only files. Geth calls this the "ancient store" and defines a separate interface family for it:

```go
// ethdb/database.go

type AncientReaderOp interface {
    Ancient(kind string, number uint64) ([]byte, error)
    AncientRange(kind string, start, count, maxBytes uint64) ([][]byte, error)
    Ancients() (uint64, error)
    Tail() (uint64, error)
    // ...
}

type AncientWriter interface {
    ModifyAncients(func(AncientWriteOp) error) (int64, error)
    TruncateHead(n uint64) (uint64, error)
    TruncateTail(n uint64) (uint64, error)
    SyncAncient() error
}
```

- **`Ancient(kind, number)`** retrieves a single item (e.g., `Ancient("headers", 42)` returns block 42's header).
- **`ModifyAncients(fn)`** is the write API. The callback receives an `AncientWriteOp` with `Append`/`AppendRaw` methods. If the callback returns an error, all changes are rolled back.
- **`Tail()`** returns the first available item number — items before this have been pruned.

### The Unified Database

At the top, a single interface combines both worlds:

```go
// ethdb/database.go

type Database interface {
    KeyValueStore
    AncientStore
}
```

Every component in geth that needs storage receives an `ethdb.Database`. Internally it is a `freezerdb` — a struct that embeds a `KeyValueStore` (Pebble) and a `chainFreezer` (flat files):

```go
// core/rawdb/database.go

type freezerdb struct {
    ethdb.KeyValueStore
    *chainFreezer

    readOnly    bool
    ancientRoot string
}
```

The `rawdb.Open()` function constructs this combination, validates that the key-value store and freezer are consistent (matching genesis hashes, no gaps in block numbers), and starts a background goroutine that periodically freezes finalized blocks.

---

## The Key-Value Store: Pebble

Pebble is geth's default storage engine (replacing LevelDB). It is an LSM-tree key-value store from CockroachDB that provides the `ethdb.KeyValueStore` interface.

### Configuration

The `pebble.New()` constructor in `ethdb/pebble/pebble.go` sets up the engine with these key parameters:

```go
// ethdb/pebble/pebble.go (inside New)

opt := &pebble.Options{
    Cache:        pebble.NewCache(int64(cache * 1024 * 1024)),
    MaxOpenFiles: handles,
    MemTableSize: uint64(memTableSize),
    MemTableStopWritesThreshold: memTableLimit,        // 4
    MaxConcurrentCompactions:    runtime.NumCPU,
    Levels: []pebble.LevelOptions{
        {TargetFileSize: 2 * 1024 * 1024, FilterPolicy: bloom.FilterPolicy(10)},
        {TargetFileSize: 4 * 1024 * 1024, FilterPolicy: bloom.FilterPolicy(10)},
        // ... 5 more levels, doubling each time up to 128 MB
    },
    L0CompactionThreshold: 2,
}
```

- **Cache** is split between read and write buffers. The total is set from geth's `--cache` flag.
- **4 memtables** allow smoother write flushing (smaller, more frequent flushes instead of large spikes).
- **Bloom filters** (10 bits per key) on every level accelerate point lookups by avoiding disk reads for keys that don't exist.
- **L0 compaction threshold = 2** is lower than Pebble's default of 4, reducing the compaction debt at the cost of more frequent compactions.

### Asynchronous Writes

By default, geth uses **async writes** — `Put` and `Batch.Write` return before the write-ahead log (WAL) is fsynced to disk:

```go
// ethdb/pebble/pebble.go

writeOptions: pebble.NoSync,
```

This gives much better write throughput, especially on macOS. Geth is designed to recover from unclean shutdowns, so losing a few recent writes is acceptable. For safety, periodic background fsyncs are triggered via `WALBytesPerSync`.

### Core Operations

The `Get` and `Put` methods are thin wrappers around Pebble's native API:

```go
// ethdb/pebble/pebble.go

func (d *Database) Get(key []byte) ([]byte, error) {
    d.quitLock.RLock()
    defer d.quitLock.RUnlock()
    if d.closed {
        return nil, pebble.ErrClosed
    }
    dat, closer, err := d.db.Get(key)
    if err != nil {
        return nil, err
    }
    ret := make([]byte, len(dat))
    copy(ret, dat)
    closer.Close()
    return ret, nil
}

func (d *Database) Put(key []byte, value []byte) error {
    // ... closed check ...
    return d.db.Set(key, value, d.writeOptions)
}
```

Note that `Get` copies the value into a new byte slice. Pebble's `Get` returns a pointer into an internal buffer with a `closer` — the data is only valid until `closer.Close()` is called. The copy ensures the caller owns the bytes.

---

## Batch Writes

Individual `Put` calls are expensive: each one goes through the WAL individually. When geth needs to write many keys atomically (e.g., inserting a block's worth of trie nodes), it uses a **batch**:

```go
// ethdb/batch.go

type Batch interface {
    KeyValueWriter        // Put, Delete
    KeyValueRangeDeleter  // DeleteRange

    ValueSize() int       // bytes queued so far
    Write() error         // flush all queued ops to disk atomically
    Reset()               // clear the batch for reuse
    Replay(w KeyValueWriter) error  // replay ops against another writer
}
```

A batch buffers Put/Delete operations in memory. Nothing touches the database until `Write()` is called, and the entire batch is applied atomically — either all writes succeed or none do.

The `IdealBatchSize` constant (100 KB) serves as a guideline: callers can check `batch.ValueSize() >= ethdb.IdealBatchSize` to decide when to flush and start a new batch. This prevents batches from growing too large in memory.

Here is how batches are used in practice. During chain freezing, old blocks are deleted from the key-value store in batches:

```go
// core/rawdb/chain_freezer.go (inside freeze)

batch := db.NewBatch()
for i := 0; i < len(ancients); i++ {
    if first+uint64(i) != 0 {
        DeleteBlockWithoutNumber(batch, ancients[i], first+uint64(i))
        DeleteCanonicalHash(batch, first+uint64(i))
    }
}
if err := batch.Write(); err != nil {
    log.Crit("Failed to delete frozen canonical blocks", "err", err)
}
batch.Reset()
```

Under the hood in Pebble, `Write()` calls `pebble.Batch.Commit()`, which applies all buffered operations to the database in a single atomic write.

---

## The Key Schema

Geth stores everything — headers, bodies, receipts, trie nodes, contract code, snapshots, transaction indices — in a single flat key-value namespace. The `core/rawdb/schema.go` file defines the **key-prefix schema** that organizes this namespace.

### Singleton Keys

Some values are global, storing a single piece of state:

```go
// core/rawdb/schema.go

headHeaderKey         = []byte("LastHeader")
headBlockKey          = []byte("LastBlock")
headFinalizedBlockKey = []byte("LastFinalized")
persistentStateIDKey  = []byte("LastStateID")
trieJournalKey        = []byte("TrieJournal")
SnapshotRootKey       = []byte("SnapshotRoot")
```

These are fixed-length keys that map to a single value (typically a 32-byte hash or an 8-byte block number).

### Prefix-Based Keys

Most data is keyed by combining a single-byte prefix with a block number (big-endian uint64) and/or a hash (32 bytes). Single-byte prefixes keep keys short and ensure different data types never collide:

| Prefix | Key Format | Value |
|--------|-----------|-------|
| `"h"` | `h` + num(8) + hash(32) | Block header (RLP) |
| `"h"` + `"n"` | `h` + num(8) + `n` | Canonical hash for block number |
| `"H"` | `H` + hash(32) | Block number for hash |
| `"b"` | `b` + num(8) + hash(32) | Block body (RLP) |
| `"r"` | `r` + num(8) + hash(32) | Block receipts (RLP) |
| `"l"` | `l` + txHash(32) | Transaction lookup metadata |
| `"c"` | `c` + codeHash(32) | Contract bytecode |
| `"a"` | `a` + accountHash(32) | Snapshot: account data |
| `"o"` | `o` + accountHash(32) + storageHash(32) | Snapshot: storage slot |
| `"A"` | `A` + hexPath | Trie node (path-based, account trie) |
| `"O"` | `O` + accountHash(32) + hexPath | Trie node (path-based, storage trie) |
| `"L"` | `L` + stateRoot(32) | State ID (path-based) |

The key-building functions are also defined in `schema.go`:

```go
// core/rawdb/schema.go

func headerKey(number uint64, hash common.Hash) []byte {
    return append(append(headerPrefix, encodeBlockNumber(number)...), hash.Bytes()...)
}

func blockBodyKey(number uint64, hash common.Hash) []byte {
    return append(append(blockBodyPrefix, encodeBlockNumber(number)...), hash.Bytes()...)
}

func codeKey(hash common.Hash) []byte {
    return append(CodePrefix, hash.Bytes()...)
}
```

Block numbers are always encoded as 8-byte big-endian integers. This ensures that keys sort in block-number order within each prefix, which makes range scans efficient.

---

## The Accessor Layer

The `core/rawdb/` package provides **accessor functions** — typed Read/Write/Delete helpers that handle key construction, RLP encoding/decoding, and the ancient-vs-live lookup logic. Higher layers never construct raw keys or call `db.Get()` directly.

### Chain Data Accessors

The pattern is consistent across all chain data. Here is `ReadHeader`:

```go
// core/rawdb/accessors_chain.go

func ReadHeader(db ethdb.Reader, hash common.Hash, number uint64) *types.Header {
    data := ReadHeaderRLP(db, hash, number)
    if len(data) == 0 {
        return nil
    }
    header := new(types.Header)
    if err := rlp.DecodeBytes(data, header); err != nil {
        log.Error("Invalid block header RLP", "hash", hash, "err", err)
        return nil
    }
    return header
}
```

It delegates to `ReadHeaderRLP`, which handles the two-tier lookup — check the freezer first, fall back to the key-value store:

```go
// core/rawdb/accessors_chain.go

func ReadHeaderRLP(db ethdb.Reader, hash common.Hash, number uint64) rlp.RawValue {
    var data []byte
    db.ReadAncients(func(reader ethdb.AncientReaderOp) error {
        data, _ = reader.Ancient(ChainFreezerHeaderTable, number)
        if len(data) > 0 && crypto.Keccak256Hash(data) == hash {
            return nil
        }
        data, _ = db.Get(headerKey(number, hash))
        return nil
    })
    return data
}
```

- First, try `reader.Ancient("headers", number)` — the freezer is indexed by block number alone.
- If found, verify the hash matches (the freezer only stores canonical data — the requested hash might be a fork block).
- If not found (or hash mismatch), fall back to `db.Get(headerKey(number, hash))` — the key-value store, which stores both canonical and non-canonical blocks.

The `ReadAncients` wrapper ensures the entire callback runs under the freezer's read lock, so no concurrent writes can change the data mid-read.

The write side is simpler — it always targets the key-value store (data is only moved to the freezer later by the background freezer goroutine):

```go
// core/rawdb/accessors_chain.go

func WriteHeader(db ethdb.KeyValueWriter, header *types.Header) {
    var (
        hash   = header.Hash()
        number = header.Number.Uint64()
    )
    WriteHeaderNumber(db, hash, number)

    data, err := rlp.EncodeToBytes(header)
    if err != nil {
        log.Crit("Failed to RLP encode header", "err", err)
    }
    key := headerKey(number, hash)
    if err := db.Put(key, data); err != nil {
        log.Crit("Failed to store header", "err", err)
    }
}
```

`WriteHeader` does two things: stores the hash→number mapping (for reverse lookups) and stores the RLP-encoded header at `h + number + hash`.

### State Data Accessors

The `core/rawdb/accessors_state.go` file provides accessors for state-related data — contract code, preimages, state IDs, and trie journals:

```go
// core/rawdb/accessors_state.go

func ReadCode(db ethdb.KeyValueReader, hash common.Hash) []byte {
    data := ReadCodeWithPrefix(db, hash)
    if len(data) != 0 {
        return data
    }
    data, _ = db.Get(hash.Bytes())
    return data
}

func WriteCode(db ethdb.KeyValueWriter, hash common.Hash, code []byte) {
    if err := db.Put(codeKey(hash), code); err != nil {
        log.Crit("Failed to store contract code", "err", err)
    }
}
```

`ReadCode` tries the current prefixed scheme (`"c" + codeHash`) first, then falls back to a legacy scheme (bare `codeHash` as key) for backward compatibility.

State IDs map state roots to sequential numbers, used by the path-based trie database (see Chapter 03):

```go
// core/rawdb/accessors_state.go

func ReadStateID(db ethdb.KeyValueReader, root common.Hash) *uint64 {
    data, err := db.Get(stateIDKey(root))
    if err != nil || len(data) == 0 {
        return nil
    }
    number := binary.BigEndian.Uint64(data)
    return &number
}
```

---

## The Freezer: Ancient Storage

The key-value store (Pebble) is optimized for random reads and writes, but it pays a cost: LSM-tree compaction continuously rewrites data on disk. For historical chain data that is never modified after finalization, this overhead is wasteful. The **freezer** solves this by moving finalized blocks out of Pebble into append-only flat files.

### How the Freezer Works

The freezer stores data in **tables** — each table holds one type of data. The chain freezer has four tables:

| Table | Data | Prunable |
|-------|------|----------|
| `"headers"` | RLP-encoded block headers | No |
| `"hashes"` | Canonical block hashes (32 bytes each) | No |
| `"bodies"` | RLP-encoded block bodies | Yes |
| `"receipts"` | RLP-encoded receipts | Yes |

Headers and hashes are kept forever (not prunable). Bodies and receipts can be pruned via `TruncateTail` — once pruned, they are no longer accessible from the freezer (though an optional Era database can serve as a backup).

Each table is stored as a pair of files on disk:

```go
// core/rawdb/freezer_table.go

type freezerTable struct {
    items      atomic.Uint64   // total items stored (including removed from tail)
    itemOffset atomic.Uint64   // items removed from the table
    itemHidden atomic.Uint64   // items marked deleted but not yet physically removed
    // ...
    head   *os.File            // current data file being written to
    index  *os.File            // index file: maps item number → (filenum, offset)
    files  map[uint32]*os.File // all open data files
    // ...
}
```

- The **index file** contains fixed-size 6-byte entries (`uint16` file number + `uint32` offset). To find item N, read 6 bytes at position N×6 in the index file.
- The **data files** contain the actual blobs, optionally Snappy-compressed. Data files are capped at 2 GB each (`freezerTableSize`).

This design makes reads O(1): seek to the index entry, read the file number and offset, seek to that position in the data file, read the blob.

### The Freezer Struct

The `Freezer` struct ties the tables together:

```go
// core/rawdb/freezer.go

type Freezer struct {
    datadir string
    frozen  atomic.Uint64          // number of items frozen
    tail    atomic.Uint64          // first stored item
    // ...
    tables       map[string]*freezerTable
    instanceLock *flock.Flock      // prevents double-open
}
```

`frozen` tracks how many items have been written. `tail` tracks how many have been pruned from the start. Valid items are in the range `[tail, frozen)`.

### Writing to the Freezer

All writes go through `ModifyAncients`, which provides transactional semantics:

```go
// core/rawdb/freezer.go

func (f *Freezer) ModifyAncients(fn func(ethdb.AncientWriteOp) error) (writeSize int64, err error) {
    f.writeLock.Lock()
    defer f.writeLock.Unlock()

    prevItem := f.frozen.Load()
    defer func() {
        if err != nil {
            for name, table := range f.tables {
                err := table.truncateHead(prevItem)
                // ...
            }
        }
    }()

    f.writeBatch.reset()
    if err := fn(f.writeBatch); err != nil {
        return 0, err
    }
    item, writeSize, err := f.writeBatch.commit()
    if err != nil {
        return 0, err
    }
    f.frozen.Store(item)
    return writeSize, nil
}
```

- The write lock is held for the entire operation.
- If the callback or the commit fails, the `defer` rolls back all tables to their previous item count.
- On success, `f.frozen` advances to the new item count.

### The Chain Freezer: Background Migration

The `chainFreezer` in `core/rawdb/chain_freezer.go` wraps the base `Freezer` and adds a background goroutine that periodically moves finalized blocks from the key-value store into the freezer:

```go
// core/rawdb/chain_freezer.go (inside freeze, simplified)

threshold, _ := f.freezeThreshold(nfdb)
frozen, _ := f.Ancients()

if frozen-1 >= threshold {
    return  // nothing to freeze
}

// Phase 1: Copy blocks to the freezer
ancients, _ := f.freezeRange(nfdb, first, last)

// Phase 2: Sync freezer files to disk
f.SyncAncient()

// Phase 3: Delete the frozen blocks from the key-value store
batch := db.NewBatch()
for i := 0; i < len(ancients); i++ {
    if first+uint64(i) != 0 {
        DeleteBlockWithoutNumber(batch, ancients[i], first+uint64(i))
        DeleteCanonicalHash(batch, first+uint64(i))
    }
}
batch.Write()
```

The freeze threshold is `max(finalized_block, head - 90000)` — the higher of the finalized block number and 90,000 blocks behind the head (about 12.5 days of blocks).

The `freezeRange` method copies each block's hash, header, body, and receipts into the freezer:

```go
// core/rawdb/chain_freezer.go

func (f *chainFreezer) freezeRange(nfdb *nofreezedb, number, limit uint64) (hashes []common.Hash, err error) {
    _, err = f.ModifyAncients(func(op ethdb.AncientWriteOp) error {
        for ; number <= limit; number++ {
            hash := ReadCanonicalHash(nfdb, number)
            header := ReadHeaderRLP(nfdb, hash, number)
            body := ReadBodyRLP(nfdb, hash, number)
            receipts := ReadReceiptsRLP(nfdb, hash, number)

            op.AppendRaw(ChainFreezerHashTable, number, hash[:])
            op.AppendRaw(ChainFreezerHeaderTable, number, header)
            op.AppendRaw(ChainFreezerBodiesTable, number, body)
            op.AppendRaw(ChainFreezerReceiptTable, number, receipts)
            hashes = append(hashes, hash)
        }
        return nil
    })
    return hashes, err
}
```

After copying, the data is fsynced to the freezer, then deleted from the key-value store in batches. The key-value store also deletes any side-chain blocks (non-canonical forks) for the same block numbers.

### State History Freezer

In addition to the chain freezer, geth has a **state history freezer** for the path-based trie database. This stores old account and storage values so nodes can serve historical state queries. The tables are:

| Table | Data |
|-------|------|
| `"history.meta"` | Metadata for each state transition |
| `"account.index"` | Index into account data |
| `"storage.index"` | Index into storage data |
| `"account.data"` | Concatenated account diffs |
| `"storage.data"` | Concatenated storage slot diffs |

All state history tables are prunable. The `accessors_state.go` file provides typed readers and writers (e.g., `ReadStateHistory`, `WriteStateHistory`) that work with these tables through the same `AncientReaderOp`/`AncientWriteOp` interfaces.

---

## Putting It All Together

Here is the complete write path for a new block, from top to bottom:

```
StateDB.Commit()
  ├─ Write contract code      → rawdb.WriteCode()       → db.Put("c" + codeHash, code)
  ├─ Commit trie nodes         → triedb batch writes     → db.Put("A" + path, node)
  └─ Update snapshot           → db.Put("a" + hash, account)

blockchain.writeBlockAndSetHead()
  ├─ rawdb.WriteHeader()       → db.Put("h" + num + hash, headerRLP)
  ├─ rawdb.WriteBody()         → db.Put("b" + num + hash, bodyRLP)
  ├─ rawdb.WriteReceipts()     → db.Put("r" + num + hash, receiptsRLP)
  ├─ rawdb.WriteTxLookupEntries() → db.Put("l" + txHash, blockNum)
  └─ rawdb.WriteHeadBlockHash() → db.Put("LastBlock", hash)

        ↓ (later, background goroutine)

chainFreezer.freeze()
  ├─ Copy h/b/r to freezer flat files
  ├─ fsync freezer
  └─ Delete h/b/r from Pebble via batch
```

All key-value writes go through Pebble with async WAL writes for speed. The freezer migrates finalized blocks to flat files in the background, keeping the key-value store lean. Reads check the freezer first (for old canonical data), then fall back to the key-value store.

---

## What's Next

With the storage stack complete, we've covered the full bottom-to-top path: from raw bytes on disk through the trie, up to the in-memory state. Chapter 06 — Transaction Execution shifts to the other axis of the system — tracing what happens when a single transaction moves through geth's execution pipeline.
