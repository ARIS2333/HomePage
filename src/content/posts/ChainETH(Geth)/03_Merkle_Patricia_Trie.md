---
title: Geth(3) Merkle Patricia Trie
published: 2026-04-12
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

Ethereum stores all account and storage data in a **Merkle Patricia Trie (MPT)** — a tree structure where every node is content-addressed by its Keccak256 hash. This means you can verify any piece of state by checking a short proof against the 32-byte state root stored in the block header.

You already know the concept. This chapter shows how geth implements it — from the node types and key encoding scheme, through trie operations and hashing, to the persistence layer that writes trie nodes to disk.

---

## How the Trie Fits into Geth

Before looking at any code, here is how the trie is used during the lifecycle of a block:

```
 1. StateDB receives state changes (SetState, AddBalance, ...)
       │
       ▼
 2. Changes accumulate in dirty maps inside stateObject
       │
       ▼
 3. At commit time, dirty values are written into per-account tries
    and the account trie via Trie.Update()
       │
       ▼
 4. Trie.Hash() walks the tree bottom-up, RLP-encodes each dirty node,
    and either inlines it (< 32 bytes) or Keccak256-hashes it
       │
       ▼
 5. Trie.Commit() collects all dirty nodes into a NodeSet
       │
       ▼
 6. triedb.Database persists the NodeSet to disk
    (pathdb or hashdb backend)
```

Steps 1–2 are covered in Chapter 04 — Account and State. This chapter focuses on steps 3–6: the trie itself, its hashing, its commit process, and its persistence.

---

## Key Encoding: Three Representations

Trie keys pass through three different encodings in geth. Understanding them is essential for reading any trie code.

The comment at the top of `trie/encoding.go` defines all three:

- **KEYBYTES** — the raw key bytes as the caller provides them (e.g., a 32-byte Keccak256 hash). This is the input to public API functions like `Trie.Get()` and `Trie.Update()`.

- **HEX** (nibbles) — each byte is split into two nibbles, so a 32-byte key becomes 64 one-byte entries (each in the range 0x00–0x0f). A trailing terminator byte `0x10` is appended for leaf keys. This is what the in-memory trie uses, because branching on individual nibbles is how the trie navigates its 16-way branch nodes.

- **COMPACT** (hex prefix encoding) — the Yellow Paper's encoding for keys stored on disk. It packs nibbles back into bytes and uses a flag byte to encode two pieces of information: whether the key length is odd, and whether the node is a leaf.

The conversion functions in `trie/encoding.go`:

```go
// trie/encoding.go

func keybytesToHex(str []byte) []byte {
    l := len(str)*2 + 1
    var nibbles = make([]byte, l)
    for i, b := range str {
        nibbles[i*2] = b / 16
        nibbles[i*2+1] = b % 16
    }
    nibbles[l-1] = 16
    return nibbles
}

func hexToCompact(hex []byte) []byte {
    terminator := byte(0)
    if hasTerm(hex) {
        terminator = 1
        hex = hex[:len(hex)-1]
    }
    buf := make([]byte, len(hex)/2+1)
    buf[0] = terminator << 5 // the flag byte
    if len(hex)&1 == 1 {
        buf[0] |= 1 << 4 // odd flag
        buf[0] |= hex[0] // first nibble is contained in the first byte
        hex = hex[1:]
    }
    decodeNibbles(hex, buf[1:])
    return buf
}
```

Walking through the encoding pipeline:

- **`keybytesToHex`** splits each byte into two nibbles (`b / 16`, `b % 16`) and appends terminator `16` (0x10). A 32-byte key becomes a 65-byte nibble array.
- **`hexToCompact`** does two things: it sets a flag byte (`buf[0]`) encoding the terminator status (bit 5) and odd-length flag (bit 4), then re-packs the remaining nibbles into bytes. If the nibble count is odd, the first nibble is squeezed into the lower half of the flag byte.

Here is a concrete example:

```
KEYBYTES:  [0xca, 0xfe]
HEX:       [0x0c, 0x0a, 0x0f, 0x0e, 0x10]   (nibbles + terminator)
COMPACT:   [0x20, 0xca, 0xfe]                 (flag 0x20 = leaf + even length)
```

