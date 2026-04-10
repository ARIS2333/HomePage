---
title: EELS(4) Cryptographic Encoding and Address Derivation
published: 2026-04-01
pinned: false
description: Encoding methods in Ethereum
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

1. [**Preface**](#preface)
2. [**Introduction**](#introduction)
3. [**1. Keccak-256 Hashing**](#1-keccak-256-hashing)
4. [**2. SHA-256 Hashing**](#2-sha-256-hashing)
5. [**3. ECDSA Signatures**](#3-ecdsa-signatures)
6. [**4. RLP Encoding**](#4-rlp-encoding)
7. [**5. Address Derivation**](#5-address-derivation)
  

## Preface

This article is part of a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**.

**What is EELS?** EELS is a Python reference implementation of the Ethereum execution client, designed with a focus on readability and clarity. It serves as a programmer-friendly, up-to-date successor to the original Yellow Paper and is the primary tool for prototyping new Ethereum Improvement Proposals (EIPs). EELS provides complete protocol snapshots at each fork and rendered diffs between them.

**Scope and Implementation** Note that EELS does not implement the JSON-RPC API or P2P networking. To validate blocks, it requires an external RPC provider to fetch data, which EELS then processes and stores in a local database.

**Version Reference** The code analysis in this article is based on the **Osaka** fork:
[ethereum/execution-specs (Osaka)](https://github.com/ethereum/execution-specs/tree/forks/amsterdam/src/ethereum/forks/osaka)

---

## Introduction

This chapter describes the cryptographic primitives and serialization formats used throughout the Ethereum Execution Layer Specification (EELS) for the Osaka fork. It covers hashing (Keccak-256 and SHA-256), elliptic curve signatures (secp256k1 and secp256r1), RLP encoding, and address derivation for EOAs and contract accounts.

---

## 1. Keccak-256 Hashing

Keccak-256 is the primary hash function in Ethereum. It is used for address derivation, code hashing, MPT node hashing, secured trie key hashing, and transaction hashing.

> **Note on naming**: Keccak-256 is *not* the same as SHA-3-256, despite sharing the same underlying sponge construction. SHA-3 was standardized with different padding than the original Keccak submission. Ethereum uses the original Keccak-256.
> 

### 1.1 Implementation

The hash function is defined in [`ethereum/crypto/hash.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/crypto/hash.py) and uses the `pycryptodome` library:

```python
# src/ethereum/crypto/hash.py

Hash32 = Bytes32
Hash64 = Bytes64

def keccak256(buffer: Bytes | bytearray) -> Hash32:
    """
    Computes the keccak256 hash of the input `buffer`.
    """
    k = keccak.new(digest_bits=256)
    return Hash32(k.update(buffer).digest())
```

The function accepts `Bytes` or `bytearray` and returns a `Hash32` (a 32-byte value). The underlying implementation uses `Crypto.Hash.keccak` from `pycryptodome`, not Python’s `hashlib.sha3_256`, which produces a different result.

### 1.2 Usage Across the Specification

Keccak-256 appears in virtually every layer of the specification:

| Context | Input |
| --- | --- |
| EOA address derivation | 64-byte uncompressed public key (prefix byte stripped) |
| Contract address (`CREATE`) | `rlp([sender, nonce])` |
| Contract address (`CREATE2`) | `0xff ++ sender ++ salt ++ keccak256(init_code)` |
| Code hash in `Account` | contract bytecode |
| Secured trie keys | address or storage slot |
| MPT node hashing | RLP-encoded node (when serialized length ≥ 32 bytes) |

---

## 2. SHA-256 Hashing

SHA-256 is not a general-purpose utility in the EELS crypto package. It is only exposed through the SHA-256 precompiled contract at address `0x02`, which is callable from the EVM.

### 2.1 The SHA-256 Precompile

The precompile is defined in [`ethereum/forks/osaka/vm/precompiled_contracts/sha256.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/sha256.py):

```python
# src/ethereum/forks/osaka/vm/precompiled_contracts/sha256.py

def sha256(evm: Evm) -> None:
    """
    Writes the sha256 hash to output.
    """
    data = evm.message.data

    # GAS
    word_count = ceil32(Uint(len(data))) // Uint(32)
    charge_gas(
        evm,
        GAS_PRECOMPILE_SHA256_BASE
        + GAS_PRECOMPILE_SHA256_PER_WORD * word_count,
    )

    # OPERATION
    evm.output = hashlib.sha256(data).digest()
```

The gas cost is `GAS_PRECOMPILE_SHA256_BASE + GAS_PRECOMPILE_SHA256_PER_WORD * ceil(len(data) / 32)`. Unlike Keccak-256, SHA-256 is not used internally by the protocol itself — its sole use in the specification is through this precompile, callable by smart contracts.

---

## 3. ECDSA Signatures

Ethereum uses two elliptic curves for cryptographic operations, both defined in [`ethereum/crypto/elliptic_curve.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/crypto/elliptic_curve.py):

- **secp256k1** — used for transaction signing and EOA address derivation.
- **secp256r1 (P-256)** — added in Osaka (EIP-7951) as a precompile for verifying signatures from secure enclaves and hardware attestation schemes.

### 3.1 secp256k1: Public Key Recovery

Transaction signing uses secp256k1. The EELS does not implement signing — it only implements **recovery**: given a message hash and a signature, it recovers the signer’s public key (and from it, their address).

The curve constants are:

```python
SECP256K1B = U256(7)
SECP256K1P = U256(0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F)
SECP256K1N = U256(0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141)
```

[**`secp256k1_recover`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/crypto/elliptic_curve.py) recovers a 64-byte uncompressed public key (without the `0x04` prefix) from a signature:

```python
def secp256k1_recover(r: U256, s: U256, v: U256, msg_hash: Hash32) -> Bytes:
    """
    Recovers the public key from a given signature.
    """
    is_square = pow(
        pow(r, U256(3), SECP256K1P) + SECP256K1B,
        (SECP256K1P - U256(1)) // U256(2),
        SECP256K1P,
    )

    if is_square != 1:
        raise InvalidSignatureError(
            "r is not the x-coordinate of a point on the secp256k1 curve"
        )

    r_bytes = r.to_be_bytes32()
    s_bytes = s.to_be_bytes32()

    signature = bytearray([0] * 65)
    signature[32 - len(r_bytes) : 32] = r_bytes
    signature[64 - len(s_bytes) : 64] = s_bytes
    signature[64] = v

    try:
        public_key = coincurve.PublicKey.from_signature_and_message(
            bytes(signature), msg_hash, hasher=None
        )
    except ValueError as e:
        raise InvalidSignatureError from e

    public_key = public_key.format(compressed=False)[1:]
    return public_key
```

**Parameter note**: The `v` parameter in `secp256k1_recover` is the raw recovery bit (`0` or `1`), not the `v` field from a transaction. The `ECRECOVER` precompile (which accepts `v = 27` or `v = 28`) subtracts 27 before calling this function: `secp256k1_recover(r, s, v - U256(27), message_hash)`.

### 3.2 The ECRECOVER Precompile

The precompile at address `0x01`, defined in [`ethereum/forks/osaka/vm/precompiled_contracts/ecrecover.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/ecrecover.py), wraps `secp256k1_recover` and derives an Ethereum address from the recovered public key:

```python
def ecrecover(evm: Evm) -> None:
    """
    Decrypts the address using elliptic curve DSA recovery mechanism and
    writes the address to output.
    """
    data = evm.message.data

    # GAS
    charge_gas(evm, GAS_PRECOMPILE_ECRECOVER)

    # OPERATION
    message_hash = Hash32(buffer_read(data, U256(0), U256(32)))
    v = U256.from_be_bytes(buffer_read(data, U256(32), U256(32)))
    r = U256.from_be_bytes(buffer_read(data, U256(64), U256(32)))
    s = U256.from_be_bytes(buffer_read(data, U256(96), U256(32)))

    if v != U256(27) and v != U256(28):
        return
    if U256(0) >= r or r >= SECP256K1N:
        return
    if U256(0) >= s or s >= SECP256K1N:
        return

    try:
        public_key = secp256k1_recover(r, s, v - U256(27), message_hash)
    except InvalidSignatureError:
        return

    address = keccak256(public_key)[12:32]
    padded_address = left_pad_zero_bytes(address, 32)
    evm.output = padded_address
```

The address derivation step `keccak256(public_key)[12:32]` takes the last 20 bytes of the 32-byte Keccak-256 hash of the public key, which is the standard way an EOA address is derived from a public key.

### 3.3 secp256r1: P-256 Signature Verification (EIP-7951)

Osaka introduces the P-256 (secp256r1) precompile at address `0x100` via EIP-7951. Unlike secp256k1, secp256r1 is used for **verification** only, not recovery. The curve constants are:

```python
SECP256R1N = U256(0xFFFFFFFF00000000FFFFFFFFFFFFFFFFBCE6FAADA7179E84F3B9CAC2FC632551)
SECP256R1P = U256(0xFFFFFFFF00000001000000000000000000000000FFFFFFFFFFFFFFFFFFFFFFFF)
SECP256R1A = U256(0xFFFFFFFF00000001000000000000000000000000FFFFFFFFFFFFFFFFFFFFFFFC)
SECP256R1B = U256(0x5AC635D8AA3A93E7B3EBBD55769886BC651D06B0CC53B0F63BCE3C3E27D2604B)
```

[**`secp256r1_verify`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/crypto/elliptic_curve.py) takes a signature `(r, s)`, public key coordinates `(x, y)`, and a pre-hashed message, and raises `InvalidSignatureError` on failure:

```python
def secp256r1_verify(
    r: U256, s: U256, x: U256, y: U256, msg_hash: Hash32
) -> None:
    """
    Verifies a P-256 signature.
    Raises InvalidSignatureError if the signature is not valid.
    """
    r_int, s_int, x_int, y_int = int(r), int(s), int(x), int(y)

    sig = DerSequence([r_int, s_int]).encode()
    pubnum = ec.EllipticCurvePublicNumbers(x_int, y_int, ec.SECP256R1())
    pubkey = pubnum.public_key(default_backend())

    try:
        pubkey.verify(sig, msg_hash, ec.ECDSA(Prehashed(hashes.SHA256())))
    except InvalidSignature as e:
        raise InvalidSignatureError from e
```

Note that this precompile hashes its input with SHA-256 internally (via `Prehashed(hashes.SHA256())`), so `msg_hash` is expected to already be a SHA-256 digest.

---

## 4. RLP Encoding

RLP (Recursive Length Prefix) is Ethereum’s canonical serialization format. It is used for encoding transactions, accounts (in the state trie), MPT nodes, and contract addresses. The EELS uses the [`ethereum_rlp`](https://github.com/ethereum/ethereum-rlp) library, which is maintained as a separate package and imported as `rlp` throughout the specification.

### 4.1 Encoding Rules

RLP encodes two categories of items: byte strings and lists. The encoding rules are:

| Input | Rule |
| --- | --- |
| Single byte in range `[0x00, 0x7f]` | Encoded as itself (identity). |
| Byte string of length `0–55` | `0x80 + len` followed by the bytes. |
| Byte string of length `> 55` | `0xb7 + len(len_bytes)`, then the big-endian length, then the bytes. |
| List whose encoded payload is `0–55` bytes | `0xc0 + len` followed by the encoded items. |
| List whose encoded payload is `> 55` bytes | `0xf7 + len(len_bytes)`, then the big-endian length, then the encoded items. |

### 4.2 Usage in the Specification

The primary entry point is `rlp.encode(item)`. Callers pass Python dataclasses, byte strings, integers, or tuples; the library dispatches on type. Key uses across the Osaka fork:

| Use | Expression |
| --- | --- |
| Account encoding | `rlp.encode((nonce, balance, storage_root, code_hash))` |
| CREATE address | `rlp.encode([address, nonce])` |
| Transaction hashing | `rlp.encode(transaction)` |
| MPT node serialization | `rlp.encode(internal_node)` |

---

## 5. Address Derivation

`Address` is a type alias for `Bytes20` (20-byte value), defined in the shared [`ethereum/state.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/state.py). The three categories of address — EOA, `CREATE`, and `CREATE2` — each use a different derivation scheme. The `CREATE` and `CREATE2` helpers are defined in [`ethereum/forks/osaka/utils/address.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/utils/address.py).

### 5.1 EOA Address

An EOA’s address is derived from the uncompressed secp256k1 public key (64 bytes, without the `0x04` prefix):

```
address = keccak256(public_key)[12:32]
```

This takes the last 20 bytes of the 32-byte Keccak-256 hash. This derivation is baked into the `ECRECOVER` precompile (see [§3.2](about:blank#32-the-ecrecover-precompile)) and is not implemented as a standalone function in the EELS, since the spec does not cover key generation.

### 5.2 Contract Address (`CREATE`)

When a contract is deployed via `CREATE`, its address is derived from the deployer’s address and their current nonce:

```
address = keccak256(rlp([sender, nonce]))[-20:]
```

[**`compute_contract_address`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/utils/address.py):

```python
def compute_contract_address(address: Address, nonce: Uint) -> Address:
    """
    Computes address of the new account that needs to be created.
    """
    computed_address = keccak256(rlp.encode([address, nonce]))
    canonical_address = computed_address[-20:]
    padded_address = left_pad_zero_bytes(canonical_address, 20)
    return Address(padded_address)
```

`left_pad_zero_bytes` zero-pads the result to exactly 20 bytes, handling edge cases where the hash truncation could yield fewer than 20 significant bytes. Because the address depends on `nonce`, it changes if the deployer sends other transactions first.

### 5.3 Contract Address (`CREATE2`)

`CREATE2` enables fully deterministic address pre-computation. The address depends on the deployer, a caller-supplied salt, and the hash of the initialization bytecode, but not on the deployer’s nonce:

```
address = keccak256(0xff ++ sender ++ salt ++ keccak256(init_code))[-20:]
```

[**`compute_create2_contract_address`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/utils/address.py):

```python
def compute_create2_contract_address(
    address: Address, salt: Bytes32, call_data: Bytes
) -> Address:
    """
    Computes address of the new account that needs to be created, which is
    based on the sender address, salt and the call data as well.
    """
    preimage = b"\xff" + address + salt + keccak256(call_data)
    computed_address = keccak256(preimage)
    canonical_address = computed_address[-20:]
    padded_address = left_pad_zero_bytes(canonical_address, 20)
    return Address(padded_address)
```

The `0xff` prefix byte prevents collisions with `CREATE` addresses (whose RLP-encoded preimage can never begin with `0xff`). The parameter name in the EELS is `call_data`, referring to the initialization bytecode passed to the `CREATE2` instruction.

### 5.4 Comparison

| Scheme | Preimage | Nonce-dependent |
| --- | --- | --- |
| EOA | `keccak256(public_key)` | No |
| `CREATE` | `rlp([sender, nonce])` | Yes |
| `CREATE2` | `0xff ++ sender ++ salt ++ keccak256(init_code)` | No |

`CREATE2` is commonly used for counterfactual deployments: the address can be computed and used (e.g., funded, referenced in other contracts) before the contract is actually deployed.