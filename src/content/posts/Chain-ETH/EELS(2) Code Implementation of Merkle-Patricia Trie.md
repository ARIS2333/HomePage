---
title: EELS(2) Code Implementation of Merkle-Patricia Trie
published: 2026-03-30
pinned: false
description: A function-by-function walkthrough of how EELS builds, encodes, and hashes the MPT in Python.
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---


# Table of Contents

1. [Preface](#preface)
2. [Introduction](#introduction)
3. [The EELS Trie Module at a Glance](#the-eels-trie-module-at-a-glance)
4. [A Code-Driven Walkthrough](#a-code-driven-walkthrough)
   - [Stage 0: The Trie Data Structure](#stage-0-the-trie-data-structure)
   - [Stage 1: Writing to the Trie — `trie_set`](#stage-1-writing-to-the-trie--trie_set)
   - [Stage 2: Key Transformation — `_prepare_trie`](#stage-2-key-transformation--_prepare_trie)
   - [Stage 3: Building the Tree — `patricialize`](#stage-3-building-the-tree--patricialize)
   - [Stage 4: Node Encoding and the Merkle Rule — `encode_internal_node`](#stage-4-node-encoding-and-the-merkle-rule--encode_internal_node)
   - [Stage 5: Computing the Root — `root`](#stage-5-computing-the-root--root)
   - [Stage 6: Trie Evolution — Inserting a Fourth Account](#stage-6-trie-evolution--inserting-a-fourth-account)
5. [Reading from the Trie — `trie_get`](#reading-from-the-trie--trie_get)
6. [Summary: The EELS Function Reference](#summary-the-eels-function-reference)
7. [Reference](#reference)

---

# Preface

This article is part of a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**.

**What is EELS?**
EELS is a Python reference implementation of the Ethereum execution client, prioritizing readability over performance. It serves as the authoritative, programmer-friendly successor to the Yellow Paper and is the primary tool for prototyping new Ethereum Improvement Proposals (EIPs). EELS provides a complete, executable protocol snapshot at each hard fork, with rendered diffs between them.

**Scope and Limitations**
EELS does not implement JSON-RPC or P2P networking. It validates blocks by receiving data from an external RPC provider, then processing and storing it locally.

**Version Reference**
All code in this article is from the **Osaka** fork of the EELS codebase. The primary file we analyze is:
[`src/ethereum/osaka/trie.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py)

> **Note:** Because the trie algorithm has been stable across forks, the logic is essentially identical from Frontier through Osaka. The osaka version is simply the most up-to-date reference.

---

# Introduction

In [Part 1](../EELS(1)%20What%20is%20Merkle-Patricia%20Trie/), we built a thorough conceptual model of the MPT: its four node types, the three key encodings, the inline-vs-hash rule, and the four tries that Ethereum maintains. This article bridges theory and implementation.

We will trace a concrete example — three accounts entering the state trie — through every function in [`trie.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py), reading each one line by line. By the end, you will understand not just *what* the MPT does, but *exactly how* EELS does it in code.

**What you will learn:**

- How the `Trie` object stores data before the tree is built.
- How `_prepare_trie` transforms raw keys and values into a form the algorithm can consume.
- How `patricialize` recursively decides whether to emit a `LeafNode`, `ExtensionNode`, or `BranchNode`.
- How `encode_internal_node` applies the `< 32 bytes` inline rule and creates the Merkle hash chain.
- How `root` orchestrates the whole pipeline to produce the final `stateRoot`.

---

# The EELS Trie Module at a Glance

Before diving in, here is the full pipeline from a raw key-value write to a final Merkle root. Each box maps directly to a function in `trie.py`:

![High-level pipeline: trie_set → _prepare_trie → patricialize → encode_internal_node → root](EELS(2)%20Code%20Implementation%20of%20Merkle-Patricia%20Tri/image.png)

The pipeline has a critical design characteristic worth stating upfront: **the MPT is never incrementally updated in EELS**. Unlike production clients such as Geth (which maintain a live in-memory trie and update it node-by-node on each transaction), EELS takes a **batch-and-rebuild** approach:

1. All writes go into a plain Python dictionary (`trie._data`).
2. When the root is needed, `patricialize` rebuilds the entire tree from scratch in one recursive pass over the flat dictionary.

This is a deliberate trade-off — simplicity and correctness over performance — perfectly suited to a reference implementation.

---

# A Code-Driven Walkthrough

We'll build a small state trie with three accounts. To keep things concrete, we assume their keccak256-hashed addresses begin with specific nibbles:

| Account | Hashed Key (nibble prefix) | Balance |
|---|---|---|
| **A1** | `[3, 7, a, 2, 9, f, …]` (64 nibbles total) | 100 wei |
| **A2** | `[3, 7, a, 5, 1, c, …]` (64 nibbles total) | 200 wei |
| **A3** | `[8, b, 0, 4, e, 3, …]` (64 nibbles total) | 300 wei |

A1 and A2 share the first three nibbles (`3, 7, a`), which will force the creation of an Extension Node. A3 diverges immediately at nibble `0` (`3` vs `8`).

---

## Stage 0: The Trie Data Structure

Before any function is called, we need a `Trie` object. The
[`Trie` dataclass](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L60)
is defined as:

```python
@dataclass
class Trie(Generic[K, V]):
    secured: bool
    default: V
    _data: Dict[K, V] = field(default_factory=dict)
```

Three fields, and all three matter:

- **`secured`**: If `True`, keys are hashed with `keccak256` before being inserted into the trie. This is `True` for the **State Trie** and **Storage Trie**, and `False` for the **Transaction Trie** and **Receipt Trie**. We explain the security reason in Stage 2.
- **`default`**: The value returned by `trie_get` for a missing key (equivalent to "empty" for this trie type). For the state trie, `default` is `None`; for the storage trie, `default` is `U256(0)`. Crucially, EELS *never stores the default value explicitly* — its absence in `_data` is itself the representation.
- **`_data`**: A plain Python `dict`. This is the entire state of the trie at rest. No tree structure, no hashes — just a key-value map.

---

## Stage 1: Writing to the Trie — `trie_set`

The external world interacts with the trie through
[`trie_set`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L130):

```python
def trie_set(trie: Trie[K, V], key: K, value: V) -> None:
    if value == trie.default:
        if key in trie._data:
            del trie._data[key]
    else:
        trie._data[key] = value
```

This is deceptively simple. Notice what is **not** happening: no hashing, no tree construction, no RLP encoding. `trie_set` is purely a dictionary operation.

The one meaningful behavior is the **deletion-on-default rule**: if the new value equals `trie.default` (i.e., you're "deleting" an account or zeroing a storage slot), the key is removed from `_data` rather than stored with an explicit empty value. This keeps the trie sparse — only accounts with meaningful state exist in `_data`.

After three calls to `trie_set` for A1, A2, and A3, the internal state is simply:

```python
trie._data = {
    b'\xa1...': Account(nonce=0, balance=100, ...),
    b'\xa2...': Account(nonce=0, balance=200, ...),
    b'\xa3...': Account(nonce=0, balance=300, ...),
}
```

No tree has been built yet.

---

## Stage 2: Key Transformation — `_prepare_trie`

The tree is only constructed when someone calls `root()`. The first thing `root()` does is call
[`_prepare_trie`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L193),
which iterates over every `(key, value)` pair in `_data` and applies three transformations, producing a new flat dictionary `{nibble_key: rlp_bytes}`.

```python
def _prepare_trie(
    trie: Trie[K, V],
    get_storage_root: Callable[[Address], Root] = None,
) -> Mapping[Bytes, Bytes]:
    mapped: MutableMapping[Bytes, Bytes] = {}
    for (preimage, value) in trie._data.items():
        if isinstance(value, Account):
            assert get_storage_root is not None
            address = Address(preimage)
            encoded_value = encode_node(value, get_storage_root(address))
        else:
            encoded_value = encode_node(value)
        ensure(encoded_value != b"", AssertionError)
        if trie.secured:
            key = keccak256(preimage)
        else:
            key = preimage
        mapped[bytes_to_nibble_list(key)] = encoded_value
    return mapped
```

### Transformation 1: Value Encoding via `encode_node`

Each Python value — an `Account`, a `U256`, a `Transaction`, a raw `Bytes` — is serialized into a `Bytes` object using
[`encode_node`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L178):

```python
def encode_node(node: Node, storage_root: Optional[Bytes] = None) -> Bytes:
    if isinstance(node, Account):
        assert storage_root is not None
        return encode_account(node, storage_root)
    elif isinstance(node, (Transaction, Receipt, U256)):
        return rlp.encode(cast(rlp.RLP, node))
    elif isinstance(node, Bytes):
        return node
    else:
        raise AssertionError(f"encoding for {type(node)} is not currently implemented")
```

For an `Account`, this calls `encode_account`, which RLP-encodes the four-field account struct `[nonce, balance, storageRoot, codeHash]`. Notice that `encode_account` requires the `storageRoot` — the root hash of the account's own storage trie — as a parameter. This is why `_prepare_trie` accepts the `get_storage_root` callback: the state trie cannot be computed without first computing every contract's storage trie root.

For our three accounts with no contract storage, `storageRoot` is `EMPTY_TRIE_ROOT` (`0x56e8…`). The output for A1 might look like:

```
b'\xd7\x80\x64\xa0\x56\xe8...\xa0\xc5\xd2...'  # RLP([0, 100, EMPTY_ROOT, EMPTY_CODE_HASH])
```

### Transformation 2: Key Hashing (the `secured` flag)

```python
if trie.secured:
    key = keccak256(preimage)   # 32-byte hash of the raw address
else:
    key = preimage
```

**Why hash the key?** If an attacker could freely choose keys, they could craft thousands of addresses sharing a very long common prefix, forcing `patricialize` to build a pathologically deep, unbalanced tree. Each `trie_get` or `trie_set` on such a tree would require O(64) steps instead of the typical O(depth), and the sheer number of shared-prefix nodes could amplify I/O costs into a viable Denial-of-Service vector.

By hashing with `keccak256` first, keys are uniformly distributed in the 256-bit space regardless of what raw addresses a user provides. No attacker can predict or control the hash output, so the tree stays naturally balanced.

**When is `secured` False?** For the **Transaction Trie** and **Receipt Trie**, keys are sequential integers (`RLP(0)`, `RLP(1)`, …). These are assigned by the block producer, not by users, so there is no attack surface. Skipping `keccak256` here saves computation and keeps those tries' keys human-readable.

### Transformation 3: Nibble Expansion via `bytes_to_nibble_list`

```python
def bytes_to_nibble_list(bytes_: Bytes) -> Bytes:
    nibble_list = bytearray(2 * len(bytes_))
    for byte_index, byte in enumerate(bytes_):
        nibble_list[byte_index * 2] = (byte & 0xF0) >> 4   # high nibble
        nibble_list[byte_index * 2 + 1] = byte & 0x0F      # low nibble
    return Bytes(nibble_list)
```

The MPT branches 16-ways (hexary), so each step down the tree consumes exactly one nibble (4 bits), not one byte (8 bits). This function splits every byte into two nibbles and stores each nibble as its own byte-value (0–15) in a new array twice the length.

For a secured (keccak256'd) key, the 32-byte hash becomes a 64-element nibble list. For example, the byte `0xAF` becomes `[0x0A, 0x0F]`.

At the end of `_prepare_trie`, our three accounts are represented as:

```python
mapped = {
    bytes([3, 7, 0xa, 2, 9, 0xf, ...]): b'\xd7\x80\x64...',  # A1: 64 nibbles → RLP bytes
    bytes([3, 7, 0xa, 5, 1, 0xc, ...]): b'\xd7\x80\xc8...',  # A2: 64 nibbles → RLP bytes
    bytes([8, 0xb, 0, 4, 0xe, 3, ...]): b'\xd7\x80\x01...',  # A3: 64 nibbles → RLP bytes
}
```

No tree structure yet — just a flat dictionary with nibble-list keys. This is the input to `patricialize`.

![After _prepare_trie: raw keys become 64-nibble lists, values become RLP bytes](EELS(2)%20Code%20Implementation%20of%20Merkle-Patricia%20Tri/image%201.png)

---

## Stage 3: Building the Tree — `patricialize`

[`patricialize`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L240)
is the heart of the EELS MPT. It takes the flat `mapped` dictionary and a `level` cursor (the current nibble index, starting at 0), and recursively converts a subset of that dictionary into the appropriate node type.

```python
def patricialize(
    obj: Mapping[Bytes, Bytes],
    level: Uint
) -> Optional[InternalNode]:
```

The function has four cases, resolved in order:

### Case 1: Empty dictionary → `None`

```python
if len(obj) == 0:
    return None
```

An empty `obj` means no keys route through this branch. The node is absent, encoded as `b""` by `encode_internal_node`.

### Case 2: Single key → `LeafNode`

```python
arbitrary_key = next(iter(obj))

if len(obj) == 1:
    leaf = LeafNode(arbitrary_key[level:], obj[arbitrary_key])
    return leaf
```

If `obj` contains only one key-value pair, there is no branching to do. The entire remaining path from `level` onward (`arbitrary_key[level:]`) becomes the `rest_of_key` of a `LeafNode`, and the RLP-encoded value is stored directly.

This base case fires for A3 the moment it is isolated into its own branch bucket.

### Case 3: Shared prefix → `ExtensionNode`

```python
substring = arbitrary_key[level:]
prefix_length = len(substring)
for key in obj:
    prefix_length = min(
        prefix_length,
        common_prefix_length(substring, key[level:])
    )
    if prefix_length == 0:
        break

if prefix_length > 0:
    prefix = arbitrary_key[level : level + prefix_length]
    return ExtensionNode(
        prefix,
        encode_internal_node(patricialize(obj, level + prefix_length)),
    )
```

This block computes the **longest common prefix** shared by all keys in `obj` at the current level. It uses a running `min()` across all keys: if any key reduces the shared prefix to zero, the loop breaks early.

[`common_prefix_length`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L108)
is straightforward:

```python
def common_prefix_length(a: Sequence, b: Sequence) -> int:
    for i in range(len(a)):
        if i >= len(b) or a[i] != b[i]:
            return i
    return len(a)
```

In our example, when `obj` contains only A1 and A2 (after A3 is separated), the comparison at `level=1` finds:

- A1 remaining: `[7, a, 2, 9, f, …]`
- A2 remaining: `[7, a, 5, 1, c, …]`
- Common prefix: `[7, a]` → `prefix_length = 2`

An `ExtensionNode` is created with `key_segment = [7, a]`. The `subnode` field is the result of calling `encode_internal_node(patricialize(obj, level + 2))` — recursing with the same dictionary but advancing `level` past the shared prefix. Notice that the same `obj` is passed; only the `level` cursor advances, instructing the next call to look at nibble `level+2` and beyond.

### Case 4: No shared prefix → `BranchNode`

```python
branches: List[MutableMapping[Bytes, Bytes]] = []
for _ in range(16):
    branches.append({})
value = b""

for key in obj:
    if len(key) == level:
        if isinstance(obj[key], (Account, Receipt, Uint)):
            raise AssertionError
        value = obj[key]
    else:
        branches[key[level]][key] = obj[key]

return BranchNode(
    [
        encode_internal_node(patricialize(branches[k], level + 1))
        for k in range(16)
    ],
    value,
)
```

When `prefix_length == 0`, keys diverge at the current nibble. A `BranchNode` is needed.

The code creates 16 empty dictionaries (one per hex nibble). It then iterates over all keys in `obj` and routes each one into the bucket corresponding to its nibble at position `level` (`branches[key[level]]`).

In our first call (`level=0`):
- A1 and A2 (first nibble `3`) → `branches[3]`
- A3 (first nibble `8`) → `branches[8]`
- All other buckets remain empty.

The `if len(key) == level` branch handles a key that terminates exactly at this branch position (i.e., some account's full 64-nibble path ends right here). For the state trie, this can never happen — all keys are exactly 64 nibbles — so this path raises an `AssertionError` if an `Account` or `Receipt` is encountered there. The comment in the source confirms: *"shouldn't ever have an account or receipt in an internal node."*

Each bucket is recursed with `level + 1` (consuming the routing nibble itself), and `encode_internal_node` is applied to each result. Empty buckets produce `b""` (encoded `None`), which becomes the 16 empty slots in the branch.

**The full recursive trace for our three accounts:**

```
patricialize({A1, A2, A3}, level=0)
  → prefix_length=0 at nibble 3 vs 8 → BranchNode
  → branches[3] = {A1, A2},  branches[8] = {A3}

    patricialize({A1, A2}, level=1)
      → common prefix [7, a] → ExtensionNode(key=[7,a])
      → patricialize({A1, A2}, level=3)
          → prefix_length=0 at nibble 2 vs 5 → BranchNode
          → branches[2] = {A1},  branches[5] = {A2}

              patricialize({A1}, level=4)
                → single key → LeafNode(rest=[9,f,...], value=RLP(A1))

              patricialize({A2}, level=4)
                → single key → LeafNode(rest=[1,c,...], value=RLP(A2))

    patricialize({A3}, level=1)
      → single key → LeafNode(rest=[b,0,4,e,3,...], value=RLP(A3))
```

![patricialize recursion trace and resulting tree structure](EELS(2)%20Code%20Implementation%20of%20Merkle-Patricia%20Tri/image%206.png)

The resulting in-memory tree (before hashing):

![In-memory tree: BranchNode at root → Extension(7,a) and LeafNode for A3; Extension → inner BranchNode → two LeafNodes](EELS(2)%20Code%20Implementation%20of%20Merkle-Patricia%20Tri/image%202.png)

---

## Stage 4: Node Encoding and the Merkle Rule — `encode_internal_node`

Because `patricialize` is recursive and builds the tree bottom-up, every node is immediately wrapped in
[`encode_internal_node`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L143)
as it is returned. This function is what transforms a plain Python tree into a **Merkle tree**.

```python
def encode_internal_node(node: Optional[InternalNode]) -> rlp.RLP:
    unencoded: rlp.RLP
    if node is None:
        unencoded = b""
    elif isinstance(node, LeafNode):
        unencoded = (
            nibble_list_to_compact(node.rest_of_key, True),
            node.value,
        )
    elif isinstance(node, ExtensionNode):
        unencoded = (
            nibble_list_to_compact(node.key_segment, False),
            node.subnode,
        )
    elif isinstance(node, BranchNode):
        unencoded = node.subnodes + [node.value]
    else:
        raise AssertionError(f"Invalid internal node type {type(node)}!")

    encoded = rlp.encode(unencoded)
    if len(encoded) < 32:
        return unencoded       # Embed inline — no database entry needed
    else:
        return keccak256(encoded)  # Return 32-byte hash reference
```

This function has three logically distinct responsibilities:

### Step A: Prepare the Node for RLP Serialization

**For `BranchNode`:** Its 16-element `subnodes` tuple (each already an `encode_internal_node` result) plus the `value` slot are concatenated into a 17-element list. No path encoding is needed — branch nodes carry no key fragment.

**For `LeafNode` and `ExtensionNode`:** Their `rest_of_key` / `key_segment` fields are nibble sequences and must be converted to byte-aligned Compact (HP) encoding before serialization.

### Step B: Compact (Hex-Prefix) Encoding via `nibble_list_to_compact`

```python
def nibble_list_to_compact(x: Bytes, is_leaf: bool) -> Bytes:
    compact = bytearray()
    if len(x) % 2 == 0:                          # even length
        compact.append(16 * (2 * is_leaf))        # prefix byte: flag nibble + 0 padding nibble
        for i in range(0, len(x), 2):
            compact.append(16 * x[i] + x[i + 1])
    else:                                          # odd length
        compact.append(16 * ((2 * is_leaf) + 1) + x[0])  # prefix byte: flag+1 nibble + first nibble
        for i in range(1, len(x), 2):
            compact.append(16 * x[i] + x[i + 1])
    return Bytes(compact)
```

This is the
[`nibble_list_to_compact`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L116)
implementation — the "Hex-Prefix" encoding described conceptually in Part 1. It packs nibbles back into bytes and embeds the node-type and parity flags into the first byte:

| Situation | First byte (hex) | Explanation |
|---|---|---|
| Extension, even path | `0x00` | flag=0, even → padding nibble 0 |
| Extension, odd path | `0x1X` | flag=1, odd → first nibble X absorbed |
| Leaf, even path | `0x20` | flag=2, even → padding nibble 0 |
| Leaf, odd path | `0x3X` | flag=3, odd → first nibble X absorbed |

**Worked examples using our data:**

*ExtensionNode with `key_segment = [7, a]` (even length):*
```
len([7, a]) % 2 == 0  →  even
first byte = 16 * (2 * False) = 0x00
pack [7, a] → 0x7a
Result: b'\x00\x7a'
```

*LeafNode for A1 with `rest_of_key = [9, f, …]` (even, 60 nibbles):*
```
len = 60  →  even
first byte = 16 * (2 * True) = 0x20
pack remaining nibbles in pairs
Result: b'\x20\x9f...'
```

*LeafNode for A2 with a hypothetical odd-length `rest_of_key = [1, c, b]` (odd, 3 nibbles):*
```
len = 3  →  odd
first byte = 16 * ((2*True)+1) + x[0] = 16*3 + 1 = 0x31
pack [c, b] → 0xcb
Result: b'\x31\xcb'
```

### Step C: The Merkle Hashing Rule

After `unencoded` is prepared, the entire structure is serialized with `rlp.encode`:

```python
encoded = rlp.encode(unencoded)
if len(encoded) < 32:
    return unencoded        # Inline: return the raw structure, not the bytes
else:
    return keccak256(encoded)  # Hash: return 32-byte Keccak reference
```

This is the **inline-vs-hash decision** we introduced in Part 1, now visible in exact code.

**Why return `unencoded` rather than `encoded` for the inline case?** When a node is inlined, its parent's `encode_internal_node` call will receive it, include it in the parent's own `unencoded` structure, and then call `rlp.encode` on the whole thing. RLP encoding is recursive — the inner structure is re-encoded as part of the outer structure. Returning the raw Python structure (`unencoded`) lets RLP handle the nesting correctly. Returning the already-encoded bytes would produce a double-encoding bug.

**The effect on our example:**

- A1's `LeafNode` (64-nibble key + ~40 byte RLP account): `rlp.encode(...)` is likely ≥ 32 bytes → **hashed**.
- The inner `BranchNode` (routing between A1 and A2): 17 slots, most filled with hashes → ≥ 32 bytes → **hashed**.
- The `ExtensionNode` (`[7, a]` + one 32-byte hash): RLP is ≥ 32 bytes → **hashed**.
- Empty branch slots: `b""` (1 byte) → **inlined** as `b""` directly.

![Bottom-up hash propagation: leaves hashed first, hashes flow up to root](EELS(2)%20Code%20Implementation%20of%20Merkle-Patricia%20Tri/image%204.png)

This is the bottom-up Merkle accumulation. Because `encode_internal_node` is invoked on return from each recursive `patricialize` call, the entire tree is encoded and hashed in leaf-to-root order.

![Final encoded tree with hash values propagating to the root](EELS(2)%20Code%20Implementation%20of%20Merkle-Patricia%20Tri/image%203.png)

---

## Stage 5: Computing the Root — `root`

[`root`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L215)
orchestrates the entire pipeline:

```python
def root(
    trie: Trie[K, V],
    get_storage_root: Callable[[Address], Root] = None,
) -> Root:
    obj = _prepare_trie(trie, get_storage_root)
    root_node = encode_internal_node(patricialize(obj, Uint(0)))
    if len(rlp.encode(root_node)) < 32:
        return keccak256(rlp.encode(root_node))
    else:
        assert isinstance(root_node, Bytes)
        return Root(root_node)
```

After `patricialize` finishes, `encode_internal_node` is applied one final time to the root node. The result is either:

- A **32-byte hash** (the normal case for any real-world state trie): returned directly as the `Root`.
- A **small inline structure** (only for a tiny trie with a single node that serializes to < 32 bytes): the code hashes it explicitly with `keccak256` to ensure the root is always exactly 32 bytes.

The second case deserves emphasis: **`root()` guarantees a 32-byte output unconditionally.** In mainnet conditions with millions of accounts, the root node is always a hashed `BranchNode`. But for edge cases — an empty trie, a trie with a single small leaf — the fallback `keccak256` ensures the block header always receives a valid 32-byte `stateRoot`.

The special constant
[`EMPTY_TRIE_ROOT`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L55)
is `keccak256(rlp.encode(b""))` = `0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421`, which is exactly what `root()` returns for an empty trie (where `patricialize` returns `None` → `encode_internal_node(None)` = `b""` → `rlp.encode(b"")` < 32 bytes → hashed).

---

## Stage 6: Trie Evolution — Inserting a Fourth Account

To see the Merkle cascade in action, let's add a fourth account:

- **A4**: hashed key prefix `[3, 7, a, 5, 8, …]` (shares `[3, 7, a, 5]` with A2)

After `trie_set(trie, addr_A4, account_A4)` and another call to `root()`, `_prepare_trie` rebuilds the flat dictionary with all four entries, and `patricialize` runs fresh from scratch.

The structural change happens deep in the tree:

1. **A2's old LeafNode** (which stored `rest_of_key = [1, c, …]`) can no longer be a leaf, because A4 also routes through nibble `5` of the inner BranchNode (the one separating A1 and A2).
2. A new **ExtensionNode** is created for the shared prefix `[8]` (the next nibble both A2 and A4 share after position 4).

   Wait — let's be precise. At `level=4`, A2's remaining key starts with `[1, c, …]` and A4's starts with `[8, …]`. They share no prefix at level 4. So the inner BranchNode simply grows another occupied slot: slot `1` → A2's leaf, slot `8` → A4's leaf. No extension node is needed here.

3. **All hashes on the path from that inner BranchNode to the root are invalidated and recomputed**: the inner `BranchNode` hash changes → the `ExtensionNode([7, a])` hash changes → the outer `BranchNode` slot 3 changes → the outer `BranchNode` hash changes → the **State Root** changes.

![After inserting A4: split of A2's leaf, new slot in inner BranchNode, cascade of hash updates to root](EELS(2)%20Code%20Implementation%20of%20Merkle-Patricia%20Tri/image%205.png)

This illustrates the **Merkle accumulator property**: you cannot change a single wei of any account's balance without producing a detectably different State Root. And because only the path from the changed leaf to the root is rehashed, the cost is O(log n) operations — not O(n).

---

# Reading from the Trie — `trie_get`

We have focused on writing and root computation. For completeness,
[`trie_get`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L155)
is the read counterpart:

```python
def trie_get(trie: Trie[K, V], key: K) -> V:
    return trie._data.get(key, trie.default)
```

In EELS, this is nothing more than a Python `dict.get` with a default fallback. There is no trie traversal, no hash lookup, no RLP decoding. The `_data` dictionary is always the source of truth; the MPT tree structure exists only ephemerally during `root()` computation.

This confirms the batch-and-rebuild design: EELS intentionally keeps reads and writes as plain dictionary operations, and only incurs the full MPT construction cost when computing a state root.

---

# Summary: The EELS Function Reference

| Function | Location | Role | Key Behavior |
|---|---|---|---|
| [`Trie`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L60) | `trie.py` | Data container | Holds `secured` flag, `default` value, and raw `_data` dict |
| [`trie_set`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L130) | `trie.py` | Write | Deletes key if `value == default`; else stores in `_data` |
| [`trie_get`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L155) | `trie.py` | Read | Returns `_data.get(key, default)` — plain dict lookup, no tree traversal |
| [`encode_node`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L178) | `trie.py` | Value serialization | RLP-encodes `Account`, `Transaction`, `Receipt`, `U256`, or passes through `Bytes` |
| [`_prepare_trie`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L193) | `trie.py` | Key/value prep | Optionally hashes keys via `keccak256`; expands to nibbles; encodes values |
| [`bytes_to_nibble_list`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L102) | `trie.py` | Nibble expansion | Splits each byte into two 4-bit nibbles; doubles key length |
| [`common_prefix_length`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L108) | `trie.py` | Prefix detection | Linear scan returning length of shared prefix between two sequences |
| [`patricialize`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L240) | `trie.py` | Tree construction | Recursive: emits `LeafNode` (single key), `ExtensionNode` (shared prefix), or `BranchNode` (divergence) |
| [`nibble_list_to_compact`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L116) | `trie.py` | HP encoding | Packs nibbles into bytes; embeds leaf/odd flags in first byte |
| [`encode_internal_node`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L143) | `trie.py` | Merkle encoding | Applies HP encoding; RLP-serializes; inlines if < 32 bytes, else returns `keccak256` |
| [`root`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py#L215) | `trie.py` | Orchestration | Calls `_prepare_trie` → `patricialize` → `encode_internal_node`; always returns 32-byte `Root` |

---

# Reference

1. EELS `trie.py` (Osaka fork) — https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/trie.py
2. Ethereum Yellow Paper, Appendix D — Modified Merkle Patricia Tree — https://ethereum.github.io/yellowpaper/paper.pdf
3. Ethereum.org — Patricia Merkle Trie — https://ethereum.org/developers/docs/data-structures-and-encoding/patricia-merkle-trie/
4. EELS Specification Docs — https://ethereum.github.io/execution-specs/
5. RLP Encoding Specification — https://ethereum.org/developers/docs/data-structures-and-encoding/rlp/