The reverse function `compactToHex` converts disk-stored compact keys back to in-memory hex nibbles when loading nodes from the database.

---

## Node Types

Geth represents all trie nodes using a single `node` interface and four concrete types. These are defined in `trie/node.go`:

```go
// trie/node.go

type node interface {
    cache() (hashNode, bool)
    encode(w rlp.EncoderBuffer)
    fstring(string) string
}

type (
    fullNode struct {
        Children [17]node // Actual trie node data to encode/decode (needs custom encoder)
        flags    nodeFlag
    }
    shortNode struct {
        Key   []byte
        Val   node
        flags nodeFlag
    }
    hashNode  []byte
    valueNode []byte
)

type nodeFlag struct {
    hash  hashNode // cached hash of the node (may be nil)
    dirty bool     // whether the node has changes that must be written to the database
}
```

Each type maps to a role in the Merkle Patricia Trie:

| Type | RLP form | Role |
|------|----------|------|
| `fullNode` | 17-element list | **Branch node.** `Children[0]`–`Children[15]` hold one child per nibble (0–f). `Children[16]` holds a value if a key terminates at this branch. |
| `shortNode` | 2-element list | **Extension or leaf node.** If `Val` is another node (fullNode, shortNode, or hashNode), it's an extension — `Key` is a shared prefix. If `Val` is a `valueNode`, it's a leaf — `Key` is the remaining key suffix with terminator. |
| `hashNode` | 32-byte string | **Hash reference.** A placeholder for a node that hasn't been loaded from disk yet. Contains the Keccak256 hash of the node's RLP encoding. |
| `valueNode` | byte string | **Leaf value.** The actual data stored in the trie (e.g., an RLP-encoded account). |

The `nodeFlag` struct caches two pieces of metadata: the Keccak256 hash of the node (computed during hashing and reused until the node is modified) and a `dirty` flag that tracks whether the node has been modified since the last commit.

When nodes are decoded from RLP (loaded from disk), the `decodeNodeUnsafe` function in `trie/node.go` distinguishes between branch and short nodes by counting the list elements:

```go
// trie/node.go

func decodeNodeUnsafe(hash, buf []byte) (node, error) {
    // ...
    elems, _, err := rlp.SplitList(buf)
    // ...
    switch c, _ := rlp.CountValues(elems); c {
    case 2:
        n, err := decodeShort(hash, elems)
        return n, wrapError(err, "short")
    case 17:
        n, err := decodeFull(hash, elems)
        return n, wrapError(err, "full")
    default:
        return nil, fmt.Errorf("invalid number of list elements: %v", c)
    }
}
```

Two elements means a short node (extension or leaf). Seventeen elements means a branch node. The function `decodeShort` further distinguishes extension from leaf by checking whether the decoded key has a terminator (using `hasTerm`).

---

## The Trie Struct

The `Trie` struct is the main in-memory trie. It is defined in `trie/trie.go`:

```go
// trie/trie.go

type Trie struct {
    root  node
    owner common.Hash

    committed bool
    unhashed  int
    uncommitted int

    reader         *Reader
    opTracer       *opTracer
    prevalueTracer *PrevalueTracer
}
```

Key fields:

- **`root`** — the root node of the trie tree. All operations begin here.
- **`owner`** — identifies which trie this is. For the account trie, this is the zero hash. For a storage trie, this is the account's hashed address. The owner is used when reading/writing nodes to the database.
- **`committed`** — once `Commit()` is called, the trie is marked as committed and can no longer be used. A new trie must be created from the new root.
- **`unhashed`** / **`uncommitted`** — counters tracking modifications since the last hash/commit. When these exceed 100, the hasher and committer enable parallel processing.
- **`reader`** — the handler for loading nodes from the database when the trie encounters a `hashNode`.

Two constructors create tries:

