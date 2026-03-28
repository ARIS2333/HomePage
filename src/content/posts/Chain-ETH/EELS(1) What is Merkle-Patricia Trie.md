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
4. [MPT Breakdown](#mpt-breakdown)
    - [Understanding the "Key" in Ethereum](#understanding-the-key-in-ethereum)
    - [Nodes in MPT](#nodes-in-mpt)
    - [Key Encoding in the MPT](#key-encoding-in-the-mpt)
    - [An Example of MPT Evolution](#an-example-of-mpt-evolution)
5. [Optimization in Geth](#optimization-in-geth)
    - [The Memory Layer: Geth's nodeFlag](#the-memory-layer-geths-nodeflag)
    - [The Persistence Layer: Content-Addressed Storage](#the-persistence-layer-content-addressed-storage)
    - [Practical Workflow: Lazy Loading](#practical-workflow-lazy-loading)
6. [The Four MPTs of Ethereum](#the-four-mpts-of-ethereum)
    - [The State Trie (Global & Persistent)](#the-state-trie-global--persistent)
    - [The Storage Trie (Per-Contract)](#the-storage-trie-per-contract)
    - [The Transaction Trie (Per-Block)](#the-transaction-trie-per-block)
    - [The Receipt Trie (Per-Block)](#the-receipt-trie-per-block)
7. [Reference](#reference)

---

# Preface

This article is part of a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**. 

**What is EELS?**
EELS is a Python reference implementation of the Ethereum execution client, designed with a focus on readability and clarity. It serves as a programmer-friendly, up-to-date successor to the original Yellow Paper and is the primary tool for prototyping new Ethereum Improvement Proposals (EIPs). EELS provides complete protocol snapshots at each fork and rendered diffs between them.

**Scope and Implementation**
Note that EELS does not implement the JSON-RPC API or P2P networking. To validate blocks, it requires an external RPC provider to fetch data, which EELS then processes and stores in a local database.

**Version Reference**
The code analysis in this article is based on the **Osaka** fork: 
[ethereum/execution-specs (Osaka)](https://github.com/ethereum/execution-specs/tree/forks/amsterdam/src/ethereum/forks/osaka)

---

# Introduction

In Ethereum, the "State" is everything. Every account balance, every smart contract variable, and every transaction receipt must be stored in a way that is both **efficient to access** and **cryptographically immutable**. The Merkle-Patricia Trie is the "backbone" of this system. Before we can dive into the Python code of EELS to see how transactions are executed or how blocks are validated, we must first understand the "container" that holds all of Ethereum’s data. 

In this first installment of the series, we will peel back the layers of the MPT. We will discuss:

*   **The Evolution of the Trie:** How Ethereum combines the integrity of Merkle Trees with the retrieval speed of Patricia Tries.
*   **The Anatomy of a Node:** A deep dive into the four types of nodes that make the tree both compact and searchable.
*   **The Three Faces of a Key:** Why Ethereum keys transform between 3 encoding methods as they move from memory to disk.
*   **From Theory to Specification:** How these logical concepts are optimized in real-world clients like Geth.

**What you will learn:**
By the end of this article, you will understand how Ethereum can prove the existence of a single transaction among millions using only a few amount of data. This architectural understanding is the essential prerequisite for reading the EELS source code and understanding how the Ethereum "World State" actually functions.

---

# **Theoretical Background**

To understand the Merkle-Patricia Trie (MPT), we must first break down its constituent parts: the **Merkle Tree**, the **Trie**, and the **Patricia Trie**.


Broadly speaking, MPT is a data structure combining the advantages of both the Merkle Tree and the Patricia Trie: the former provides data integrity verification through upward hash propagation, while the latter enables efficient key-value retrieval via prefix path compression.

---

## Merkle Tree

Invented by Ralph Merkle in 1988, Merkle Trees are a foundational cryptographic data structure. They allow for the efficient verification of large datasets without requiring a user to download or process the entire database.

### **How it Works**

In a Merkle Tree, data blocks (such as transactions) are grouped at the bottom as "leaves." Each leaf is hashed, and pairs of these hashes are hashed together to form a parent node. This process continues up the tree until a single hash remains at the top: the **Merkle Root**.

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image.png)

This structure creates a "digital fingerprint" of the entire dataset. Because of the nature of cryptographic hashing:

- **Integrity:** Any change to a single data block (e.g., D_1) changes its hash (N_1), which changes its parent’s hash (N_4), eventually resulting in a completely different **Root**.
- **Efficiency:** To locate a modification or verify a specific piece of data, we only need to follow the path from the root down to the leaf. In a tree with n elements, this takes only **O(log n)** time.

For example, once it is found that the value of a node such as Root has changed, along the path Root --> N4 --> N1, the actually modified data block D1 can be quickly located in at most O(lgN) time.

### **Typical Applications**

- **Efficient Integrity Checks:**
    
    Instead of comparing two massive databases bit-by-bit, systems can simply compare their Merkle Roots. If the roots match, the data is identical.
    
- **Proving Membership:**
    
    A user can prove that a specific transaction exists in a block without downloading the whole block. By providing the transaction's Merkle Path (the hashes of its "sibling" nodes at each level), a light client can re-calculate the root and see if it matches the official block header. This Merkle Path is called a **Merkle Proof**.
    
- **Non-Existence Proofs:**
    
    By sorting the data and using "null" values for empty leaves, a Merkle Tree can also prove that a specific piece of data is **not** included in the set.
    

### **Advantage**

- **Fast Rehashing (Incremental Updates):**
    
    If one piece of data changes, you do not need to re-hash the entire tree. You only need to recalculate the hashes for the nodes along the path from that specific leaf to the root.
    
- **Enabling "Light Nodes":**
    
    Merkle Trees allow low-power devices (like smartphones) to participate in the network. A **Light Node** only stores the 80-byte block headers containing the Merkle Root. It relies on full nodes to provide the necessary "Merkle Paths" to verify specific transactions on demand.
    
    ![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%201.png)
    

### **Disadvantage**

- **Storage Overhead:**
    
    While efficient for verification, Merkle Trees require storing all the intermediate hashes (internal nodes). This increases the total storage footprint compared to a simple flat list of data.
    
- **Static Structure:**
    
    Standard Merkle Trees are typically "static." If you want to add or remove an element without just updating an existing one, the tree may need to be entirely restructured, which can be computationally expensive. (This is exactly where the **Trie** structure helps Ethereum).
    

---

## Trie

The terms **Tree** and **Trie** are often confused, but they represent fundamentally different concepts. In a standard **Tree**, nodes are arranged based on the logical structure of the data, and the relationship between nodes is independent of a "key." In contrast, the structure of a **Trie** (derived from "re**trie**val") is determined entirely by the **content of the key**.

In a Trie, the path from the root to a leaf represents the key itself. Its primary purpose is to enable rapid data location through prefix matching.

### How it Works

A Trie stores keys by breaking them down into individual characters or symbols. Each step down the tree corresponds to one character in the key.

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%202.png)

As shown in the above figure, multiple strings can share the same path if they share a common prefix. For example, the words "Ball" and "Bat" both branch off the "B" → "a" path. The $ symbol acts as a "terminator," indicating that a valid string ends at that specific node.

In a technical implementation (such as storing English words), each node is typically an array of pointers. For the English alphabet, a node might contain 27 slots: indices 0–25 represent the characters 'a' through 'z', and the 26th slot serves as a marker for the end of a string.

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%203.png)

As seen in the figure, while this allows for direct indexing, it often results in many "NULL" pointers, as not every node will have a child for every possible letter.

### Advantages

- **High Efficiency for Prefix Queries:**
Unlike a Hash Table, which requires a full scan (O(n)) to find all keys starting with a specific prefix (e.g., "Ba"), a Trie only needs to traverse the path to the "Ba" node. From there, it simply explores the sub-tree. This makes it ideal for autocomplete and "starts with" searches.
- **No Hash Collisions:**
Because each key has a unique path defined by its characters, Tries do not suffer from the collision issues found in Hash Tables, ensuring deterministic lookup paths.

### Disadvantages

- **Lookup Latency and I/O Overhead:**
While a Hash Table offers O(1) average lookup time, a Trie requires O(m) time, where m is the length of the key. Furthermore, in a database context, every character transition may represent a separate disk I/O operation, which can be significantly slower than a single direct lookup.
- **Space Inefficiency:**
If a key is very long and does not share a prefix with any other keys, the Trie must still create a unique node for every single character in that string. This creates a chain of "single-child" nodes that consume significant memory and storage without branching.

---

## Patricia Trie

The space inefficiency of the standard Trie is a major hurdle for blockchain state storage. To solve this, we use the **Patricia Trie**, which "compresses" these long, unbranching paths into single nodes to save space and reduce the number of I/O steps.

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%204.png)

---

### Combining the Best of Both Worlds

Before we dive into the **Merkle-Patricia Trie**, let’s summarize the core strengths we inherit from the two structures discussed:

| **From the Merkle Tree (Security)** | **From the Patricia Trie (Efficiency)** |
| --- | --- |
| **Hierarchical Hashing:** Every node stores a hash of its children, ensuring the Root Hash represents the entire dataset. | **Prefix-Based Paths:** Data is organized by keys, enabling efficient insertion, deletion, and prefix-based lookups. |
| **Tamper-Evidence:** Any change to a leaf node propagates upward, immediately invalidating the Root Hash. | **Path Compression:** Long, non-branching paths are compressed into single nodes, drastically reducing the tree's depth and storage overhead. |
| **Light Client Verification:** Allows "Light Nodes" to verify the existence of specific data without downloading the entire state. | **Deterministic Structure:** For any given set of key-value pairs, there is exactly one unique Trie representation, avoiding the unpredictability of hash tables. |

---

# MPT Breakdown

The core logic of the Merkle-Patricia Trie (MPT) is beautifully simple: **Use the Trie structure to organize data, and use the Merkle Tree's hashing mechanism to secure it.**

In an MPT, the **skeleton is a Trie**—you follow a path based on a key to reach your data. However, **every node is also a Merkle node**—it contains the cryptographic hash of its children. This fusion allows Ethereum to manage a massive, constantly changing state (millions of accounts and balances) while providing a single "State Root" hash that guarantees the integrity of every single piece of data in the network.

## Understanding the "Key" in Ethereum

Previously we discussed the trie using English words as keys. In Ethereum, a key is typically the `keccak256` hash of an entity, such as an account address or a smart contract storage slot. This hash is represented as a hexadecimal string, e.g.

```
**a711355**
```

In practice this should be 32 bytes, but let's use this short one for simplicity. The MPT processes this key by breaking it down into **nibbles** (a nibble is half a byte, representing a single hexadecimal character from `0` to `f`). The path you take through the MPT is determined exactly by the sequence of nibbles in the key.

---

## Nodes in MPT

To optimize storage and avoid the "sparse node" problem of standard Tries, Ethereum's MPT utilizes four distinct types of nodes.

### 1. Null Node

The simplest node. It represents an empty tree or an empty branch. In the code, it is simply represented as an empty string.

### 2. Branch Node

A Branch Node is used when paths diverge. Because keys are made of hex characters (`0`-`f`), there are 16 possible directions a path can take at any given fork. Therefore, a Branch Node contains **17 items: `[v0, v1, ..., v15, value]`**.

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%205.png)

Think of a Branch Node as a major intersection:

- **Slots 0 to 15:** These correspond to the 16 possible hex nibbles. Each slot is either `null` (if no keys go down that path) or contains the cryptographic hash pointing to the next child node. For example, if the current nibble is `a`, you look at slot 10 (which is `a` in hex) to search for the next node.
- **Slot 16:** Occasionally, a key's path ends *exactly* at this Branch Node. If so, the value for that key is stored in this final slot.

### 3. Leaf Node

A Leaf Node represents the end of a path.

In a standard Trie, if you have a long key but no other keys share its path, you would have to create a long chain of empty branch nodes just to store it. The MPT optimizes this by using a Leaf Node, which contains **2 items: `[encodedPath, value]`**.

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%206.png)

As shown in the figure, when a path reaches a point where **there are no more branches**, the MPT bundles the remaining nibbles together.

- **`encodedPath`**: The remaining portion of the key (e.g., `"11355"`).
- **`value`**: The actual target data (e.g., the account balance, nonce, etc.).

By packing these together, the MPT saves significant storage space.

### 4. Extension Node

An Extension Node is another compression mechanism, but it is used for *shared prefixes* rather than endpoints. It also contains **2 items: `[encodedPath, hash]`**.

When multiple keys share a long common sequence of nibbles before they finally branch off, creating a Branch Node for every single shared character is wasteful. Instead, an Extension Node compresses that shared path.

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%207.png)

- **`encodedPath`**: The shared prefix common to multiple keys (e.g., `"a7f"`).
- **`hash`**: A cryptographic pointer to the next node (usually a Branch Node) where the paths finally split. The system uses this hash to look up the next node in the underlying database (like LevelDB).

Because both Leaf and Extension nodes look structurally identical (both contain 2 items: `[encodedPath, target]`), how does the Ethereum protocol differentiate them?

The answer lies in a special **prefix flag bit** hidden inside the `encodedPath`.

- If the flag indicates an **Extension Node**, the system knows the second field is a *hash* pointing to another node, meaning the journey continues.
- If the flag indicates a **Leaf Node**, the system knows the second field is the *actual value*, meaning the path has terminated.

We will discuss this prefix flag in the next section.

---

To conclude this section on the MPT nodes, let’s put all the things together.

![Gemini_Generated_Image_347jbs347jbs347j.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/Gemini_Generated_Image_347jbs347jbs347j.png)

| **Node Type** | **Context / Usage** | **Contents** |
| --- | --- | --- |
| **Extension** | When multiple keys share a common prefix; used to "compress" the path. | Shared prefix + Hash of the next node. |
| **Branch** | When paths diverge (a fork in the road). | 16 slots (0–f) for children + 1 slot for an optional value. |
| **Leaf** | When the path no longer branches and reaches its conclusion. | Remaining unique path + Actual data (value). |
| **Null** | When a specific path or slot is empty. | An empty string (null). |

Imagine we need to store three accounts in the Ethereum state with the following keys:

1. a711355
2. a77d337
3. a7f9365

the MPT organizes them as follows:

**1. The Extension Node:**

All three keys start with **a7**. To save space, we don't create two separate levels for 'a' and '7'. Instead, we create a single **Extension Node** that stores the shared prefix a7 and points to the next node.

**2. The Branch Node:**

Immediately after a7, the keys diverge at the third nibble: **1**, **7**, and **f**. We use a **Branch Node** to handle this fork.

- Slot 1 points to the leaf for the first key.
- Slot 7 points to the leaf for the second key.
- Slot f points to the leaf for the third key.

**3. The Leaf Nodes:**

Each path ends in a **Leaf Node** containing the unique "suffix" of the key and the actual account data (balance, nonce, etc.).

- Key 1 leaf stores: 11355 + Value 1
- Key 2 leaf stores: d337 + Value 2
- Key 3 leaf stores: 9365 + Value 3

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%208.png)

As shown in the figures, the "pointers" connecting these nodes are not simple memory addresses. They are **RLP-encoded Keccak256 hashes** of the child nodes.

1. We hash the **Leaf Nodes**.
2. Those hashes are stored in the **Branch Node** slots.
3. We hash the **Branch Node**, and that hash is stored in the **Extension Node**.
4. The hash of the Extension Node (the root of this sub-tree) eventually contributes to the **State Root**.

This ensures that if a single balance inside a Leaf Node is tampered with, the hash of that leaf changes, which changes the Branch Node, which changes the Extension Node, and finally results in a completely different **State Root**. This is how Ethereum achieves both efficient lookup and cryptographic integrity in one structure.

---

## Key Encoding in the MPT

In the node definitions above, we referred to fields like encodedPath (e.g., the "a7" in an Extension node or the "11355" in a Leaf node). However, these paths are not stored as simple text strings. 

In Ethereum, a key does not maintain a single static form. Instead, it transitions between three different encoding formats depending on where it is being used: the **External Interface**, the **In-Memory Tree**, or the **On-Disk Database**. To manage these keys efficiently—moving from a user's account address to a searchable tree in memory, and finally to a persistent database—Ethereum's MPT transitions between three distinct encoding formats.

| Context | Encoding Type | Purpose |
| --- | --- | --- |
| **External Interface** | **Raw Encoding** | Standard byte arrays (e.g., user input "cat"). |
| **Memory** | **Hex Encoding** | Nibble-based paths used to traverse the tree. |
| **Disk (LevelDB)** | **Hex-Prefix (HP)** | Compact byte storage that identifies node types. |

---

### Raw Encoding

Raw encoding is the "natural" state of a key—the raw bytes of the string. This is the default format for the MPT’s external APIs.

- **Example:** The key `"cat"` is represented as its ASCII bytes: `['c', 'a', 't']`, which is `[0x63, 0x61, 0x74]`.

### Hex Encoding

When the tree is active in memory, we need to traverse it. As we learned earlier, a **Branch Node** has 16 slots (0–f). This means the tree moves one **nibble** (4 bits) at a time, not one full byte (8 bits) at a time.

To facilitate this, we convert the Raw bytes into a sequence of nibbles:

- `'c' (0x63)` becomes `6, 3`
- `'a' (0x61)` becomes `6, 1`
- `'t' (0x74)` becomes `7, 4`

**The Terminator (16):**
The tree also needs a way to distinguish between a **Leaf Node** (the end of a path) and an **Extension Node** (a shortcut to another branch) while in memory. We do this by appending a "terminator" value of `16` (which is outside the hex range of 0–15) to the end of Leaf Node keys.

- **Leaf Node path:** `[6, 3, 6, 1, 7, 4, 16]` (Terminator present)
- **Extension Node path:** `[6, 3, 6, 1, 7, 4]` (No terminator)

### Hex-Prefix (HP) Encoding

When it is time to save the data to the disk (LevelDB), Hex encoding faces two physical limitations:

1. **Byte-Addressability:** Disks store data in bytes. We cannot store a single 4-bit nibble alone; we must pair them up into 8-bit bytes.
2. **The Terminator Problem:** We cannot store the value `16` as a nibble. We need a more space-efficient way to tell the database whether a node is a Leaf or an Extension.

**HP Encoding** solves both by adding a **prefix nibble** to the front of the key. This prefix uses two bits of information:

- **Bit 1:** `1` for Leaf, `0` for Extension.
- **Bit 0:** `1` if the remaining path length is odd, `0` if it is even.

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%209.png)

Here we use padding to make sure that, the **total number of nibbles must be even** when ****we eventually combine nibbles into bytes.

- If the path length is **Odd**: The 1-nibble prefix acts as the "odd" piece out. Total length = `1 (prefix) + Odd = Even`. .
- If the path length is **Even**: Adding a 1-nibble prefix would make the total length odd (`1 + Even = Odd`). To fix this, we add a **padding nibble (`0`)** after the prefix. Total length = `1 (prefix) + 1 (padding) + Even = Even`.

---

At the end of the encoding section, let’s look at how the key for "cat" moves through the system:

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2010.png)

1. **Raw (Interface):** `[0x63, 0x61, 0x74]` ("cat")
2. **Hex (Memory):** We split it into nibbles and add `16` because it's a leaf in this example:
    
    `[6, 3, 6, 1, 7, 4, 16]`
    
3. **HP (Disk):**
    - **Step A:** Identify it as a **Leaf** with an **Even** remaining path (`6, 3, 6, 1, 7, 4`).
    - **Step B:** According to the table, the prefix is `0x2`. Since the path is even, we add a padding `0`, making the full prefix `0x20`.
    - **Step C:** Prepend the prefix and pair the nibbles into bytes:
        
        `20 63 61 74`
        

In summary,

- **Raw → Hex:** Allows the MPT to navigate the tree **nibble-by-nibble**.
- **Hex → HP:** Allows the MPT to store the tree **compactly on disk** while maintaining the ability to instantly distinguish between Leaf and Extension nodes upon reading.

---

## **An Example of MPT Evolution**

To see how the MPT evolves, let’s trace the insertion of four account states into a "Simplified World State."

**The Dataset:**

| Key (Hex) | Value (Balance) |
| --- | --- |
| `a711355` | 45.0 ETH |
| `a77d337` | 1.00 WEI |
| `a7f9365` | 1.1 ETH |
| `a77d397` | 0.12 ETH |

The MPT grows dynamically as keys are added. Instead of a deep, static tree, it optimizes itself through splitting and compression:

1. **First Insertion (`a711355`):** Since there is only one key, the entire path is stored in a single **Leaf Node**.
    
    ![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2011.png)
    
2. **Second Insertion (`a77d337`):** This key shares the prefix `a7` with the first. The trie is reorganized: an **Extension Node** is created for `a7`, leading to a **Branch Node**. The remaining paths (`11355` and `7d337`) become two separate Leaf Nodes connected to slots `1` and `7` of that branch.
    
    ![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2012.png)
    
3. **Third Insertion (`a7f9365`):** This also shares the `a7` prefix. We simply add a new Leaf Node (`9365`) to slot `f` of the existing Branch Node.
    
    ![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2013.png)
    
4. **Fourth Insertion (`a77d397`):** This key shares a longer prefix (`a7` + `7d`) with the second key. This triggers a further split: slot `7` of the first branch now points to a new **Extension Node** (`d3`), which leads to another **Branch Node** to handle the final divergence between the nibbles `3` and `9`.
    
    ![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2014.png)
    
5. **Optional:** Things could end here, but the source code implementation of Geth needs to go one level deeper, and only the leaf nodes at the final level contain data.
    
    ![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2015.png)
    

Below figure shows The final state of the World State Trie after all the operations:

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2016.png)

---

## Optimization in Geth

While the Merkle-Patricia Trie is a logical data structure, its real-world performance in clients like Geth depends on how it is cached in memory and persisted on disk.

### **The Memory Layer: Geth's nodeFlag**

In the Geth client, every node currently loaded in memory carries a `nodeFlag` structure. This metadata manages the node's lifecycle and prevents unnecessary computations.

```
// nodeFlag contains caching-related metadata about a node.
type nodeFlag struct {
    hash  hashNode // cached hash of the node (may be nil)
    gen   uint16   // cache generation counter for eviction
    dirty bool     // whether the node has changes that must be written to disk
}
```

- **Cached Hash:** If a node is "clean" (unmodified), it stores its keccak256 hash here. When calculating the Root Hash of the entire tree, the system simply reads this value instead of re-hashing the entire sub-tree, providing a massive performance boost.
- **Dirty Flag:** When a node is modified (e.g., an account balance changes), it is marked as dirty. This signals that the node's hash is now invalid and must be recalculated and eventually written to the database.
- **Cache Generation (gen):** To prevent the state from consuming all available RAM, Geth uses this counter to "evict" old, unmodified nodes from memory, keeping only the most active parts of the state in the cache.

### **The Persistence Layer: Content-Addressed Storage**

The MPT exists logically as a tree, but it is stored physically in a flat **Key-Value database (LevelDB)**. The "Merkle" magic happens in how these are mapped:

- **Database Key:** The `keccak256` hash of the node's RLP-encoded data.
- **Database Value:** The actual RLP-encoded data of the node.

This is known as **Content-Addressed Storage**. Because every node is referenced by its hash, the **State Root** (the hash of the root node) serves as a secure entry point.

### Practical Workflow: Lazy Loading

Because nodes reference their children via hashes, the client does not need to load the entire multi-gigabyte state into memory. Instead, it performs **Lazy Loading**—only "unpacking" the nodes along the specific path it needs to access.

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2017.png)

As shown in the figure above, if we want to access the data for a key a711355, the process works as follows:

1. **Start at the Root:** The client looks up the hash h00 in LevelDB to retrieve the **Extension Node** (a7).
2. **Follow the Hash:** The Extension Node contains the hash h10. The client queries the database for h10 to retrieve the **Branch Node**.
3. **Navigate the Branch:** Looking at the Branch Node, the client sees that for nibble 1, the next hash is h20.
4. **Reach the Leaf:** The client queries h20, which returns the **Leaf Node** containing the remaining path and the final value (45).
5. **Final Result:** The value 45 itself is often stored as a hash (h30) if it is a large data blob, completing the chain.

This process demonstrates the two desired advantages of MPT:

- **Integrity:** Any change to the data 45 would change h30, which changes h20, eventually changing the h00 (Root Hash).
- **Efficiency:** To verify or read a single account, you only need to perform a few disk lookups (e.g., h00 → h10 → h20 → h30), rather than processing the millions of other accounts in the Ethereum state.

---

# The Four MPTs of Ethereum

To conclude our deep dive into the MPT, we look at how Ethereum applies this structure in practice. Ethereum does not use just one trie; it manages **four distinct MPTs** to track every aspect of the network. These are categorized by their lifespan and scope.

**A. The State Trie (Global & Persistent)**

The State Trie is the most important structure in Ethereum. It acts as a single, global "Golden Record" of every account on the network.

- **Path (Key):** `keccak256(ethereum_address)`
- **Value:** An RLP-encoded account structure containing: `[nonce, balance, storageRoot, codeHash]`.
- **Lifespan:** Persistent. It is updated after every block but carries over from the genesis block to the present.

**B. The Storage Trie (Per-Contract)**

Every smart contract has its own independent Storage Trie to hold its internal variables (state variables).

- **Path (Key):** `keccak256(storage_slot_index)`
- **Value:** The RLP-encoded data stored at that slot.
- **Linkage:** The **root hash** of this trie is stored within the corresponding account’s entry in the **State Trie**. Externally Owned Accounts (EOAs) have an empty storage root.

**C. The Transaction Trie (Per-Block)**

Every block has its own unique Transaction Trie containing all transactions included in that specific block.

- **Path (Key):** The index of the transaction in the block (e.g., `0`, `1`, `2`).
- **Lifespan:** Temporary. Once a block is sealed, this trie never changes.
- **Purpose:** Allows for "Merkle Proofs" to prove a specific transaction was included in a block.

**D. The Receipt Trie (Per-Block)**

Like the Transaction Trie, every block has its own Receipt Trie. It records the *outcome* of the transactions.

- **Path (Key):** The index of the transaction.
- **Value:** Includes the post-transaction state, gas used, and **Event Logs** (which are critical for dApps to "listen" to contract activity).

![image.png](EELS(1)%20What%20is%20Merkle-Patricia%20Trie/image%2018.png)

The **Block Header** acts as the cryptographic anchor for the entire network. It contains three specific Merkle roots that allow any node to verify the integrity of the blockchain:

1. **State Root:** The root of the State Trie after all transactions in the block are applied.
2. **Transactions Root:** The root of the Transaction Trie for this block.
3. **Receipts Root:** The root of the Receipt Trie for this block.

Note that the **Storage Trie** is nested inside the **State Trie**. This means to verify a single piece of contract data (e.g., a token balance), you verify the path:
`Block Header → State Root → Account Leaf → Storage Root → Data Slot`.

This architecture is what makes **Light Clients** possible. A smartphone does not need to store the hundreds of gigabytes of the "World State." It only needs to download the small **Block Headers**.

If the smartphone needs to verify your account balance, it asks a full node for a **Merkle Proof**. The full node provides the path of hashes from your account up to the **State Root**. The light client hashes them together; if the result matches the State Root in its trusted Block Header, the balance is mathematically proven to be correct without the client ever seeing the rest of the database.

---

# Reference

1. [https://en.wikipedia.org/wiki/Merkle_tree](https://en.wikipedia.org/wiki/Merkle_tree)
2. [https://en.wikipedia.org/wiki/Radix_tree](https://en.wikipedia.org/wiki/Radix_tree)
3. [https://ethereum.org/developers/docs/data-structures-and-encoding/patricia-merkle-trie](https://ethereum.org/developers/docs/data-structures-and-encoding/patricia-merkle-trie/)
4. [https://zhuanlan.zhihu.com/p/46702178](https://zhuanlan.zhihu.com/p/46702178)
5. [https://masteringethereum.xyz/intro.html](https://masteringethereum.xyz/intro.html)
6. [https://yeasy.gitbook.io/blockchain_guide/05_crypto/merkle_trie](https://yeasy.gitbook.io/blockchain_guide/05_crypto/merkle_trie)