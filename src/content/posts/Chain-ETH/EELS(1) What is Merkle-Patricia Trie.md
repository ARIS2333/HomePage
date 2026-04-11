---
title: EELS(1) What is Merkle-Patricia Trie
published: 2026-03-27
pinned: false
description: A comprehensive theoretical deep dive into the MPT architecture.
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

# Table of Contents

1. [Preface](#preface)
2. [Introduction](#introduction)
3. [Theoretical Background](#theoretical-background)
   - [Merkle Tree](#merkle-tree)
   - [Trie](#trie)
   - [Patricia Trie](#patricia-trie)
   - [Combining the Best of Both Worlds](#combining-the-best-of-both-worlds)
4. [MPT Breakdown](#mpt-breakdown)
   - [Understanding the "Key" in Ethereum](#understanding-the-key-in-ethereum)
   - [Node Types in the MPT](#node-types-in-the-mpt)
   - [Inline Nodes vs. Hash References](#inline-nodes-vs-hash-references)
   - [Key Encoding in the MPT](#key-encoding-in-the-mpt)
   - [A Worked Example: MPT Evolution](#a-worked-example-mpt-evolution)
5. [From Theory to Implementation: Geth Internals](#from-theory-to-implementation-geth-internals)
   - [In-Memory Representation: nodeFlag and Node Types](#in-memory-representation-nodeflag-and-node-types)
   - [On-Disk Persistence: Content-Addressed Storage](#on-disk-persistence-content-addressed-storage)
   - [Lazy Loading: Traversing Without Full State](#lazy-loading-traversing-without-full-state)
   - [What Is Actually Stored in LevelDB?](#what-is-actually-stored-in-leveldb)
6. [The Four MPTs of Ethereum](#the-four-mpts-of-ethereum)
7. [Reference](#reference)

---

# Preface

This article is the first in a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**.

**What is EELS?**
EELS is a Python reference implementation of the Ethereum execution client, designed with a focus on readability and clarity. It serves as a programmer-friendly, up-to-date successor to the original Yellow Paper, and is the primary tool for prototyping new Ethereum Improvement Proposals (EIPs). EELS provides complete protocol snapshots at each fork with rendered diffs between them.

**Scope and Limitations**
EELS does not implement the JSON-RPC API or P2P networking layer. To validate blocks, it requires an external RPC provider to supply raw data, which EELS then processes and stores in a local database.

**Version Reference**
The code analysis in this article is based on the **Osaka** fork of the execution specs:
[ethereum/execution-specs](https://github.com/ethereum/execution-specs)

---

# Introduction

In Ethereum, **state is everything**. Every account balance, every smart contract storage variable, and every transaction receipt must be stored in a way that is simultaneously efficient to access and cryptographically verifiable. The **Merkle-Patricia Trie (MPT)** is the data structure that satisfies both requirements.

Before we can read the EELS source code to understand transaction execution or block validation, we must understand the container that holds all of Ethereum's data. This first installment peels back the layers of the MPT:

- **The Building Blocks:** How Ethereum combines the cryptographic integrity of Merkle Trees with the retrieval efficiency of Patricia Tries.
- **The Anatomy of a Node:** A deep dive into the four node types that make the trie both compact and verifiable.
- **The Three Faces of a Key:** Why Ethereum keys are transformed between three encoding formats as they move from user input to in-memory traversal to on-disk storage.
- **The Persistence Model:** What Geth actually writes to LevelDB, including the critical inline-node optimization and the reality of how account data is stored.

By the end of this article, you will understand how Ethereum can cryptographically prove the existence (or absence) of a single account among millions using only a short sequence of hashes, and why this design is essential for both full nodes and light clients.

---

# Theoretical Background

To understand the MPT, we must first decompose its constituent parts: the **Merkle Tree**, the **Trie**, and the **Patricia Trie**.

At a high level, the MPT fuses two complementary strengths: the Merkle Tree contributes **cryptographic integrity** via upward hash propagation, while the Patricia Trie contributes **efficient key-value lookup** via prefix-based path compression.

---

## Merkle Tree

Ralph Merkle introduced Merkle Trees in his 1979 doctoral dissertation at Stanford. They are a foundational cryptographic primitive that allows the integrity of a large dataset to be verified without downloading the entire dataset.

### How It Works

Data items (such as transactions) sit at the bottom of the tree as **leaves**. Each leaf is hashed individually. Then, pairs of leaf hashes are hashed together to form parent nodes. This process repeats up the tree until a single hash remains: the **Merkle Root**.

![Merkle Tree structure](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image.png)

This structure creates a compact "digital fingerprint" of the entire dataset. The properties of cryptographic hash functions guarantee:

- **Integrity:** Changing any single leaf (e.g., D1) propagates upward — its hash changes, its parent's hash changes, and ultimately the Root changes. Tampering is immediately detectable.
- **Efficiency:** Locating a modification or verifying a leaf requires traversing only from the root to that leaf: **O(log n)** operations.

For example, once a discrepancy is detected at the Root, we can follow the path Root → N4 → N1 to locate the modified data block D1 in at most O(log n) steps.

### Applications

**Efficient integrity checks:** Rather than comparing two large databases byte-by-byte, systems compare only their Merkle Roots. Equal roots imply equal datasets.

**Membership proofs (Merkle Proofs):** A user can prove that a specific transaction exists in a block without downloading the full block. By providing the transaction and the hashes of its sibling nodes at each level (the "Merkle path"), a light client can recompute the root and check it against the trusted block header.

**Non-existence proofs:** Standard Merkle Trees do not natively support non-membership proofs. Special constructions — such as sorted Merkle trees or Sparse Merkle Trees (SMTs) — are required for this. In Ethereum's MPT specifically, non-membership is proven by demonstrating that following the key's path through the trie yields a null node or a mismatched partial key, rather than the target leaf.

### Advantages

- **Incremental rehashing:** When one piece of data changes, only the O(log n) nodes on the path from that leaf to the root need to be rehashed.
- **Light client support:** Devices with limited storage can participate in the network by storing only block headers (containing the Merkle Root) and requesting on-demand proofs from full nodes.

![Light node requesting a Merkle proof from a full node](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%201.png)

### Limitations

- **Storage overhead:** All intermediate hash nodes must be stored in addition to the leaf data.
- **Position-based organization:** Standard Merkle Trees index data by position (0, 1, 2, …), not by an arbitrary key. This makes them unsuitable for key-value lookups where keys are not pre-assigned integer positions. This is precisely the gap that the **Trie** structure addresses.

---

## Trie

The word *trie* is derived from "re**trie**val." Unlike a standard tree — where the relationships between nodes depend on the application's data model — in a **trie**, the position of a node is determined entirely by the key used to store it. Following the path from the root to any node spells out the key associated with the data at that node.

### How It Works

A trie stores keys by decomposing them into individual characters or symbols. Each edge in the tree represents one character, and traversing from root to a leaf spells out the key.

![Trie with shared prefixes — Ball, Bat, Bear](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%202.png)

Multiple keys sharing a common prefix share the same path from the root down to the point where they diverge. In a typical implementation for English words, each node is an array of 27 pointers: indices 0–25 for letters 'a'–'z', and one slot for a terminator indicating a valid word ends here.

![Internal trie node as an array of 27 slots, most null](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%203.png)

This direct-indexing structure is fast, but results in many null pointers in sparse datasets.

### Advantages

- **Prefix queries in O(m):** Finding all keys with a given prefix requires traversing only to the prefix's node and then exploring its subtree. Hash tables require a full scan for this operation.
- **No hash collisions:** Each key's unique sequence of characters defines a unique path. No two distinct keys share the same path.

### Disadvantages

- **O(m) lookup per key length:** A trie lookup requires one step per character in the key.
- **Space inefficiency for long, unique keys:** If a key is very long and shares no prefix with other keys, the trie must allocate a separate node for each character, creating a chain of single-child nodes. This is particularly wasteful in a blockchain context where keys are typically 32-byte (256-bit) hashes.

---

## Patricia Trie

The **Patricia Trie** (Practical Algorithm To Retrieve Information Coded In Alphanumeric) solves the space inefficiency of the standard trie by **collapsing chains of single-child nodes into a single edge labeled with the entire shared substring**.

![Standard Trie vs. Patricia Trie — path compression of long unique keys](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%204.png)

Rather than creating one node per character along an unbranched path, the Patricia Trie stores the entire shared segment as a single edge label. This reduces both the depth of the tree and the number of database lookups required for traversal.

---

## Combining the Best of Both Worlds

The **Merkle-Patricia Trie** unifies these two structures:

| From the Merkle Tree (Security) | From the Patricia Trie (Efficiency) |
|---|---|
| Every node stores a hash of its children, making the root hash a commitment to the entire dataset | Keys define paths; the tree is organized for efficient prefix-based lookup |
| Any modification to any leaf propagates upward to invalidate the root hash | Long non-branching paths are compressed into single nodes |
| Light clients can verify membership with a short proof path | For any given set of key-value pairs, there is exactly one unique trie structure (determinism) |

**Determinism** deserves emphasis: given the same key-value pairs, the MPT **always produces the same root hash**, regardless of insertion order. This property is what allows every Ethereum node to independently compute and agree on the same State Root.

---

# MPT Breakdown

The core design of the MPT is straightforward: organize data using a trie (navigated by key nibbles), and secure the entire structure using Merkle-style hash chaining. Every node in the MPT is referenced by the hash of its contents (or inlined directly when small — more on this shortly), so the root hash cryptographically commits to every piece of data in the tree.

## Understanding the "Key" in Ethereum

In a toy trie, keys might be English words. In Ethereum, the key for a state trie entry is `keccak256(account_address)` — a 32-byte (64 hex character) value. For illustration we use the shortened form:

```
a711355
```

In practice this is a full 64-nibble hex string, but the short form demonstrates all the structural concepts.

Hashing addresses before using them as trie keys serves two purposes: (1) it produces fixed-length, uniformly distributed keys regardless of address distribution, and (2) it prevents adversarial key crafting that could produce pathologically deep tries.

The MPT processes keys as sequences of **nibbles** — 4-bit units, each representing one hex digit (`0`–`f`). A 32-byte key becomes a 64-nibble path. Each step down the tree consumes one nibble, which is why Branch Nodes have exactly 16 children slots.

---

## Node Types in the MPT

To avoid the "sparse node" problem of standard tries, Ethereum's MPT defines four distinct node types.

### 1. Null Node

The simplest node. Represents an empty tree or an empty slot within a Branch. Encoded as an empty byte string `""` in RLP.

### 2. Branch Node

Used wherever the path diverges into two or more directions. A Branch Node is a 17-element list:

```
[child_0, child_1, ..., child_15, value]
```

![Branch Node with 16 child slots (0x0–0xf) and one value slot (index 16)](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%205.png)

- **Slots 0–15:** One slot per hex nibble. Each slot contains either null (no child in that direction) or a **node reference** pointing to the next child node. (Node references are explained in detail in the next section.)
- **Slot 16:** If a key terminates exactly at this branch node (i.e., no remaining nibbles), the associated value is stored here. Otherwise it is empty.

### 3. Leaf Node

Used when a path reaches its unique endpoint — no other keys in the trie share this terminal path segment. A Leaf Node has two fields:

```
[encodedPath, value]
```

![Leaf Node with remaining path "11355" and encoded account data as value](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%206.png)

- **`encodedPath`:** The remaining nibble sequence (the part of the key not yet consumed by parent nodes), encoded using HP encoding (described in the next section).
- **`value`:** The actual data payload. In the **state trie**, this is the RLP encoding of `[nonce, balance, storageRoot, codeHash]`. In the **storage trie**, it is the RLP encoding of the slot value.

### 4. Extension Node

Used when multiple keys share a long common prefix before they diverge. Rather than creating a Branch Node for every shared nibble, an Extension Node compresses the entire shared segment:

```
[encodedPath, nodeReference]
```

![Extension Node with "a7" prefix pointing to a Branch Node](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%207.png)

- **`encodedPath`:** The shared nibble sequence, stored using HP encoding.
- **`nodeReference`:** A pointer to the next node (typically a Branch Node where the paths diverge). This reference is either an inline embedding or a 32-byte hash — explained in the next section.

**Distinguishing Leaf from Extension:** Both are 2-element lists. The HP encoding's prefix nibble (detailed in the later section) differentiates them: leaf paths carry a flag bit set to 1, extension paths to 0.

### Node Summary

![Full MPT overview — Extension, Branch, and Leaf nodes](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/Gemini_Generated_Image_347jbs347jbs347j.png)

| Node Type | When Used | Structure |
|---|---|---|
| **Extension** | Multiple keys share a non-branching common prefix | `[encodedPath, nodeReference]` |
| **Branch** | Paths diverge in two or more directions | `[child_0, …, child_15, value]` |
| **Leaf** | Path terminates uniquely at this node | `[encodedPath, value]` |
| **Null** | Empty slot or empty tree | `""` (empty bytes) |

**Quick example:** For keys `a711355`, `a77d337`, and `a7f9365`:

1. An **Extension Node** stores the shared prefix `a7` and points to a Branch Node.
2. The **Branch Node** routes on the third nibble: slot `1` → first key, slot `7` → second key, slot `f` → third key.
3. Three **Leaf Nodes** store the remaining suffixes (`11355`, `7d337`, `9365`) and their associated account data.

![Three-key MPT example with Extension, Branch, and Leaf nodes](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%208.png)

---

## Inline Nodes vs. Hash References

In MPT, **Not all child references are 32-byte hashes.**

The Ethereum Yellow Paper specifies a size-based rule for how a node reference is computed:

> If the RLP encoding of a node is **fewer than 32 bytes**, the RLP-encoded node is embedded (inlined) directly into its parent.  
> If the encoding is **32 bytes or more**, the node is stored in the database under key `keccak256(RLP(node))`, and the 32-byte hash is used as the reference.

Expressed as pseudocode:

```python
def node_reference(node):
    encoded = RLP(node)
    if len(encoded) < 32:
        return encoded                  # Inline: embed the raw RLP directly
    else:
        db[keccak256(encoded)] = encoded
        return keccak256(encoded)       # Hash reference: 32-byte key
```

![Inline node vs hash reference decision rule](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/inline_vs_hash.png)

This optimization has significant practical implications:

- **Small nodes** (e.g., short leaf nodes in a sparse storage trie) are inlined into their parent, eliminating a database round-trip.
- **Large nodes** (e.g., a Branch Node with many populated children) are stored separately and referenced by hash.
- **Proof sizes** are affected: inlined nodes are not separate database entries, so a Merkle proof embeds their RLP data directly rather than providing a hash for the verifier to look up.

Throughout the rest of this article, whenever we say "node reference," we mean: either an inline RLP encoding, or a 32-byte keccak256 hash — whichever the size rule dictates.

---

## Key Encoding in the MPT

A key does not maintain a single fixed form throughout its lifecycle. It transitions between three encoding formats depending on context:

| Context | Format | Purpose |
|---|---|---|
| External API | **Raw** | Raw byte array (e.g., account address bytes) |
| In-memory traversal | **Hex (nibble)** | Decomposed into nibbles for branch navigation |
| On-disk storage | **Hex-Prefix (HP)** | Compact byte encoding with embedded node-type flag |

### Raw Encoding

The natural format of the key. For example, the string `"cat"` is the byte array `[0x63, 0x61, 0x74]`. This is the format presented to the MPT's public API.

### Hex (Nibble) Encoding

When traversing the trie in memory, we process the key one nibble at a time. Each byte is split into its high and low nibbles:

```
'c' = 0x63  →  6, 3
'a' = 0x61  →  6, 1
't' = 0x74  →  7, 4
```

So `"cat"` becomes the nibble sequence `[6, 3, 6, 1, 7, 4]`.

**The terminator:** In memory, we distinguish Leaf Nodes from Extension Nodes by appending a special terminator nibble `16` (outside the normal 0–15 range) to Leaf Node paths:

- **Leaf path:** `[6, 3, 6, 1, 7, 4, 16]` — terminator present
- **Extension path:** `[6, 3, 6, 1, 7, 4]` — no terminator

### Hex-Prefix (HP) Encoding

When writing to disk, hex encoding faces two problems:

1. **Byte alignment:** Disks store bytes (8 bits), not nibbles (4 bits). A nibble sequence must be packed into whole bytes.
2. **The terminator value 16** cannot be stored as a nibble. We need a compact, byte-aligned way to encode node type and path parity together.

HP encoding prepends a single **prefix nibble** to the path. This nibble encodes two bits: the high bit (value 2) is the leaf flag, and the low bit (value 1) is the odd-parity flag:

| Prefix nibble | Node type | Path length parity | Action |
|---|---|---|---|
| `0x0` | Extension | Even | Add padding nibble `0` after prefix |
| `0x1` | Extension | Odd  | No padding needed |
| `0x2` | Leaf       | Even | Add padding nibble `0` after prefix |
| `0x3` | Leaf       | Odd  | No padding needed |

![HP encoding table with four cases](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%209.png)

The padding ensures the total number of nibbles is always even, so they can be packed cleanly into bytes.

**Worked example — encoding `"cat"` as a Leaf:**

![Raw → Hex → HP encoding pipeline for "cat"](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2010.png)

1. **Raw:** `[0x63, 0x61, 0x74]`
2. **Hex (memory):** `[6, 3, 6, 1, 7, 4, 16]` — leaf, terminator appended
3. **HP (disk):**
   - Path nibbles (strip terminator): `[6, 3, 6, 1, 7, 4]` — even length
   - Node type = Leaf (bit 1 = 1), parity = Even (bit 0 = 0) → prefix nibble = `0x2`
   - Even path → pad with `0`: total nibbles = `[2, 0, 6, 3, 6, 1, 7, 4]`
   - Pack into bytes: `0x20 0x63 0x61 0x74`

**Summary:**

- **Raw → Hex:** Decomposes bytes into nibbles for nibble-by-nibble tree traversal.
- **Hex → HP:** Packs nibbles back into bytes for disk storage, embedding the node-type flag in the prefix nibble.

---

## A Worked Example: MPT Evolution

Let's trace the construction of a small state trie with four accounts. Short hex keys are used for illustration; real keys are 64-nibble keccak256 hashes.

| Key (hex nibbles) | Account Value |
|---|---|
| `a711355` | 45.0 ETH |
| `a77d337` | 1.00 WEI |
| `a7f9365` | 1.1 ETH |
| `a77d397` | 0.12 ETH |

**Step 1 — Insert `a711355`:**
Only one key exists. No branching is needed. The entire path is stored as a single **Leaf Node**.

![Step 1: single Leaf Node for the first key](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2011.png)

**Step 2 — Insert `a77d337`:**
Both keys share prefix `a7`. The trie restructures: an **Extension Node** captures the shared `a7`, pointing to a new **Branch Node**. From the Branch, nibble `1` leads to a Leaf for suffix `11355` / account_1, and nibble `7` leads to a Leaf for suffix `7d337` / account_2.

![Step 2: Extension(a7) → Branch → two Leaf nodes at slots 1 and 7](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2012.png)

**Step 3 — Insert `a7f9365`:**
Also shares `a7`. We add a Leaf for suffix `f9365` / account_3 at slot `f` of the existing Branch Node.

![Step 3: Branch now has three Leaf nodes at slots 1, 7, f](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2013.png)

**Step 4 — Insert `a77d397`:**
This key shares `a7` + `7d3` with the key at slot `7`. That Leaf must split: slot `7` of the first Branch now points to a new **Extension Node** for `7d3`, which leads to a second **Branch Node** routing on nibble `3` (→ Leaf `37` / account_2) and nibble `9` (→ Leaf `97` / account_4).

![Step 4: Nested Extension and Branch for keys sharing prefix 7d3](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2014.png)

**Final state:**

![Final MPT after all four insertions](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2016.png)

**How node references propagate integrity:**

Starting from the leaves and working upward, each node is RLP-encoded and either inlined (if < 32 bytes) or stored in LevelDB under its keccak256 hash. Parent nodes store these references in their slots. The root node's reference is the **State Root** — a single 32-byte hash that commits to all four accounts.

Any change to any account value causes a chain reaction: the Leaf changes → its reference changes → the Branch slot changes → the Branch hash changes → the Extension changes → the **State Root** changes. Tamper-evidence propagates to the root automatically.

---

# From Theory to Implementation: Geth Internals

The logical Merkle Patricia Trie (MPT) described above is implemented in production clients with additional layers for performance and persistence. This section focuses on Geth’s approach. While our current series centers on EELS rather than Geth, it is still worthwhile to cover it here, as Geth implements the full end-to-end workflow related to the MPT — including storage in LevelDB — which EELS doesn't support for simplicity issue.

## In-Memory Representation: nodeFlag and Node Types

Geth represents MPT nodes in memory using Go interfaces. There are four concrete types:

| Geth type | Corresponds to |
|---|---|
| `shortNode` | Extension or Leaf node (2-element list) |
| `fullNode` | Branch node (17-element list) |
| `valueNode` | The raw data payload stored at a leaf |
| `hashNode` | A placeholder — a 32-byte hash for a node not yet loaded from disk |

The `hashNode` type is what enables lazy loading: it represents a node the client knows exists by its hash but has not yet fetched from LevelDB.

Every loaded node carries a `nodeFlag`:

```go
type nodeFlag struct {
    hash  hashNode  // cached keccak256 hash of this node (nil if dirty)
    gen   uint16    // cache generation counter for LRU eviction
    dirty bool      // true if modified and not yet written to disk
}
```

- **`hash`:** When a node is clean (unmodified since last commit), its keccak256 hash is cached here. Computing the State Root reuses these cached hashes rather than re-encoding and re-hashing the entire sub-tree.
- **`dirty`:** Set to `true` when a node is modified. Dirty nodes must be re-hashed and written to LevelDB on the next commit.
- **`gen`:** A generation counter used for LRU-style cache eviction. Old, clean, low-generation nodes can be evicted from memory and reloaded on demand.

---

## On-Disk Persistence: Content-Addressed Storage

Geth's default (hash-based) storage scheme writes MPT trie nodes to LevelDB as:

```
LevelDB key:   keccak256(RLP(node))
LevelDB value: RLP(node)
```

Because the key is derived from the content itself, looking up a node by its hash is a single O(1) database read. This scheme also makes deduplication natural: if two parts of the trie reference the same child node, they share a single LevelDB entry.

> **Note:** Geth v1.13+ also supports a **path-based storage scheme** where the LevelDB key encodes the trie path rather than the node hash. This alternative enables more efficient state pruning. When reading about Geth's LevelDB layout, always check which scheme is in use.

---

## Lazy Loading: Traversing Without Full State

Because nodes reference their children via node references (hashes or inline data), Geth does not need to load the entire state into memory. It uses lazy loading:

![Corrected lazy loading walkthrough — no h30, account struct decoded directly from leaf](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/lazy_loading_corrected.png)

Let's trace a lookup of key `a711355`:

1. **Root lookup:** The client queries LevelDB with the `stateRoot` hash → receives the Extension Node for `a7`.
2. **Extension Node:** Nibbles `a`, `7` are consumed. The `nodeReference` field either inlines the Branch Node or provides its hash → LevelDB is queried if needed.
3. **Branch Node:** Nibble `1` is consumed → slot `1` contains a node reference to the Leaf.
4. **Leaf Node:** The remaining path `11355` matches. The `value` field is `RLP([nonce, balance, storageRoot, codeHash])`. Decoding this directly yields the balance of 45 ETH.

There is no "hash of the value" step. The account data is stored directly inside the Leaf Node's value field as an RLP-encoded struct. The only hash indirection *within* an account is the `codeHash` field, which points to contract bytecode stored separately — and this applies only to contract accounts, not EOAs.

**Integrity:** If account_1's balance changes from 45 ETH to 46 ETH, the Leaf's RLP encoding changes → its hash changes → the Branch slot changes → the Branch hash changes → the Extension hash changes → a new **State Root** is produced.

**Efficiency:** Reading a single account requires only O(depth) LevelDB lookups — not a full scan of millions of accounts.

---

## What Is Actually Stored in LevelDB?

LevelDB in Geth is a general-purpose key-value store used for many different data types, each with its own key format. Understanding this prevents the misconception that "all LevelDB keys are keccak256 hashes."

| Data Type | LevelDB Key Format | Notes |
|---|---|---|
| MPT trie nodes (hash scheme) | `keccak256(RLP(node))` | The case described throughout this article |
| MPT trie nodes (path scheme) | encoded trie path | Geth v1.13+ opt-in |
| Block headers | `"h" + blockNum_BE8 + blockHash` | Namespaced by prefix byte |
| Block bodies | `"b" + blockNum_BE8 + blockHash` | Different prefix |
| Canonical block hash | `"h" + blockNum_BE8 + "n"` | Single-byte suffix |
| Contract bytecode | `keccak256(bytecode)` | Stored separately from the trie |

**What leaf values look like in practice:**

- **State trie leaf value:** `RLP([nonce, balance, storageRoot, codeHash])` — all four account fields encoded directly. No further hash indirection for the value itself.
- **Storage trie leaf value:** `RLP(slot_value)` — the storage slot integer, encoded directly.
- **`storageRoot`** in an account: the root hash of that contract's own storage trie. It is a hash, but a hash of a *trie root*, not of a value blob.
- **`codeHash`:** `keccak256(bytecode)`. The bytecode is a separate LevelDB entry looked up on demand. This is the *only* value-level hash indirection in the account model.

---

# The Four MPTs of Ethereum

Ethereum manages **four distinct MPTs**, differentiated by their scope and lifespan.

![Block header anchoring four trie roots; Storage Trie nested inside the State Trie](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2018.png)

## A. The State Trie (Global and Persistent)

The State Trie is a single, continuously evolving record of every account on the network.

- **Key:** `keccak256(account_address)` — the 20-byte address is hashed to produce a 32-byte (64-nibble) trie key.
- **Value (leaf):** `RLP([nonce, balance, storageRoot, codeHash])`
- **Lifespan:** Persistent. Each block modifies the trie and produces a new State Root, carrying over from genesis.

The **State Root** in a block header commits to the complete account state *after* all transactions in that block are applied.

## B. The Storage Trie (Per-Contract)

Every smart contract account has its own independent Storage Trie.

- **Key:** `keccak256(uint256(slot_index))` — the slot index is zero-padded to 32 bytes (big-endian) before hashing.
- **Value (leaf):** `RLP(slot_value)` — the 32-byte integer stored at that slot, encoded directly.
- **Linkage:** The `storageRoot` field in an account's state trie leaf contains the root hash of this contract's storage trie. Externally Owned Accounts (EOAs) have `storageRoot` equal to the hash of the empty trie.

Verifying a storage slot requires traversing two tries: the state trie (to reach the account and read `storageRoot`) then the storage trie (to reach the slot value).

## C. The Transaction Trie (Per-Block)

Each block has its own transaction trie, built from the ordered list of transactions included in that block.

- **Key:** `RLP(transaction_index)` — the integer index (0, 1, 2, …) is RLP-encoded before use as the trie key.
- **Value:** The RLP encoding of the full transaction.
- **Lifespan:** Permanent but frozen. Once a block is finalized, its transaction trie never changes.
- **Purpose:** The `transactionsRoot` enables Merkle proofs proving a specific transaction was included in a specific block.

## D. The Receipt Trie (Per-Block)

Mirrors the Transaction Trie in structure, storing the outcome of each transaction.

- **Key:** `RLP(transaction_index)` — same index-based keying as the transaction trie.
- **Value:** The RLP encoding of the transaction receipt: cumulative gas used, bloom filter, and the list of event logs emitted by contracts during execution.
- **Purpose:** The `receiptsRoot` enables light clients and dApps to verify event logs without downloading the full block body.

## The Block Header as Cryptographic Anchor

The block header ties together all four tries:

```
BlockHeader {
    parentHash,
    stateRoot,         ← root of the State Trie after this block
    transactionsRoot,  ← root of the Transaction Trie for this block
    receiptsRoot,      ← root of the Receipt Trie for this block
    ...
}
```

The Storage Trie is **nested inside** the State Trie via the `storageRoot` field. The full verification chain for a storage slot is:

```
Block Header
  → stateRoot
    → State Trie leaf  →  [nonce, balance, storageRoot, codeHash]
                                                 ↓
                               Storage Trie leaf  →  slot value
```

**This is what makes Light Clients practical.** A mobile device needs only block headers (a few hundred bytes each) and can request on-demand Merkle proofs for any account balance, storage slot, transaction, or log. The full node provides the chain of node references; the light client verifies them against its trusted header. No gigabytes of state required.

---

# Reference

1. Merkle, R. (1979). *Secrecy, Authentication, and Public Key Systems*. Stanford PhD Dissertation.
2. Ethereum Yellow Paper — https://ethereum.github.io/yellowpaper/paper.pdf
3. Ethereum.org — Patricia Merkle Trie — https://ethereum.org/developers/docs/data-structures-and-encoding/patricia-merkle-trie/
4. go-ethereum (Geth) source — trie package — https://github.com/ethereum/go-ethereum/tree/master/trie
5. EELS — ethereum/execution-specs — https://github.com/ethereum/execution-specs
6. Wikipedia — Merkle Tree — https://en.wikipedia.org/wiki/Merkle_tree
7. Wikipedia — Radix Tree — https://en.wikipedia.org/wiki/Radix_tree
8. Mastering Ethereum — https://masteringethereum.xyz/intro.html