```go
// trie/trie.go

func New(id *ID, db database.NodeDatabase) (*Trie, error) {
    reader, err := NewReader(id.StateRoot, id.Owner, db)
    // ...
    trie := &Trie{
        owner:          id.Owner,
        reader:         reader,
        opTracer:       newOpTracer(),
        prevalueTracer: NewPrevalueTracer(),
    }
    if id.Root != (common.Hash{}) && id.Root != types.EmptyRootHash {
        rootnode, err := trie.resolveAndTrack(id.Root[:], nil)
        // ...
        trie.root = rootnode
    }
    return trie, nil
}

func NewEmpty(db database.NodeDatabase) *Trie {
    tr, _ := New(TrieID(types.EmptyRootHash), db)
    return tr
}
```

`New` creates a trie rooted at a specific state. If the root isn't the empty root hash, it loads the root node from disk via `resolveAndTrack`. `NewEmpty` is a shorthand for an empty trie used mainly in tests.

---

## Trie Operations

### Get

`Get` looks up a key by walking the trie from root to leaf, consuming one nibble at each branch node and matching key prefixes at short nodes:

```go
// trie/trie.go

func (t *Trie) Get(key []byte) ([]byte, error) {
    // ...
    value, newroot, didResolve, err := t.get(t.root, keybytesToHex(key), 0)
    if err == nil && didResolve {
        t.root = newroot
    }
    return value, err
}

func (t *Trie) get(origNode node, key []byte, pos int) (value []byte, newnode node, didResolve bool, err error) {
    switch n := (origNode).(type) {
    case nil:
        return nil, nil, false, nil
    case valueNode:
        return n, n, false, nil
    case *shortNode:
        if !bytes.HasPrefix(key[pos:], n.Key) {
            return nil, n, false, nil
        }
        value, newnode, didResolve, err = t.get(n.Val, key, pos+len(n.Key))
        // ...
    case *fullNode:
        value, newnode, didResolve, err = t.get(n.Children[key[pos]], key, pos+1)
        // ...
    case hashNode:
        child, err := t.resolveAndTrack(n, key[:pos])
        // ...
        value, newnode, _, err := t.get(child, key, pos)
        return value, newnode, true, err
    // ...
    }
}
```

Walking through the recursive descent:

- **`nil`** — key not found, return nil.
- **`valueNode`** — reached a leaf value, return it.
- **`shortNode`** — check if the remaining key starts with `n.Key`. If yes, skip past the matched prefix and recurse into `n.Val`. If no, key not found.
- **`fullNode`** — consume one nibble (`key[pos]`) to select the child, recurse.
- **`hashNode`** — the node isn't loaded yet. Call `resolveAndTrack` to fetch it from disk, then retry with the resolved node. The `didResolve` flag propagates upward so the resolved node replaces the `hashNode` in-place (lazy loading).

### Update

`Update` inserts or deletes a key depending on the value length:

```go
// trie/trie.go

func (t *Trie) update(key, value []byte) error {
    t.unhashed++
    t.uncommitted++
    k := keybytesToHex(key)
    if len(value) != 0 {
        _, n, err := t.insert(t.root, nil, k, valueNode(value))
        // ...
        t.root = n
    } else {
        _, n, err := t.delete(t.root, nil, k)
        // ...
        t.root = n
    }
    return nil
}
```

A non-empty value triggers `insert`; an empty value triggers `delete`. Both return the new root node.

The `insert` method handles the most complex case — splitting a short node when two keys diverge:

```go
// trie/trie.go (insert, shortened)

func (t *Trie) insert(n node, prefix, key []byte, value node) (bool, node, error) {
    if len(key) == 0 {
        if v, ok := n.(valueNode); ok {
            return !bytes.Equal(v, value.(valueNode)), value, nil
        }
        return true, value, nil
    }
    switch n := n.(type) {
    case *shortNode:
        matchlen := prefixLen(key, n.Key)
        if matchlen == len(n.Key) {
            dirty, nn, err := t.insert(n.Val, append(prefix, key[:matchlen]...), key[matchlen:], value)
            // ...
            return true, &shortNode{n.Key, nn, t.newFlag()}, nil
        }
        // Otherwise branch out at the index where they differ.
        branch := &fullNode{flags: t.newFlag()}
        _, branch.Children[n.Key[matchlen]], _ = t.insert(nil, ..., n.Key[matchlen+1:], n.Val)
        _, branch.Children[key[matchlen]], _ = t.insert(nil, ..., key[matchlen+1:], value)
        if matchlen == 0 {
            return true, branch, nil
        }
        return true, &shortNode{key[:matchlen], branch, t.newFlag()}, nil

    case *fullNode:
        dirty, nn, err := t.insert(n.Children[key[0]], append(prefix, key[0]), key[1:], value)
        // ...
        n.flags = t.newFlag()
        n.Children[key[0]] = nn
        return true, n, nil

    case nil:
        return true, &shortNode{key, value, t.newFlag()}, nil

    case hashNode:
        rn, err := t.resolveAndTrack(n, prefix)
        // ...
        return t.insert(rn, prefix, key, value)
    }
}
```

