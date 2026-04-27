---
title: EELS(10) Blob Mechanism
published: 2026-04-07
pinned: false
description: Explain how Ethereum uses blob for scaling
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

1. [**Preface**](https://www.notion.so/EELS-10-Blob-Mechanism-33e7d90fb9ee801cb68fc9617d0589da?pvs=21)
2. [**Introduction**](https://www.notion.so/EELS-10-Blob-Mechanism-33e7d90fb9ee801cb68fc9617d0589da?pvs=21)
3. [**The Problem Blobs Solve**](https://www.notion.so/EELS-10-Blob-Mechanism-33e7d90fb9ee801cb68fc9617d0589da?pvs=21)
4. [**Blob Anatomy: Structure and Size**](https://www.notion.so/EELS-10-Blob-Mechanism-33e7d90fb9ee801cb68fc9617d0589da?pvs=21)
5. [**KZG Commitments: The Math Made Intuitive**](https://www.notion.so/EELS-10-Blob-Mechanism-33e7d90fb9ee801cb68fc9617d0589da?pvs=21)
6. [**Sharding and the Path to Danksharding**](https://www.notion.so/EELS-10-Blob-Mechanism-33e7d90fb9ee801cb68fc9617d0589da?pvs=21)
7. [**The Blob Transaction (Type 0x03)**](https://www.notion.so/EELS-10-Blob-Mechanism-33e7d90fb9ee801cb68fc9617d0589da?pvs=21)
8. [**L2 Workflow: How Rollups Use Blobs**](https://www.notion.so/EELS-10-Blob-Mechanism-33e7d90fb9ee801cb68fc9617d0589da?pvs=21)
9. [**Mental Model Summary**](https://www.notion.so/EELS-10-Blob-Mechanism-33e7d90fb9ee801cb68fc9617d0589da?pvs=21)

---

## Preface

This article is part of a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**.

**What is EELS?** EELS is a Python reference implementation of the Ethereum execution client, designed with a focus on readability and clarity. It serves as a programmer-friendly, up-to-date successor to the original Yellow Paper and is the primary tool for prototyping new Ethereum Improvement Proposals (EIPs).

**Scope and Implementation** While the previous chapter focused on persistent state and the account model, this chapter explores the implementation of **EIP-4844 (Proto-Danksharding)**, which introduced a new way to handle transient data through "blobs."

**Version Reference** The code and logic analyzed in this article are based on the **Osaka** fork:
[ethereum/execution-specs (Osaka)](https://github.com/ethereum/execution-specs/tree/forks/amsterdam/src/ethereum/forks/osaka)

---

## Introduction

This chapter focuses on **EIP-4844**. We will analyze the blob data and how it remains cryptographically bound to the execution layer via KZG commitments. We will explore how these blobs are structured and how L2s utilize blobs.

---

## 1. The Problem Blobs Solve

Before getting into what a blob *is*, let's ground ourselves. L2 rollups (Optimism, Arbitrum, zkSync, etc.) derive their security from Ethereum L1 — they periodically publish their transaction data to L1 so that anyone can verify state or raise a fraud proof. They used to do this by stuffing that data into **calldata** on regular transactions.

The problem: calldata is expensive because it lives in transaction history **forever**, and every full node must store it permanently. But rollups only need the data to be available for a short window — long enough for fraud proofs (~7 days for optimistic rollups) or for ZK proof generation. Paying for permanent storage when you only need temporary availability is a terrible deal.

EIP-4844 introduces a new type of data carrier — the **blob** — that is cheap, temporary, and purpose-built for rollups.

---

## 2. What Does a Blob Actually Look Like?

A blob is fundamentally a large byte array. Concretely:

- `FIELD_ELEMENTS_PER_BLOB = 4096` field elements
- Each field element is `BYTES_PER_FIELD_ELEMENT = 32` bytes
- Total size: **4096 × 32 = 131,072 bytes ≈ 128 KB** per blob

Each field element is not an arbitrary 32-byte value — it must be a number strictly less than the **BLS modulus** (`BLS_MODULUS ≈ 5.24 × 10^76`). This is the prime that defines the finite field used in the KZG cryptography. You can think of the blob as an array of 4096 "slots," each holding a large integer within this field.

![image.png](EELS(10)%20Blob%20Mechanism/image.png)

The blob data itself is **not accessible by the EVM**. The EVM cannot read the 128 KB of raw blob content. What the EVM *can* access is the blob's **commitment** — a compact 48-byte fingerprint that cryptographically represents the entire blob. This leads us to the most important cryptographic primitive in EIP-4844.

---

## 3. KZG Commitments: The Math Made Intuitive

KZG stands for **Kate-Zaverucha-Goldberg**, the authors of the 2010 paper that introduced this polynomial commitment scheme. Understanding it requires a short detour through polynomials.

### The Core Idea: A Blob is a Polynomial

Treat the 4096 field elements of a blob as the **evaluations of a polynomial** at 4096 known points (roots of unity in the BLS12-381 field). There exists a unique polynomial `p(x)` of degree ≤ 4095 that passes through all of those points. The KZG scheme lets you commit to that polynomial with a single 48-byte value.

![image.png](EELS(10)%20Blob%20Mechanism/image%201.png)

### How the Commitment is Created

KZG uses an **elliptic curve** (BLS12-381) and a pre-computed **Structured Reference String (SRS)** — a set of elliptic curve points of the form `[τ⁰·G, τ¹·G, τ²·G, ..., τ⁴⁰⁹⁵·G]` where `τ` ("tau") is a secret that was destroyed after the trusted setup ceremony. The commitment to polynomial `p(x) = a₀ + a₁x + a₂x² + ... + a₄₀₉₅x⁴⁰⁹⁵` is simply:

```
C = a₀·[G] + a₁·[τG] + a₂·[τ²G] + ... + a₄₀₉₅·[τ⁴⁰⁹⁵G]
```

This is a single point on the BLS12-381 G1 curve — 48 bytes. Because `τ` is unknown, you cannot work backwards from `C` to `p(x)`, making it a **binding** commitment.

### The Point Evaluation Precompile

Here's where things get practical. EIP-4844 adds a new precompile at address `0x0A` called `point_evaluation_precompile`. It verifies the claim: *"at point z, polynomial p evaluates to value y"* — i.e., `p(z) = y`.

You give it:

- The **versioned hash** of the blob (derived from the KZG commitment)
- A point `z` (which field element index to evaluate at)
- A claimed value `y` (what the blob data is at that index)
- The **KZG commitment** `C` (48 bytes)
- A **KZG proof** `π` (48 bytes — a G1 point proving consistency)

The precompile uses a **pairing check** on BLS12-381 to verify all of this in O(1) time without ever touching the raw blob data. This is the bridge between the blob world and the EVM world — smart contracts can verify specific pieces of blob data through proofs, without the EVM needing the full 128 KB.

The versioned hash itself is derived as:

```python
def kzg_to_versioned_hash(commitment: KZGCommitment) -> VersionedHash:
    return VERSIONED_HASH_VERSION_KZG + sha256(commitment)[1:]
    # = 0x01 ++ sha256(C)[1:]  →  32 bytes total
```

The `0x01` prefix is a version byte — future cryptographic schemes (post-quantum, etc.) can use a different version byte, keeping the system forward-compatible.

---

## 4. Sharding and Its Relationship to Blobs

### What is Sharding?

Ethereum's long-term scalability roadmap calls for **data sharding** (specifically "Danksharding" after researcher Dankrad Feist). The idea is to split the network's data responsibilities across many parallel "shards," so no single node needs to download and store all the data — only a random subset, while cryptographic techniques (Data Availability Sampling, DAS) provide guarantees that the full data exists somewhere on the network.

Full Danksharding would allow **~32 MB of blob data per block** and use DAS so light clients can verify data availability without downloading it all.

### Where EIP-4844 (Proto-Danksharding) Fits

EIP-4844 is the deliberate *first step* toward that vision. It introduces the exact same transaction format and cryptographic machinery (KZG) that full Danksharding will use — but without the actual sharding. Instead, blobs are simply carried as **sidecars** alongside beacon blocks and downloaded fully by all consensus nodes.

![image.png](EELS(10)%20Blob%20Mechanism/image%202.png)

The key design insight here: because EIP-4844 uses the exact same transaction type and KZG machinery as full Danksharding will require, **rollups only have to upgrade once**. When full Danksharding eventually ships, they don't change their code — validators just start doing DAS instead of full downloads behind the scenes.

---

## 5. The Blob Transaction Format (Type 0x03)

A blob transaction is a new EIP-2718 transaction type (`0x03`). Let's look at what it actually contains and how its two network representations work.

![image.png](EELS(10)%20Blob%20Mechanism/55fdaa89-3345-4b61-b597-0738d5438e1f.png)

### The Separate Blob Gas Market

Blobs introduce a second, independent gas market that mirrors EIP-1559. The block header now carries `blob_gas_used` and `excess_blob_gas`. Each blob costs `GAS_PER_BLOB = 2¹⁷ = 131072` blob gas. The base fee per blob gas adjusts exponentially based on how much excess blob gas has accumulated:

```python
base_fee_per_blob_gas = MIN_BASE_FEE_PER_BLOB_GAS * e^(excess_blob_gas / BLOB_BASE_FEE_UPDATE_FRACTION)
```

This means blob fees are entirely decoupled from regular ETH gas fees — heavy calldata usage doesn't make blobs expensive and vice versa. Blob fees are **burned** like EIP-1559 base fees.

---

## 6. How L2s Use Blobs: The Full Picture

Now let's put it all together. When an L2 (say Optimism) wants to commit a batch of transactions to L1, here is exactly what happens:

![image.png](EELS(10)%20Blob%20Mechanism/d4e600c4-5ce7-4e85-9a9a-73278af4c5ca.png)

Let's walk through each step carefully.

**Step 1 — Compress the batch.** The L2 sequencer takes hundreds or thousands of L2 transactions, compresses them (often with zlib/brotli), and packs them into the 128 KB blob layout — filling the 4096 field element slots. If more than one blob is needed, up to 6 can be attached to a single transaction.

**Step 2 — Compute KZG commitment locally.** The sequencer's client (e.g. op-node) uses a KZG library to compute the commitment `C`, the proof `π`, and derives the `versioned_hash = 0x01 || sha256(C)[1:]`. This is all done off-chain before broadcasting.

**Step 3 — Build the Type 0x03 transaction.** The transaction's `to` field points to the **L1 inbox contract** — a known L1 smart contract address maintained by the L2 protocol. The transaction payload includes `blob_versioned_hashes` — the array of 32-byte versioned hashes — but **not** the raw blob data. The blob data sits alongside in the network wrapper.

**Step 4 — Broadcast and include.** The transaction is announced (not pushed, to avoid DoS) via `NewPooledTransactionHashes`. Once included in an L1 block, the **EVM executes the call to the inbox contract**. Inside that contract, the `BLOBHASH(0)` opcode (opcode `0x49`) reads `blob_versioned_hashes[0]` from the transaction — returning the 32-byte versioned hash. The contract stores this hash on-chain as an anchor point. **This is the only blob-related data that becomes permanent L1 state.**

**Consensus layer — sidecar path.** Simultaneously, the beacon node gossips the blob sidecar (raw blob + commitment + proof) across the P2P network. Every consensus node downloads the full sidecars and runs `verify_blob_kzg_proof_batch()` to confirm the blob data matches the commitments, which match the versioned hashes in the execution block. The blobs persist for `~4096 epochs ≈ 18 days`, then are pruned.

**Fraud proofs / ZK proofs.** When an optimistic rollup verifier needs to challenge a state transition, or a ZK rollup needs to prove equivalence between its proof and the blob data, they call the `point_evaluation_precompile` at `0x0A`. They provide the versioned hash (retrieved from the inbox contract), the commitment, a claimed value `y` at point `z`, and a KZG proof. The precompile verifies `p(z) = y` using a pairing check — confirming that a specific 32-byte chunk of the original blob data is what the challenger claims, without the verifier needing the full 128 KB.

---

## 7. Everything Together: A Mental Model

![image.png](EELS(10)%20Blob%20Mechanism/image%203.png)

---

## Summary

Here's the full conceptual stack in one pass:

**Blob** — a 128 KB array of 4096 field elements in the BLS12-381 prime field. It's a large-but-cheap data carrier. The EVM cannot read it directly; it lives transiently in the consensus layer for ~18 days.

**KZG commitment** — a single 48-byte elliptic curve point that uniquely and cryptographically binds the entire blob's polynomial representation. You can prove any individual "read" from the blob (`p(z) = y`) with a 48-byte proof, verifiable against the commitment without touching the raw data. The commitment's sha256 hash (prefixed `0x01`) becomes the **versioned hash**, which is what gets recorded permanently on L1.

**Sharding relationship** — EIP-4844 (Proto-Danksharding) is a deliberate partial implementation of full Danksharding. It uses the exact final transaction format and KZG cryptography, but all consensus nodes still download blobs fully. Full Danksharding will replace this with Data Availability Sampling so validators only check random samples. Rollups will not need to change their code when that transition happens.

**L2 posting on L1** — the L2 sequencer compresses its tx batch into blob data, computes a KZG commitment locally, and posts a Type 0x03 transaction whose `to` field is the L2's L1 inbox contract. The raw blob rides as a beacon sidecar; only the versioned hash enters the EVM. The inbox contract reads the versioned hash via `BLOBHASH` and stores it as an on-chain anchor. Later, fraud proofs or ZK equivalence proofs can verify specific blob values against that anchor using the `point_evaluation_precompile` — no need to ever touch the raw 128 KB on-chain.