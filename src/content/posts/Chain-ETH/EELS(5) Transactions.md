---
title: EELS(5) Transactions
published: 2026-04-02
pinned: false
description: Ethereum’s user transaction formats, validation stages, and execution lifecycle
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

1. [**Preface**](#preface)
2. [**Introduction**](#introduction)
3. [**Transaction Types**](#transaction-types)
4. [**Transaction Validation**](#transaction-validation)
5. [**Receipt Generation**](#receipt-generation)
6. [**Transaction Processing**](#transaction-processing)

## Preface

This article is part of a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**.

**What is EELS?** EELS is a Python reference implementation of the Ethereum execution client, designed with a focus on readability and clarity. It serves as a programmer-friendly, up-to-date successor to the original Yellow Paper and is the primary tool for prototyping new Ethereum Improvement Proposals (EIPs). EELS provides complete protocol snapshots at each fork and rendered diffs between them.

**Scope and Implementation** Note that EELS does not implement the JSON-RPC API or P2P networking. To validate blocks, it requires an external RPC provider to fetch data, which EELS then processes and stores in a local database.

**Version Reference** The code analysis in this article is based on the **Osaka** fork:
[ethereum/execution-specs (Osaka)](https://github.com/ethereum/execution-specs/tree/forks/amsterdam/src/ethereum/forks/osaka)

---

## Introduction

This chapter covers transaction formats, validation, and processing as implemented in the Osaka fork of the Ethereum Execution Layer Specification (EELS).

---

## Transaction Types

Ethereum defines five transaction formats, each introduced by a distinct EIP. All types are defined in [`transactions.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/transactions.py) as a union:

```python
Transaction = (
    LegacyTransaction
    | AccessListTransaction
    | FeeMarketTransaction
    | BlobTransaction
    | SetCodeTransaction
)
```

> **Rendered spec**: [`ethereum.forks.osaka.transactions`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html)
> 

### Evolution Timeline

```
Legacy (Frontier)
  └─> EIP-2930 (Berlin):  Access lists + explicit chain ID
      └─> EIP-1559 (London): Fee market (base fee + priority fee)
          ├─> EIP-4844 (Cancun): Blob-carrying transactions
          └─> EIP-7702 (Prague): EOA code delegation
```

---

### Shared Structures

Before examining individual types, two shared structures used across typed transactions are worth noting.

[**`Access`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/transactions.py) — a structured representation of an account and its pre-warmed storage slots, used in the `access_list` field of Types 1–4:

```python
@slotted_freezable
@dataclass
class Access:
    account: Address
    slots: Tuple[Bytes32, ...]
```

[**`Authorization`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork_types.py) — carries a signed delegation from an EOA to a target contract address, used exclusively by the Type 4 `SetCodeTransaction`:

```python
@slotted_freezable
@dataclass
class Authorization:
    chain_id: U256
    address: Address   # Target contract to delegate to
    nonce: U64
    y_parity: U256
    r: U256
    s: U256
```

---

### Legacy Transaction (Type 0)

**Type identifier**: none (pre-[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) envelope format)

```python
@slotted_freezable
@dataclass
class LegacyTransaction:
    nonce: U256
    gas_price: Uint
    gas: Uint
    to: Bytes0 | Address   # Bytes0(b"") signals contract creation
    value: U256
    data: Bytes
    v: U256
    r: U256
    s: U256
```

> [`LegacyTransaction` spec](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.LegacyTransaction:0)
> 

The single `gas_price` field covers both base fee and priority fee, predating the EIP-1559 fee model. The `v` field encodes replay protection:

- Pre-[EIP-155](https://eips.ethereum.org/EIPS/eip-155): `v = 27` or `28`
- EIP-155: `v = chain_id × 2 + 35` or `chain_id × 2 + 36`

**Signing hashes** ([`signing_hash_pre155`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.signing_hash_pre155:0) / [`signing_hash_155`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.signing_hash_155:0)):

```python
# Pre-EIP-155
signing_hash = keccak256(rlp.encode(
    (nonce, gas_price, gas, to, value, data)
))

# EIP-155
signing_hash = keccak256(rlp.encode(
    (nonce, gas_price, gas, to, value, data, chain_id, Uint(0), Uint(0))
))
```

**Wire encoding**: RLP-encoded directly (no type prefix).

---

### Access List Transaction (Type 1, [EIP-2930](https://eips.ethereum.org/EIPS/eip-2930))

**Type identifier**: `0x01`

```python
@slotted_freezable
@dataclass
class AccessListTransaction:
    chain_id: U64
    nonce: U256
    gas_price: Uint
    gas: Uint
    to: Bytes0 | Address
    value: U256
    data: Bytes
    access_list: Tuple[Access, ...]
    y_parity: U256
    r: U256
    s: U256
```

> [`AccessListTransaction` spec](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.AccessListTransaction:0)
> 

**New fields over Legacy**:

- `chain_id`: Explicit chain identifier, replacing the implicit EIP-155 encoding in `v`. Prevents cross-chain replay without relying on signature encoding tricks.
- `access_list`: Tuple of `Access` objects declaring addresses and storage slots the transaction will touch. Slots declared here are loaded into the warm access set before execution begins, reducing their per-access gas cost (2,400 gas per warm address vs. 2,600 cold; 1,900 per warm slot vs. 2,100 cold).
- `y_parity`: Normalized signature recovery bit (`0` or `1`), replacing the legacy `v` value.

**Wire encoding**: `0x01 || rlp([chain_id, nonce, gas_price, gas, to, value, data, access_list, y_parity, r, s])`

---

### Fee Market Transaction (Type 2, [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559))

**Type identifier**: `0x02`

```python
@slotted_freezable
@dataclass
class FeeMarketTransaction:
    chain_id: U64
    nonce: U256
    max_priority_fee_per_gas: Uint
    max_fee_per_gas: Uint
    gas: Uint
    to: Bytes0 | Address
    value: U256
    data: Bytes
    access_list: Tuple[Access, ...]
    y_parity: U256
    r: U256
    s: U256
```

> [`FeeMarketTransaction` spec](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.FeeMarketTransaction:0)
> 

**New fields over Type 1**:

- `max_priority_fee_per_gas`: The tip the sender is willing to pay to the validator, on top of the base fee.
- `max_fee_per_gas`: The maximum total fee per gas unit (base fee + tip) the sender will accept. If the block’s `base_fee_per_gas` exceeds this, the transaction is excluded.

The **effective gas price** actually charged ([`check_transaction`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html#ethereum.forks.osaka.fork.check_transaction:0)):

```python
priority_fee_per_gas = min(
    tx.max_priority_fee_per_gas,
    tx.max_fee_per_gas - block_env.base_fee_per_gas,
)
effective_gas_price = priority_fee_per_gas + block_env.base_fee_per_gas
```

The base fee is burned; the priority fee goes to the validator. Any difference between `max_fee_per_gas` and `effective_gas_price` is refunded to the sender.

**Wire encoding**: `0x02 || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas, to, value, data, access_list, y_parity, r, s])`

---

### Blob Transaction (Type 3, [EIP-4844](https://eips.ethereum.org/EIPS/eip-4844))

**Type identifier**: `0x03`

```python
@slotted_freezable
@dataclass
class BlobTransaction:
    chain_id: U64
    nonce: U256
    max_priority_fee_per_gas: Uint
    max_fee_per_gas: Uint
    gas: Uint
    to: Address        # Contract creation is not permitted
    value: U256
    data: Bytes
    access_list: Tuple[Access, ...]
    max_fee_per_blob_gas: U256
    blob_versioned_hashes: Tuple[VersionedHash, ...]
    y_parity: U256
    r: U256
    s: U256
```

> [`BlobTransaction` spec](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.BlobTransaction:0)
> 

**New fields over Type 2**:

- `max_fee_per_blob_gas`: The maximum blob gas price per unit of blob gas the sender is willing to pay. Blob gas is priced independently of execution gas via a separate exponential fee market.
- `blob_versioned_hashes`: KZG commitment hashes for the blobs attached to this transaction. Each hash has a one-byte version prefix (`0x01` for KZG). The actual blob data is propagated over the P2P layer and is not stored on-chain.

**Constraints** (enforced in [`check_transaction`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html#ethereum.forks.osaka.fork.check_transaction:0)):

- `to` must be an `Address`; contract creation is disallowed.
- At least one blob versioned hash must be present.
- The blob count must not exceed `BLOB_COUNT_LIMIT = 6` (Osaka).
- Every versioned hash must start with `VERSIONED_HASH_VERSION_KZG = b"\x01"`.
- `max_fee_per_blob_gas` must be at least the current `blob_gas_price`.

**Wire encoding**: `0x03 || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas, to, value, data, access_list, max_fee_per_blob_gas, blob_versioned_hashes, y_parity, r, s])`

---

### Set Code Transaction (Type 4, [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702))

**Type identifier**: `0x04`

```python
@slotted_freezable
@dataclass
class SetCodeTransaction:
    chain_id: U64
    nonce: U64          # Note: U64, not U256
    max_priority_fee_per_gas: Uint
    max_fee_per_gas: Uint
    gas: Uint
    to: Address        # Contract creation is not permitted
    value: U256
    data: Bytes
    access_list: Tuple[Access, ...]
    authorizations: Tuple[Authorization, ...]
    y_parity: U256
    r: U256
    s: U256
```

> [`SetCodeTransaction` spec](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.SetCodeTransaction:0)
> 

**New field over Type 2**:

- `authorizations`: One or more `Authorization` objects. Each authorization is signed separately by an EOA that wishes to delegate its execution to a target contract. Upon processing, the delegating account’s code is set to the delegation designator `0xef0100 || address` (23 bytes), causing calls to that EOA to execute the target contract’s code in the EOA’s context.

**Constraints**:

- `to` must be an `Address`; contract creation is disallowed.
- `authorizations` must contain at least one entry.

**Wire encoding**: `0x04 || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas, to, value, data, access_list, authorizations, y_parity, r, s])`

---

### Transaction Type Comparison

| Feature | Legacy (0) | EIP-2930 (1) | EIP-1559 (2) | EIP-4844 (3) | EIP-7702 (4) |
| --- | --- | --- | --- | --- | --- |
| Explicit chain ID | ✅ (in `v`) | ✅ | ✅ | ✅ | ✅ |
| Access list | ❌ | ✅ | ✅ | ✅ | ✅ |
| EIP-1559 fee fields | ❌ | ❌ | ✅ | ✅ | ✅ |
| Blob versioned hashes | ❌ | ❌ | ❌ | ✅ | ❌ |
| Authorizations | ❌ | ❌ | ❌ | ❌ | ✅ |
| Contract creation | ✅ | ✅ | ✅ | ❌ | ❌ |

---

## Transaction Validation

Every user transaction goes through two validation stages before execution: **static validation** (format and intrinsic cost, independent of world state) followed by **dynamic validation** (state-dependent checks). Both are invoked from [`process_transaction`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html) inside [`apply_body`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html).

### Static Validation — `validate_transaction`

[`validate_transaction`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.validate_transaction:0) performs state-independent checks and returns a pair `(intrinsic_gas, calldata_floor_gas_cost)`:

```python
def validate_transaction(tx: Transaction) -> Tuple[Uint, Uint]:
    intrinsic_gas, calldata_floor_gas_cost = calculate_intrinsic_cost(tx)

    # 1. Gas limit must cover both intrinsic cost and calldata floor
    if max(intrinsic_gas, calldata_floor_gas_cost) > tx.gas:
        raise InsufficientTransactionGasError("Insufficient gas")

    # 2. EIP-2681: nonce must be < 2^64 - 1
    if U256(tx.nonce) >= U256(U64.MAX_VALUE):
        raise NonceOverflowError("Nonce too high")

    # 3. EIP-3860: init code size limit for contract creation
    if tx.to == Bytes0(b"") and len(tx.data) > MAX_INIT_CODE_SIZE:
        raise InitCodeTooLargeError("Code size too large")

    # 4. EIP-7825: per-transaction gas cap
    if tx.gas > TX_MAX_GAS_LIMIT:
        raise TransactionGasLimitExceededError("Gas limit too high")

    return intrinsic_gas, calldata_floor_gas_cost
```

This function verifies a transaction by enforcing several critical constraints.

- The gas in a transaction gets used to pay for the intrinsic cost of operations, therefore if there is insufficient gas then it would not be possible to execute a transaction and it will be declared invalid.
- Additionally, the nonce of a transaction must not equal or exceed the limit defined in EIP-2681. In practice, defining the limit as 2^64-1 has no impact because sending 2^64-1 transactions is improbable.
- Also, the code size of a contract creation transaction must be within limits of the protocol.

This function takes a transaction as a parameter and returns the intrinsic gas cost and the minimum calldata gas cost for the transaction after validation.

The intrinsic cost computation is delegated to [`calculate_intrinsic_cost`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.calculate_intrinsic_cost:0):

```python
def calculate_intrinsic_cost(tx: Transaction) -> Tuple[Uint, Uint]:
    num_zeros = Uint(tx.data.count(0))
    num_non_zeros = ulen(tx.data) - num_zeros

    # EIP-7623: token model — 1 zero byte = 1 token, 1 non-zero byte = 4 tokens
    tokens_in_calldata = num_zeros + num_non_zeros * Uint(4)

    # Calldata floor gas cost (EIP-7623)
    calldata_floor_gas_cost = (
        tokens_in_calldata * GAS_TX_DATA_TOKEN_FLOOR + GAS_TX_BASE
    )

    # Standard calldata cost charged toward intrinsic gas
    data_cost = tokens_in_calldata * GAS_TX_DATA_TOKEN_STANDARD

    # Additional cost for contract creation
    if tx.to == Bytes0(b""):
        create_cost = GAS_TX_CREATE + init_code_cost(ulen(tx.data))
    else:
        create_cost = Uint(0)

    # Access list cost (Types 1–4)
    access_list_cost = Uint(0)
    if has_access_list(tx):
        for access in tx.access_list:
            access_list_cost += GAS_TX_ACCESS_LIST_ADDRESS
            access_list_cost += ulen(access.slots) * GAS_TX_ACCESS_LIST_STORAGE_KEY

    # Authorization cost (Type 4 only)
    auth_cost = Uint(0)
    if isinstance(tx, SetCodeTransaction):
        auth_cost += Uint(GAS_AUTH_PER_EMPTY_ACCOUNT * len(tx.authorizations))

    return (
        Uint(GAS_TX_BASE + data_cost + create_cost + access_list_cost + auth_cost),
        calldata_floor_gas_cost,
    )
```

This function takes a transaction as a parameter and returns the intrinsic gas cost of the transaction and the minimum gas cost used by the transaction based on the calldata size. The intrinsic cost includes:

- Base cost (`GAS_TX_BASE`)
- Cost for data (zero and non-zero bytes)
- Cost for contract creation (if applicable)
- Cost for access list entries (if applicable)
- Cost for authorizations (if applicable)

**Gas constants** ([`transactions.py`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html)):

| Constant | Value | Purpose |
| --- | --- | --- |
| `GAS_TX_BASE` | 21,000 | Minimum cost for any transaction |
| `GAS_TX_DATA_TOKEN_STANDARD` | 4 | Gas per calldata token (standard) |
| `GAS_TX_DATA_TOKEN_FLOOR` | 10 | Gas per calldata token (EIP-7623 floor) |
| `GAS_TX_CREATE` | 32,000 | Additional cost for contract creation |
| `GAS_TX_ACCESS_LIST_ADDRESS` | 2,400 | Cost per address in access list |
| `GAS_TX_ACCESS_LIST_STORAGE_KEY` | 1,900 | Cost per storage slot in access list |
| `TX_MAX_GAS_LIMIT` | 16,777,216 | Per-transaction gas cap (EIP-7825, Osaka) |

The EIP-7623 calldata floor prevents transactions from using cheap calldata as a data-availability channel while avoiding meaningful execution. After the refund is applied post-execution, the protocol enforces `gas_used = max(execution_gas_used, calldata_floor_gas_cost)`.

---

### Dynamic Validation — `check_transaction`

[`check_transaction`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html#ethereum.forks.osaka.fork.check_transaction:0) performs world-state-dependent checks and returns `(sender_address, effective_gas_price, blob_versioned_hashes, tx_blob_gas_used)`:

```python
def check_transaction(
    block_env: BlockEnvironment,
    block_output: BlockOutput,
    tx: Transaction,
) -> Tuple[Address, Uint, Tuple[VersionedHash, ...], U64]:

    # 1. Block gas availability
    gas_available = block_env.block_gas_limit - block_output.block_gas_used
    if tx.gas > gas_available:
        raise GasUsedExceedsLimitError("gas used exceeds limit")

    # 2. Blob gas availability
    tx_blob_gas_used = calculate_total_blob_gas(tx)
    blob_gas_available = MAX_BLOB_GAS_PER_BLOCK - block_output.blob_gas_used
    if tx_blob_gas_used > blob_gas_available:
        raise BlobGasLimitExceededError("blob gas limit exceeded")

    # 3. Sender recovery from signature
    sender_address = recover_sender(block_env.chain_id, tx)
    sender_account = get_account(block_env.state, sender_address)

    # 4. Fee validation
    if isinstance(tx, (FeeMarketTransaction, BlobTransaction, SetCodeTransaction)):
        if tx.max_fee_per_gas < tx.max_priority_fee_per_gas:
            raise PriorityFeeGreaterThanMaxFeeError(...)
        if tx.max_fee_per_gas < block_env.base_fee_per_gas:
            raise InsufficientMaxFeePerGasError(...)
        priority_fee_per_gas = min(
            tx.max_priority_fee_per_gas,
            tx.max_fee_per_gas - block_env.base_fee_per_gas,
        )
        effective_gas_price = priority_fee_per_gas + block_env.base_fee_per_gas
        max_gas_fee = tx.gas * tx.max_fee_per_gas
    else:  # Legacy and Type 1
        if tx.gas_price < block_env.base_fee_per_gas:
            raise InvalidBlock
        effective_gas_price = tx.gas_price
        max_gas_fee = tx.gas * tx.gas_price

    # 5. Blob-specific validation (Type 3)
    if isinstance(tx, BlobTransaction):
        blob_count = len(tx.blob_versioned_hashes)
        if blob_count == 0:
            raise NoBlobDataError(...)
        if blob_count > BLOB_COUNT_LIMIT:
            raise BlobCountExceededError(...)
        for blob_versioned_hash in tx.blob_versioned_hashes:
            if blob_versioned_hash[0:1] != VERSIONED_HASH_VERSION_KZG:
                raise InvalidBlobVersionedHashError(...)
        blob_gas_price = calculate_blob_gas_price(block_env.excess_blob_gas)
        if Uint(tx.max_fee_per_blob_gas) < blob_gas_price:
            raise InsufficientMaxFeePerBlobGasError(...)
        max_gas_fee += Uint(calculate_total_blob_gas(tx)) * Uint(tx.max_fee_per_blob_gas)
        blob_versioned_hashes = tx.blob_versioned_hashes
    else:
        blob_versioned_hashes = ()

    # 6. Contract creation restrictions (Types 3 and 4)
    if isinstance(tx, (BlobTransaction, SetCodeTransaction)):
        if not isinstance(tx.to, Address):
            raise TransactionTypeContractCreationError(tx)

    # 7. Authorization list must be non-empty (Type 4)
    if isinstance(tx, SetCodeTransaction):
        if not any(tx.authorizations):
            raise EmptyAuthorizationListError(...)

    # 8. Nonce check
    if sender_account.nonce > Uint(tx.nonce):
        raise NonceMismatchError("nonce too low")
    elif sender_account.nonce < Uint(tx.nonce):
        raise NonceMismatchError("nonce too high")

    # 9. Balance check
    if Uint(sender_account.balance) < max_gas_fee + Uint(tx.value):
        raise InsufficientBalanceError(...)

    # 10. Sender must be an EOA (or a delegated EOA per EIP-7702)
    sender_code = get_code(block_env.state, sender_account.code_hash)
    if sender_account.code_hash != EMPTY_CODE_HASH and not is_valid_delegation(sender_code):
        raise InvalidSenderError("not EOA")

    return (sender_address, effective_gas_price, blob_versioned_hashes, tx_blob_gas_used)
```

The dynamic validation process follows these ten steps:

1. **Block Gas Availability**: The protocol ensures the transaction's requested gas limit does not exceed the remaining gas capacity of the current block.
2. **Blob Gas Availability**: For blob-carrying transactions, the protocol verifies that the total blob gas required fits within the remaining blob gas limit for the block (max 6 blobs in Osaka fork).
3. **Sender Recovery**: The sender's address is recovered from the ECDSA signature. This address is then used to look up the sender's account state in the trie.
4. **Fee Validation**: The transaction's fee parameters are checked against the block's base_fee_per_gas. For EIP-1559 types, the effective_gas_price is calculated as the sum of the base fee and the priority fee (capped by the max fee).
5. **Blob-specific Validation**: For Type 3 transactions, the protocol validates that the blob count is within limits, hashes use the correct KZG version prefix, and the provided max_fee_per_blob_gas is sufficient to cover the network's current blob gas price.
6. **Contract Creation Restrictions**: Blob transactions (Type 3) and Set Code transactions (Type 4) are explicitly prohibited from performing contract creation; the to field must be a valid Address.
7. **Authorization Check**: Specifically for Type 4 transactions, the protocol ensures that the authorizations list contains at least one signed delegation.
8. **Nonce Check**: The transaction nonce must exactly match the sender’s current nonce in the state. This enforces strict transaction ordering and prevents replay attacks.
9. **Balance Check**: The sender's balance must be sufficient to cover the "upfront cost," which includes the maximum possible gas fee, any blob fees, and the endowment (ETH value) being sent.
10. **Sender Type Verification**: Finally, the protocol confirms the sender is an Externally Owned Account (EOA). An account with code is generally invalid as a sender, unless that code is a valid EIP-7702 delegation designator.

**Dynamic checks summary**:

| Check | Condition | Error |
| --- | --- | --- |
| Block gas | `tx.gas ≤ gas_available` | `GasUsedExceedsLimitError` |
| Blob gas | `tx_blob_gas_used ≤ blob_gas_available` | `BlobGasLimitExceededError` |
| Fee sufficiency | `max_fee_per_gas ≥ base_fee_per_gas` | `InsufficientMaxFeePerGasError` |
| Blob fee | `max_fee_per_blob_gas ≥ blob_gas_price` | `InsufficientMaxFeePerBlobGasError` |
| Blob count | `1 ≤ blob_count ≤ BLOB_COUNT_LIMIT` | `NoBlobDataError` / `BlobCountExceededError` |
| Nonce | `sender.nonce == tx.nonce` | `NonceMismatchError` |
| Balance | `sender.balance ≥ max_gas_fee + tx.value` | `InsufficientBalanceError` |
| Sender type | EOA or delegated EOA | `InvalidSenderError` |

---

### Sender Recovery — `recover_sender`

[`recover_sender`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.recover_sender:0) reconstructs the sender’s address from the transaction’s ECDSA signature. The signing hash differs per transaction type, each using a dedicated function:

| Transaction type | Signing hash function |
| --- | --- |
| Legacy, pre-EIP-155 | [`signing_hash_pre155`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.signing_hash_pre155:0) |
| Legacy, EIP-155 | [`signing_hash_155`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.signing_hash_155:0) |
| Type 1 (EIP-2930) | [`signing_hash_2930`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.signing_hash_2930:0) |
| Type 2 (EIP-1559) | [`signing_hash_1559`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.signing_hash_1559:0) |
| Type 3 (EIP-4844) | [`signing_hash_4844`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.signing_hash_4844:0) |
| Type 4 (EIP-7702) | [`signing_hash_7702`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.signing_hash_7702:0) |

In all cases the recovered public key is hashed and the last 20 bytes become the sender address:

```python
public_key = secp256k1_recover(r, s, y_parity, signing_hash)
return Address(keccak256(public_key)[12:32])
```

Signature bounds are validated before recovery: `r` and `s` must be non-zero, `r < SECP256K1N`, and `s ≤ SECP256K1N // 2` (the low-S constraint from [EIP-2](https://eips.ethereum.org/EIPS/eip-2)).

---

## Receipt Generation

A receipt is produced for every executed transaction regardless of success or failure. Receipts are assembled by [`make_receipt`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html#ethereum.forks.osaka.fork.make_receipt:0) and stored in the receipts trie.

### Receipt Structure

Defined in [`blocks.py`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/blocks.py.html):

```python
@dataclass
class Receipt:
    succeeded: bool         # True if the transaction did not revert
    cumulative_gas_used: Uint  # Total gas used in the block up to and including this tx
    bloom: Bloom            # Bloom filter over this transaction's logs
    logs: Tuple[Log, ...]   # Events emitted during execution
```

1. **Status (succeeded)**: A boolean indicating if the top-level EVM execution completed without an error.
2. **Cumulative Gas Used**: This represents the running total of gas consumed in the block. By subtracting the cumulative gas of the previous receipt from the current one, the gas cost of a specific transaction can be derived.
3. **Logs**: An ordered sequence of event objects (Log) emitted by the EVM LOG0 through LOG4 instructions.
4. **Bloom Filter**: A 2048-bit data structure that summarizes the addresses and topics present in the logs, allowing for efficient searching of events without scanning the entire state.

### Receipt Encoding

Receipts are type-prefixed to match their corresponding transaction type ([`encode_transaction`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html#ethereum.forks.osaka.transactions.encode_transaction:0) mirrors this pattern for transactions). The code is shown in [`block.py`](http://block.pyhttps://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/blocks.py):

```python
# Type 1 (EIP-2930): 0x01 || rlp(receipt)
if isinstance(tx, AccessListTransaction): 
    return b"\x01" + rlp.encode(receipt)
# Type 2 (EIP-1559): 0x02 || rlp(receipt)
elif isinstance(tx, FeeMarketTransaction):
    return b"\x02" + rlp.encode(receipt)
# Type 3 (EIP-4844): 0x03 || rlp(receipt)
elif isinstance(tx, BlobTransaction):
    return b"\x03" + rlp.encode(receipt)
# Type 4 (EIP-7702): 0x04 || rlp(receipt)
elif isinstance(tx, SetCodeTransaction):
    return b"\x04" + rlp.encode(receipt)
# Legacy: RLP-encoded directly.
else:
    return receipt
```

Once generated, receipts are stored in a dedicated Receipts Trie, the root of which is committed to the block header.

## Transaction Processing

[`process_transaction`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py) orchestrates the full lifecycle of a single user transaction within a block. It is called for each transaction by [`apply_body`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py) in the order they appear in the block.

```python
def process_transaction(
    block_env: vm.BlockEnvironment,
    block_output: vm.BlockOutput,
    tx: Transaction,
    index: Uint,
) -> None:
    # 1. Insert transaction into the transactions trie
    trie_set(
        block_output.transactions_trie,
        rlp.encode(index),
        encode_transaction(tx),
    )

    # 2. Validation
    intrinsic_gas, calldata_floor_gas_cost = validate_transaction(tx)
    (sender, effective_gas_price, blob_hashes, tx_blob_gas_used) = check_transaction(
        block_env=block_env, block_output=block_output, tx=tx,
    )

    # 3. Initial state changes & upfront fee deduction
    sender_account = get_account(block_env.state, sender)
    blob_gas_fee = calculate_data_fee(block_env.excess_blob_gas, tx) if isinstance(tx, BlobTransaction) else Uint(0)
    effective_gas_fee = tx.gas * effective_gas_price
    
    increment_nonce(block_env.state, sender)
    sender_balance_after_fee = Uint(sender_account.balance) - effective_gas_fee - blob_gas_fee
    set_account_balance(block_env.state, sender, U256(sender_balance_after_fee))

    # 4. Prepare execution environment and access lists
    access_list_addresses = {block_env.coinbase}
    access_list_storage_keys = set()
    if has_access_list(tx): # Types 1-4
        for access in tx.access_list:
            access_list_addresses.add(access.account)
            for slot in access.slots:
                access_list_storage_keys.add((access.account, slot))

    tx_env = vm.TransactionEnvironment(
        origin=sender,
        gas_price=effective_gas_price,
        gas=tx.gas - intrinsic_gas,
        access_list_addresses=access_list_addresses,
        access_list_storage_keys=access_list_storage_keys,
        transient_storage=TransientStorage(),
        blob_versioned_hashes=blob_hashes,
        authorizations=tx.authorizations if isinstance(tx, SetCodeTransaction) else (),
        index_in_block=index,
        tx_hash=get_transaction_hash(encode_transaction(tx)),
    )

    # 5. EVM Execution
    message = prepare_message(block_env, tx_env, tx)
    tx_output = process_message_call(message)

    # 6. Gas Accounting (Refunds and EIP-7623 Floor)
    tx_gas_used_before_refund = tx.gas - tx_output.gas_left
    tx_gas_refund = min(tx_gas_used_before_refund // Uint(5), Uint(tx_output.refund_counter))
    tx_gas_used_after_refund = max(tx_gas_used_before_refund - tx_gas_refund, calldata_floor_gas_cost)

    # 7. Fee Settlement
    gas_refund_amount = (tx.gas - tx_gas_used_after_refund) * effective_gas_price
    transaction_fee = tx_gas_used_after_refund * (effective_gas_price - block_env.base_fee_per_gas)

    # Credit sender refund and validator fees
    set_account_balance(block_env.state, sender, get_account(block_env.state, sender).balance + U256(gas_refund_amount))
    set_account_balance(block_env.state, block_env.coinbase, get_account(block_env.state, block_env.coinbase).balance + U256(transaction_fee))

    # 8. Finalization and Receipt Generation
    for address in tx_output.accounts_to_delete:
        destroy_account(block_env.state, address)

    block_output.block_gas_used += tx_gas_used_after_refund
    block_output.blob_gas_used += tx_blob_gas_used

    # Create and store the receipt
    receipt = make_receipt(tx, tx_output.error, block_output.block_gas_used, tx_output.logs)
    receipt_key = rlp.encode(Uint(index))
    block_output.receipt_keys += (receipt_key,)
    trie_set(block_output.receipts_trie, receipt_key, receipt)
    block_output.block_logs += tx_output.logs
```

### Processing Steps

1. **Trie Initialization**: The transaction is RLP-encoded and immediately inserted into the transactions_trie at its designated index.
2. **Validation**: The system runs static checks (intrinsic cost, nonce limits) and dynamic checks (balance, block gas availability).
3. **Upfront Deduction**: The sender’s nonce is incremented, and the maximum possible gas fee (including blob fees) is deducted. This ensures the sender cannot spend the same funds elsewhere during execution.
4. **Environment Setup**: The TransactionEnvironment is built, pre-warming the coinbase address and any addresses/slots specified in the transaction's access list.
5. **Execution**: The EVM processes the message. This step results in state changes, logs, and a record of gas remaining.
6. **Gas Refinement**: The total gas used is calculated. The protocol applies the EIP-3529 refund cap (maximum 20% of consumption) and then enforces the EIP-7623 calldata floor.
7. **Economic Settlement**: Unused gas is refunded to the sender at the effective_gas_price. The validator’s priority fee (the "tip") is calculated and credited to the coinbase account.
8. **Cleanup & Receipts**: Accounts that called SELFDESTRUCT are removed. Finally, a Receipt is generated using the transaction status and the current block-level gas counter, then stored in the receipts_trie.

**Gas flow summary**:

```
Sender Upfront Payment: gas_limit × effective_gas_price
                             │
                             ▼
                       EVM Execution
                             │
               ┌─────────────┴─────────────┐
               ▼                           ▼
            Success                      Revert
       (State Committed)           (State Rolled Back)
               │                           │
       gas_used = gas_limit                │
                - gas_left                 │
               │                    gas_used = gas_limit
       Apply Refund (≤ 1/5)                │
       Apply Calldata Floor                │
               └─────────────┬─────────────┘
                             │
                             ▼
                     Fee Distribution
               ┌─────────────┴─────────────┐
               ▼                           ▼
       Refund Unused Gas           Final Fee (Price × Used)
     (Limit - Used) × Price        ┌───────┴───────┐
               │                   ▼               ▼
         Back to Sender        Base Fee      Priority Fee
                               (Burned)     (To Validator)
```