The key logic for a `shortNode` with a key mismatch:

1. Find the common prefix length (`matchlen`).
2. If keys match entirely, recurse into the child.
3. If keys diverge, create a new `fullNode` (branch) with two children: the existing value at `n.Key[matchlen]` and the new value at `key[matchlen]`.
4. If there's a common prefix (`matchlen > 0`), wrap the branch in a new `shortNode` for that prefix.

All newly created nodes are marked dirty via `t.newFlag()`, which returns `nodeFlag{dirty: true}`.

### Delete

The `delete` method mirrors `insert` but also handles **node reduction** — after removing a child from a branch node, if only one child remains, the branch collapses back into a short node. This keeps the trie minimal:

```go
// trie/trie.go (delete, reduction logic)

case *fullNode:
    dirty, nn, err := t.delete(n.Children[key[0]], append(prefix, key[0]), key[1:])
    // ...
    n.Children[key[0]] = nn

    if nn != nil {
        return true, n, nil
    }
    // Count remaining children
    pos := -1
    for i, cld := range &n.Children {
        if cld != nil {
            if pos == -1 {
                pos = i
            } else {
                pos = -2
                break
            }
        }
    }
    if pos >= 0 {
        // Only one child remains — collapse branch into shortNode
        // ...
    }
```

When a deletion leaves a branch with only one non-nil child, the branch is replaced by a short node. If that single remaining child is itself a short node, their keys are concatenated to avoid a `shortNode{..., shortNode{...}}` chain.

---

## Hashing: From Tree to Root Hash

The `Hash()` method computes the trie's root hash without modifying the database. It walks the tree bottom-up, RLP-encoding each node and either inlining it or replacing it with its Keccak256 hash.

The hashing logic lives in `trie/hasher.go`:

```go
// trie/hasher.go

type hasher struct {
    sha      crypto.KeccakState
    tmp      []byte
    encbuf   rlp.EncoderBuffer
    parallel bool
}

func (h *hasher) hash(n node, force bool) []byte {
    if hash, _ := n.cache(); hash != nil {
        return hash
    }
    switch n := n.(type) {
    case *shortNode:
        enc := h.encodeShortNode(n)
        if len(enc) < 32 && !force {
            buf := make([]byte, len(enc))
            copy(buf, enc)
            return buf
        }
        hash := h.hashData(enc)
        n.flags.hash = hash
        return hash

    case *fullNode:
        enc := h.encodeFullNode(n)
        if len(enc) < 32 && !force {
            buf := make([]byte, len(enc))
            copy(buf, enc)
            return buf
        }
        hash := h.hashData(enc)
        n.flags.hash = hash
        return hash

    case hashNode:
        return n
    // ...
    }
}
```

The **32-byte inlining rule** is the central decision in the hasher:

- **If the RLP-encoded node is less than 32 bytes AND `force` is false:** the raw encoded bytes are returned. This node will be *inlined* (embedded directly in its parent's encoding) rather than referenced by hash.
- **If the RLP-encoded node is 32 bytes or more, OR `force` is true:** the node is Keccak256-hashed, and the 32-byte hash is cached in `n.flags.hash` for future reuse.

The `force` parameter is only `true` for the root node (called from `hashRoot`):

```go
// trie/trie.go

func (t *Trie) hashRoot() []byte {
    if t.root == nil {
        return types.EmptyRootHash.Bytes()
    }
    h := newHasher(t.unhashed >= 100)
    defer func() {
        returnHasherToPool(h)
        t.unhashed = 0
    }()
    return h.hash(t.root, true)
}
```

The root is always hashed (never inlined), even if its encoding is small, because the root hash is the state root stored in the block header.

When hashing a short node, the hasher converts the hex-encoded key to compact encoding before RLP-encoding it. For extension nodes, it recursively hashes the child first:

```go
// trie/hasher.go

func (h *hasher) encodeShortNode(n *shortNode) []byte {
    if hasTerm(n.Key) {
        // Leaf node: encode [compactKey, value]
        var ln leafNodeEncoder
        ln.Key = hexToCompact(n.Key)
        ln.Val = n.Val.(valueNode)
        ln.encode(h.encbuf)
        return h.encodedBytes()
    }
    // Extension node: encode [compactKey, hash(child)]
    var en extNodeEncoder
    en.Key = hexToCompact(n.Key)
    en.Val = h.hash(n.Val, false)
    en.encode(h.encbuf)
    return h.encodedBytes()
}
```

For full nodes with many children, when `parallel` is true (triggered when 100+ nodes are unhashed), each child is hashed in its own goroutine using a separate hasher from the pool. Hashers are pooled via `sync.Pool` to reduce allocation overhead.

---

## Committing: Dirty Nodes to NodeSet

While `Hash()` only computes hashes (non-destructive), `Commit()` collects all dirty nodes into a `trienode.NodeSet` for writing to the database. After committing, the trie is unusable — a new one must be created from the new root.

The commit flow in `trie/trie.go`:

```go
// trie/trie.go

func (t *Trie) Commit(collectLeaf bool) (common.Hash, *trienode.NodeSet) {
    defer func() {
        t.committed = true
    }()
    if t.root == nil {
        // Handle empty trie...
    }
    rootHash := t.Hash()

    if hashedNode, dirty := t.root.cache(); !dirty {
        t.root = hashedNode
        return rootHash, nil
    }
    nodes := trienode.NewNodeSet(t.owner)
    for _, path := range t.deletedNodes() {
        nodes.AddNode(path, trienode.NewDeletedWithPrev(t.prevalueTracer.Get(path)))
    }
    t.root = newCommitter(nodes, t.prevalueTracer, collectLeaf).Commit(t.root, t.uncommitted > 100)
    t.uncommitted = 0
    return rootHash, nodes
}
```

The sequence:

1. **Hash first** — `t.Hash()` ensures all nodes have their hashes computed.
2. **Quick check** — if the root isn't dirty, there's nothing to commit; return nil.
3. **Collect deleted nodes** — nodes that were deleted or replaced are tracked so the database can remove stale entries.
4. **Create a committer** — the `committer` walks the tree and collects every dirty node into the `NodeSet`.

The `committer` in `trie/committer.go` recurses through the tree:

```go
// trie/committer.go

type committer struct {
    nodes       *trienode.NodeSet
    tracer      *PrevalueTracer
    collectLeaf bool
}

func (c *committer) commit(path []byte, n node, parallel bool) node {
    hash, dirty := n.cache()
    if hash != nil && !dirty {
        return hash
    }
    switch cn := n.(type) {
    case *shortNode:
        if _, ok := cn.Val.(*fullNode); ok {
            cn.Val = c.commit(append(path, cn.Key...), cn.Val, false)
        }
        cn.Key = hexToCompact(cn.Key)
        hashedNode := c.store(path, cn)
        // ...
    case *fullNode:
        c.commitChildren(path, cn, parallel)
        hashedNode := c.store(path, cn)
        // ...
    case hashNode:
        return cn
    }
}
```

For each dirty node, the committer:

1. Recursively commits children first (bottom-up).
2. Converts the short node's key from hex to compact encoding (for disk storage).
3. Calls `store()`, which adds the node to the `NodeSet` with both its current encoding and its previous value (for the database to track changes).

The `store` method applies the same 32-byte threshold: if a node's hash is nil (meaning it was too small to be hashed), it's an embedded node that won't be stored independently in the database. Larger nodes are added to the `NodeSet` keyed by their path.

When `parallel` is true (more than 100 uncommitted changes), `commitChildren` processes each child of a full node in its own goroutine, each with a separate `NodeSet` that's merged back under a mutex.

---

## Trie Database: The Persistence Layer

The trie itself is purely in-memory. The `triedb.Database` (in `triedb/database.go`) handles persistence. It wraps a disk key-value store and delegates to one of two backends:

```go
// triedb/database.go

type Database struct {
    disk      ethdb.Database
    config    *Config
    preimages *preimageStore
    backend   backend        // either *hashdb.Database or *pathdb.Database
}

func NewDatabase(diskdb ethdb.Database, config *Config) *Database {
    // ...
    if config.PathDB != nil {
        db.backend = pathdb.New(diskdb, config.PathDB, config.IsVerkle)
    } else {
        db.backend = hashdb.New(diskdb, config.HashDB)
    }
    return db
}
```

The `backend` interface defines the operations both backends must support:

```go
// triedb/database.go

type backend interface {
    NodeReader(root common.Hash) (database.NodeReader, error)
    StateReader(root common.Hash) (database.StateReader, error)
    Size() (common.StorageSize, common.StorageSize)
    Commit(root common.Hash, report bool) error
    Close() error
}
```

### Hash-Based Scheme (hashdb)

The original backend. Nodes are indexed by their **Keccak256 hash** — the same hash used as the node reference inside the trie. This is conceptually simple: to look up a node, hash its reference and query the database.

The hashdb backend keeps dirty nodes in a memory cache with reference counting. When `Commit()` is called with a state root, it walks the tree of dirty nodes starting from that root and flushes them to disk. `Cap()` can be called to evict old cached nodes when memory grows too large.

Key operations: `Reference()` adds a parent→child reference, `Dereference()` removes one. When a root's reference count drops to zero, its nodes can be garbage collected.

### Path-Based Scheme (pathdb)

The newer backend. Nodes are indexed by their **trie path** (the sequence of nibbles from root to the node) rather than by hash. This enables a layer-based diff model:

- A **`diskLayer`** holds the base state that has been flushed to disk.
- **`diffLayer`**s stack on top, each representing the state changes from one block. They form a tree (not a chain) to support reorgs.

When reading a node, pathdb searches from the newest diff layer downward. If no layer has the node, it falls back to disk. When enough diff layers accumulate, the oldest is merged (flattened) into the disk layer.

The path-based scheme is more efficient for block-by-block state management because it naturally groups changes by block and avoids the reference-counting complexity of hashdb. It also supports features like `Journal()` for crash recovery and `Recover()` for rollback to a historical state.

The scheme is selected at database creation time via the `Config`:

```go
// triedb/database.go

var HashDefaults = &Config{
    Preimages: false,
    IsVerkle:  false,
    HashDB:    hashdb.Defaults,
}
```

If `config.PathDB` is set, the path-based backend is used. Otherwise, the hash-based backend is the default.

---

## StateTrie: Key Hashing for Security

The raw `Trie` uses keys as-is. In Ethereum's state trie, this would be dangerous — an attacker could craft keys that create deep, unbalanced paths, slowing down lookups to O(n).

The `StateTrie` (in `trie/secure_trie.go`) solves this by hashing every key with Keccak256 before passing it to the underlying trie:

```go
// trie/secure_trie.go

type StateTrie struct {
    trie        Trie
    db          database.NodeDatabase
    preimages   preimageStore
    secKeyCache map[common.Hash][]byte
}

func (t *StateTrie) GetStorage(_ common.Address, key []byte) ([]byte, error) {
    enc, err := t.trie.Get(crypto.Keccak256(key))
    // ...
}

func (t *StateTrie) UpdateAccount(address common.Address, acc *types.StateAccount, _ int) error {
    hk := crypto.Keccak256(address.Bytes())
    data, err := rlp.EncodeToBytes(acc)
    // ...
    if err := t.trie.Update(hk, data); err != nil {
        return err
    }
    if t.preimages != nil {
        t.secKeyCache[common.Hash(hk)] = address.Bytes()
    }
    return nil
}
```

Every key goes through `crypto.Keccak256()` before reaching the trie. Since Keccak256 outputs are uniformly distributed, the resulting trie is balanced regardless of the input key pattern.

The `secKeyCache` stores the reverse mapping (hash → original key) so that the original key can be recovered when needed. These preimages are flushed to disk during `Commit()`:

```go
// trie/secure_trie.go

func (t *StateTrie) Commit(collectLeaf bool) (common.Hash, *trienode.NodeSet) {
    if len(t.secKeyCache) > 0 {
        if t.preimages != nil {
            t.preimages.InsertPreimage(t.secKeyCache)
        }
        clear(t.secKeyCache)
    }
    return t.trie.Commit(collectLeaf)
}
```

In Ethereum's state model, two `StateTrie` instances are used:

- The **account trie** (also called the world state trie) maps `keccak256(address)` → RLP-encoded account data. Its root is the `Root` field in the block header (see Chapter 02).
- Each account's **storage trie** maps `keccak256(storageSlot)` → RLP-encoded storage value. Its root is the `StorageRoot` field in the account object.

---

## StackTrie: Write-Only Trie for Block Building

The regular `Trie` supports arbitrary reads, inserts, and deletes. But during block building and receipt hashing, geth builds a trie from scratch with keys inserted in sorted order. For this use case, `StackTrie` (in `trie/stacktrie.go`) is a specialized write-only variant that is much more memory-efficient.

```go
// trie/stacktrie.go

type StackTrie struct {
    root       *stNode
    h          *hasher
    last       []byte
    onTrieNode OnTrieNode
    // ...
}

type stNode struct {
    typ      uint8       // node type (emptyNode, branchNode, extNode, leafNode, hashedNode)
    key      []byte
    val      []byte
    children [16]*stNode // no slot 16 — values are stored in val, not children
}

type OnTrieNode func(path []byte, hash common.Hash, blob []byte)
```

Key differences from `Trie`:

- **Sorted insertion only.** `Update` enforces ascending key order and returns an error if keys arrive out of order. Deletions are not supported.
- **Eager hashing.** When a new key is inserted, StackTrie checks if any earlier subtrees are now complete (no future key can land in them because keys are sorted). Completed subtrees are immediately hashed and freed, so only the "right frontier" of the trie is kept in memory.
- **Callback on commit.** The `OnTrieNode` callback is invoked for each node as it's hashed, allowing the caller to write the node to disk immediately. This avoids accumulating all nodes in memory.
- **Simpler node types.** Uses `stNode` with a `typ` field instead of separate Go types. Branch nodes have only 16 children (no slot 16 for values — values go in `val`).

The Update method enforces sorted order:

```go
// trie/stacktrie.go

func (t *StackTrie) Update(key, value []byte) error {
    if len(value) == 0 {
        return errors.New("trying to insert empty (deletion)")
    }
    // ...
    k := writeHexKey(t.kBuf, key)
    if bytes.Compare(t.last, k) >= 0 {
        return errors.New("non-ascending key order")
    }
    // ...
    t.insert(t.root, k, vBuf, t.pBuf[:0])
    return nil
}
```

The hashing logic in `StackTrie.hash()` applies the same 32-byte threshold as the regular hasher — nodes smaller than 32 bytes that aren't the root are inlined. For larger nodes (or the root), the Keccak256 hash is computed, and the `onTrieNode` callback is invoked:

```go
// trie/stacktrie.go (hash method, simplified)

func (t *StackTrie) hash(st *stNode, path []byte) {
    // ... encode node into blob based on st.typ ...

    st.typ = hashedNode
    st.key = st.key[:0]

    if len(blob) < 32 && len(path) > 0 {
        st.val = bPool.getWithSize(len(blob))
        copy(st.val, blob)
        return
    }
    st.val = bPool.getWithSize(32)
    t.h.hashDataTo(st.val, blob)

    if t.onTrieNode != nil {
        t.onTrieNode(path, common.BytesToHash(st.val), blob)
    }
}
```

StackTrie is used in geth for:
- Computing the **transactions root** and **receipts root** in block headers, where transactions/receipts are keyed by their RLP-encoded index in sorted order.
- Writing trie nodes during **snap sync** state download, where accounts arrive in sorted hash order.

---

## What's Next

With the trie implementation covered, we now have the data structure that backs all of Ethereum's state. Chapter 04 — Account and State builds on this foundation to explain how `StateDB` and `stateObject` use the trie to read and write account balances, nonces, contract code, and storage slots.
