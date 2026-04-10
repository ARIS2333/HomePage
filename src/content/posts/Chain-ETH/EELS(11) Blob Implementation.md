---
title: EELS(11) Blob Implementation
published: 2026-04-08
pinned: false
description: Demystifying the blob scaling solution using a code-driven walkthrough
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

1. [**Preface**](https://www.notion.so/EELS-11-Blob-Implementation-33e7d90fb9ee80ecb380f6bdc371fec6?pvs=21)
2. [**Introduction**](https://www.notion.so/EELS-11-Blob-Implementation-33e7d90fb9ee80ecb380f6bdc371fec6?pvs=21)
3. [**The BlobTransaction Structure**](https://www.notion.so/EELS-11-Blob-Implementation-33e7d90fb9ee80ecb380f6bdc371fec6?pvs=21)
4. [**KZG Commitments: Data to Anchors**](https://www.notion.so/EELS-11-Blob-Implementation-33e7d90fb9ee80ecb380f6bdc371fec6?pvs=21)
5. [**Transaction Validation Logic**](https://www.notion.so/EELS-11-Blob-Implementation-33e7d90fb9ee80ecb380f6bdc371fec6?pvs=21)
6. [**Blob Gas Accounting**](https://www.notion.so/EELS-11-Blob-Implementation-33e7d90fb9ee80ecb380f6bdc371fec6?pvs=21)
7. [**Block Header Integration**](https://www.notion.so/EELS-11-Blob-Implementation-33e7d90fb9ee80ecb380f6bdc371fec6?pvs=21)
8. [**Sharding Architecture**](https://www.notion.so/EELS-11-Blob-Implementation-33e7d90fb9ee80ecb380f6bdc371fec6?pvs=21)
9. [**EVM Integration**](https://www.notion.so/EELS-11-Blob-Implementation-33e7d90fb9ee80ecb380f6bdc371fec6?pvs=21)
10. [**End-to-End Execution Flow**](https://www.notion.so/EELS-11-Blob-Implementation-33e7d90fb9ee80ecb380f6bdc371fec6?pvs=21)
11. [**Constants Reference**](https://www.notion.so/EELS-11-Blob-Implementation-33e7d90fb9ee80ecb380f6bdc371fec6?pvs=21)

---

## Preface

This article is part of a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**.

**What is EELS?** EELS is a Python reference implementation of the Ethereum execution client. It serves as an up-to-date successor to the Yellow Paper, prioritizing readability for researchers and developers.

**Scope** This article focuses on the **Execution Layer (EL)** implementation of EIP-4844 (Proto-Danksharding) in the **Osaka** fork. While blobs are primarily a Consensus Layer (CL) feature, the EL manages the transaction format, gas accounting, and the cryptographic interface (precompiles) used by smart contracts.

**Version Reference** [ethereum/execution-specs (Osaka Fork)](https://github.com/ethereum/execution-specs/tree/forks/amsterdam/src/ethereum/forks/osaka)

---

## Introduction

In the previous chapter, i.e. EELS(10), we introduced the mechanism of blob. This chapter will be focusing on how EELS implements the code related to blob.

---

## 1. The `BlobTransaction` Structure

### 1.1 Dataclass Definition

The `BlobTransaction` type (`0x03`) is defined in [`transactions.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/transactions.py):

```python
@slotted_freezable
@dataclass
class BlobTransaction:
    chain_id: U64
    nonce: U256
    max_priority_fee_per_gas: Uint
    max_fee_per_gas: Uint
    gas: Uint
    to: Address # must be Address, never None — no contract creation
    value: U256
    data: Bytes
    access_list: Tuple[Access, ...]
    max_fee_per_blob_gas: U256 # sender's maximum bid per unit of blob gas
    blob_versioned_hashes: Tuple[VersionedHash, ...] # one 32-byte hash per attached blob
    y_parity: U256
    r: U256
    s: U256
```

`VersionedHash` is a type alias in [`fork_types.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork_types.py):

```python
VersionedHash = Bytes32
```

The two blob-specific fields are:

| Field | Type | Role |
| --- | --- | --- |
| `max_fee_per_blob_gas` | `U256` | Maximum the sender will pay per unit of blob gas. Checked against the current blob base fee at inclusion time. |
| `blob_versioned_hashes` | `Tuple[VersionedHash, ...]` | One 32-byte versioned hash per attached blob. Each is `0x01 ++ sha256(KZG_commitment)[1:]`. This is the execution layer’s only reference to the blob. |

The raw 128 KB blob data is **not present** in this struct. It travels as a beacon-layer sidecar, completely outside the execution layer. The execution layer only ever handles the 32-byte versioned hashes.

### 1.2 Encoding and Decoding

Blob transactions are framed with the EIP-2718 type prefix `0x03`. Encoding in [`transactions.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/transactions.py):

```python
def encode_transaction(tx: Transaction) -> LegacyTransaction | Bytes:
    ...
    elif isinstance(tx, BlobTransaction):
        return b"\x03" + rlp.encode(tx)
```

Decoding detects the type prefix and dispatches accordingly:

```python
def decode_transaction(tx: LegacyTransaction | Bytes) -> Transaction:
    if isinstance(tx, Bytes):
        ...
        elif tx[0] == 3:
            return rlp.decode_to(BlobTransaction, tx[1:])
```

Receipts are encoded with the matching prefix so that decoders can distinguish blob transaction receipts from other types:

```python
def encode_receipt(tx: Transaction, receipt: Receipt) -> Bytes | Receipt:
    if isinstance(tx, BlobTransaction):
        return b"\x03" + rlp.encode(receipt)
```

### 1.3 Signing Hash

The signing hash commits to all blob-specific fields — including `max_fee_per_blob_gas` and `blob_versioned_hashes` — but crucially **not** to the raw blob data, KZG commitments, or proofs. Those are consensus-layer concerns:

```python
def signing_hash_4844(tx: BlobTransaction) -> Hash32:
    return keccak256(
        b"\x03"
        + rlp.encode(
            (
                tx.chain_id,
                tx.nonce,
                tx.max_priority_fee_per_gas,
                tx.max_fee_per_gas,
                tx.gas,
                tx.to,
                tx.value,
                tx.data,
                tx.access_list,
                tx.max_fee_per_blob_gas,
                tx.blob_versioned_hashes, # hashes, not blobs
            )
        )
    )
```

### 1.4 The Network Wrapper: Where the Raw Blob Actually Lives

When a blob transaction propagates through the P2P mempool (via `PooledTransactions`), it is wrapped with its blob data for that journey:

```
rlp([tx_payload_body, blobs, commitments, proofs])
```

- `tx_payload_body` — the standard signed `BlobTransaction`, type-prefixed with `0x03`.
- `blobs` — the raw 131 072-byte arrays, one per versioned hash.
- `commitments` — 48-byte KZG commitments, one per blob.
- `proofs` — 48-byte KZG proofs, one per blob. Under Osaka’s EIP-7594 PeerDAS, this expands to 128 cell proofs per blob.

When the transaction is included in a block and served via `BlockBodies`, only the `tx_payload_body` is returned. The blobs are stripped and handed off to the consensus layer as a beacon block sidecar. At the block level, the execution layer holds nothing except the versioned hashes permanently recorded in the transaction trie.

---

## 2. KZG Commitments: From Blob Data to On-Chain Anchor

### 2.1 What a KZG Commitment Is

A blob is treated as a polynomial `p(x)` of degree ≤ 4095, whose 4096 coefficients are the blob’s field elements evaluated at 4096 roots of unity in the BLS12-381 prime field. The KZG commitment to this polynomial is an inner product against the Structured Reference String (SRS) — the output of Ethereum’s Powers of Tau trusted setup ceremony:

```
C = a₀·G + a₁·[τ]G + a₂·[τ²]G + ... + a₄₀₉₅·[τ⁴⁰⁹⁵]G
```

where `τ` is a secret destroyed after the ceremony, `G` is the BLS12-381 G1 generator, and `a₀..a₄₀₉₅` are the polynomial coefficients. The result `C` is a single 48-byte G1 curve point that uniquely and irreversibly encodes the entire blob polynomial. Because `τ` is unknown, it is computationally infeasible to reverse from `C` to `p(x)`.

### 2.2 `kzg_to_versioned_hash`: Compressing the Commitment to 32 Bytes

The EVM stack word is 32 bytes. A KZG commitment is 48 bytes. The function `kzg_to_versioned_hash` bridges this by hashing the commitment down and prepending a version byte:

```python
VERSIONED_HASH_VERSION_KZG = b"\x01"

def kzg_to_versioned_hash(commitment: KZGCommitment) -> VersionedHash:
    return VERSIONED_HASH_VERSION_KZG + sha256(commitment)[1:]
    #      ├─ 1 byte: version prefix (0x01 = KZG)
    #      └─ 31 bytes: tail of SHA256 of the 48-byte G1 point
```

The `0x01` prefix byte is a forward-compatibility version marker. If Ethereum ever transitions to a different commitment scheme — a post-quantum alternative, for example — a new version byte would be used, and the point-evaluation precompile would dispatch on it. Smart contracts written against this scheme do not need to change; they check the version byte and call the same precompile address.

The version check is enforced in [`fork.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py) inside `check_transaction`:

```python
for blob_versioned_hash in tx.blob_versioned_hashes:
    if blob_versioned_hash[0:1] != VERSIONED_HASH_VERSION_KZG:
        raise InvalidBlobVersionedHashError("invalid blob versioned hash")
```

This is a format check only. The execution layer does not verify that the hash was correctly derived from a real KZG commitment — that is performed by the consensus client against the blob sidecar before the block is accepted.

### 2.3 The BLS Modulus Constraint

The `BLS_MODULUS` is the prime that defines the BLS12-381 scalar field:

```python
BLS_MODULUS = 52435875175126190479447740508185965837690552500527637822603658699938581184513
```

Every one of the 4096 field elements in a blob must be strictly less than this value. This is not a soft constraint: the KZG polynomial arithmetic is defined modulo `BLS_MODULUS`, and any value ≥ `BLS_MODULUS` has no well-defined evaluation in the field. The point-evaluation precompile rejects such inputs.

In practice this means blob data is not arbitrary bytes. Binary data must be encoded so that the most-significant byte of each 32-byte field element is zero — sacrificing one byte per element and reducing effective capacity from 131 072 bytes to approximately 127 KB of usable payload.

---

## 3. Transaction Validation

The function `check_transaction` in [`fork.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py) runs blob-specific checks before any state changes:

### 3.1 Blob Gas Availability

```python
blob_gas_available = MAX_BLOB_GAS_PER_BLOCK - block_output.blob_gas_used

tx_blob_gas_used = calculate_total_blob_gas(tx)
if tx_blob_gas_used > blob_gas_available:
    raise BlobGasLimitExceededError("blob gas limit exceeded")
```

`MAX_BLOB_GAS_PER_BLOCK` is derived at runtime from the active `blobSchedule` as `GAS_PER_BLOB * blobSchedule.max` — in Osaka, `9 × 131 072 = 1 179 648` blob gas per block. The remaining capacity is tracked in `block_output.blob_gas_used`, which accumulates across all transactions in the block.

### 3.2 Blob Count and Versioned Hash Format

```python
if isinstance(tx, BlobTransaction):
    blob_count = len(tx.blob_versioned_hashes)
    if blob_count == 0:
        raise NoBlobDataError("no blob data in transaction")
    if blob_count > BLOB_COUNT_LIMIT:   # 6
        raise BlobCountExceededError(...)

    for blob_versioned_hash in tx.blob_versioned_hashes:
        if blob_versioned_hash[0:1] != VERSIONED_HASH_VERSION_KZG:
            raise InvalidBlobVersionedHashError("invalid blob versioned hash")
```

Each transaction must carry between 1 and 6 blobs. Every versioned hash must begin with `0x01`.

### 3.3 Blob Gas Fee Check

```python
blob_gas_price = calculate_blob_gas_price(block_env.excess_blob_gas)
if Uint(tx.max_fee_per_blob_gas) < blob_gas_price:
    raise InsufficientMaxFeePerBlobGasError("insufficient max fee per blob gas")
```

The sender’s stated maximum must cover the current blob base fee, computed from `excess_blob_gas` inherited from the parent block.

### 3.4 No Contract Creation

```python
if isinstance(tx, BlobTransaction):
    if not isinstance(tx.to, Address):
        raise TransactionTypeContractCreationError(tx)
```

Blob transactions cannot have an empty `to` field. Unlike `FeeMarketTransaction` where `to` may be `Bytes0 | Address`, `BlobTransaction.to` is typed as `Address` only — the constraint is structural, not just a runtime check.

### 3.5 Exception Types

Blob validation failures are raised as distinct typed exceptions defined in [`exceptions.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/exceptions.py):

```python
class BlobGasLimitExceededError(InvalidTransaction): ...
class InsufficientMaxFeePerBlobGasError(InvalidTransaction): ...
class InvalidBlobVersionedHashError(InvalidTransaction): ...
class NoBlobDataError(InvalidTransaction): ...
class BlobCountExceededError(InvalidTransaction): ...
```

---

## 4. Blob Gas Accounting

All blob gas functions live in [`vm/gas.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/gas.py).

### 4.1 Gas Per Blob

```python
GAS_PER_BLOB = U64(2**17)   # 131 072
```

Each blob in a transaction costs exactly `GAS_PER_BLOB` units of blob gas:

```python
def calculate_total_blob_gas(tx: Transaction) -> U64:
    if isinstance(tx, BlobTransaction):
        return GAS_PER_BLOB * U64(len(tx.blob_versioned_hashes))
    return U64(0)
```

A transaction carrying two blobs costs 262 144 blob gas, counted against the per-block cap. This is a distinct gas dimension from execution gas — blob gas has no interaction with the EIP-1559 execution base fee.

### 4.2 Blob Gas Price: `fake_exponential`

The blob gas price is computed from `excess_blob_gas` using a Taylor-series integer approximation of `e^x`:

```python
def fake_exponential(factor: Uint, numerator: Uint, denominator: Uint) -> Uint:
    """
    Approximates factor * e^(numerator/denominator).
    Used to price blob gas: as excess_blob_gas grows, price rises
    exponentially; as it falls, price falls exponentially.
    """
    i = Uint(1)
    output = Uint(0)
    numerator_accum = factor * denominator
    while numerator_accum > 0:
        output += numerator_accum
        numerator_accum = (numerator_accum * numerator) // (denominator * i)
        i += 1
    return output // denominator

MIN_BLOB_BASE_FEE = Uint(1)             # 1 wei minimum
BLOB_BASE_FEE_UPDATE_FRACTION = Uint(5007716)  # Osaka value (EIP-7892 blobSchedule)

def calculate_blob_gas_price(excess_blob_gas: U64) -> Uint:
    return Uint(
        fake_exponential(
            MIN_BLOB_BASE_FEE,
            Uint(excess_blob_gas),
            BLOB_BASE_FEE_UPDATE_FRACTION,
        )
    )
```

The exponential shape means small deviations from the target propagate only weak price signals, while sustained over-target usage drives the price up sharply — identical in design to EIP-1559’s base fee, but operating on a fully independent axis. Heavy calldata usage cannot raise blob fees; a blob demand spike cannot raise execution gas prices.

### 4.3 Data Fee and Fee Burning

The blob data fee charged to the sender:

```python
def calculate_data_fee(excess_blob_gas: U64, tx: Transaction) -> Uint:
    return Uint(calculate_total_blob_gas(tx)) * calculate_blob_gas_price(excess_blob_gas)
```

This fee is deducted from the sender’s balance upfront in `process_transaction` and is **burned** — it is never credited to the coinbase. Only the execution-gas priority fee is paid to the block producer:

```python
# In process_transaction (fork.py):
effective_gas_fee = tx.gas * effective_gas_price
blob_gas_fee = calculate_data_fee(block_env.excess_blob_gas, tx)

sender_balance_after_gas_fee = (
    Uint(sender_account.balance) - effective_gas_fee - blob_gas_fee
)
set_account_balance(block_env.state, sender, U256(sender_balance_after_gas_fee))

# ... transaction executes ...

# Only the priority fee goes to coinbase:
priority_fee_per_gas = effective_gas_price - block_env.base_fee_per_gas
transaction_fee = tx_gas_used_after_refund * priority_fee_per_gas
coinbase_balance += U256(transaction_fee)
```

The blob fee does not participate in EIP-7623 calldata floor logic either — that floor applies only to execution gas:

```python
# EIP-7623 floor: execution gas only
tx_gas_used_after_refund = max(tx_gas_used_after_refund, calldata_floor_gas_cost)

# Blob gas tracked entirely separately
block_output.blob_gas_used += tx_blob_gas_used
```

### 4.4 Excess Blob Gas and the EIP-7918 Reserve Price

`excess_blob_gas` is a running counter in the block header. Each block’s value is computed from the parent’s header:

```python
def calculate_excess_blob_gas(parent_header: Header) -> U64:
    blob_schedule = get_blob_schedule(parent_header.timestamp)
    target_blob_gas = GAS_PER_BLOB * blob_schedule.target   # 6 × 131 072

    parent_blob_gas = parent_header.excess_blob_gas + parent_header.blob_gas_used
    if parent_blob_gas < target_blob_gas:
        return U64(0)

    # EIP-7918: reserve price check
    # If the blob base fee is below BLOB_BASE_COST/GAS_PER_BLOB of the
    # execution base fee, execution fees dominate — stop decreasing excess.
    BLOB_BASE_COST = Uint(2**13)   # 8192
    current_blob_base_fee = calculate_blob_gas_price(parent_header.excess_blob_gas)
    reserve_price = BLOB_BASE_COST * Uint(parent_header.base_fee_per_gas)

    if reserve_price > GAS_PER_BLOB * current_blob_base_fee:
        # Execution-fee-led regime: excess grows with blob_gas_used but never decreases
        max_blob_gas = GAS_PER_BLOB * blob_schedule.max
        return U64(
            parent_header.excess_blob_gas
            + parent_header.blob_gas_used
            * (max_blob_gas - target_blob_gas)
            // max_blob_gas
        )
    else:
        return U64(parent_blob_gas - target_blob_gas)
```

Without this `if` branch (which is the pre-Osaka behaviour), `excess_blob_gas` would decrease whenever blocks are below target, driving the blob base fee toward 1 wei. During periods of high execution fees this would make blobs nearly free while node compute costs for KZG proof verification remain high. EIP-7918 prevents the fee from falling below `BLOB_BASE_COST * base_fee_per_gas / GAS_PER_BLOB` — a reserve price of `1/16` of the execution base fee expressed per blob gas unit.

---

## 5. Block Header Fields

The `Header` dataclass in [`blocks.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/blocks.py) carries two blob-related fields:

```python
@slotted_freezable
@dataclass
class Header:
    ...
    blob_gas_used: U64
    """Total blob gas consumed by all blob transactions in this block."""
    excess_blob_gas: U64
    """
    Running total of blob gas consumed in excess of target, carried from
    the parent block. The blob base fee for this block's transactions is
    a deterministic function of this single value.
    """
```

`excess_blob_gas` is the execution layer’s entire persistent blob-related state. Everything else about blobs — raw data, commitments, proofs — is managed by the consensus layer and pruned after approximately 18 days (4096 epochs). Header validation in `validate_header` confirms the field matches what `calculate_excess_blob_gas` produces from the parent:

```python
if header.excess_blob_gas != calculate_excess_blob_gas(parent_header):
    raise InvalidBlock
```

---

## 6. The Sharding Architecture: Execution vs. Consensus Separation

### 6.1 The Execution Layer Sees Only Hashes

The central architectural principle is enforced by structure, not policy. The `Block` type in [`blocks.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/blocks.py) contains:

```python
@slotted_freezable
@dataclass
class Block:
    header: Header
    transactions: Tuple[Bytes | LegacyTransaction, ...]
    ommers: Tuple[Header, ...]
    withdrawals: Tuple[Withdrawal, ...]
```

There is no `blob_sidecars` field. The state transition function `state_transition(chain, block)` in [`fork.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py) never receives raw blob data. Blob data is validated by the consensus client against the sidecar before the block is accepted; the execution layer is then handed a fully validated `Block` containing only versioned hashes in the transaction payloads.

### 6.2 `BlockEnvironment` and `BlockOutput`

The EVM context in [`vm/__init__.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/__init__.py) carries `excess_blob_gas` for opcode use:

```python
@dataclass
class BlockEnvironment:
    chain_id: U64
    state: State
    block_gas_limit: Uint
    block_hashes: List[Hash32]
    coinbase: Address
    number: Uint
    base_fee_per_gas: Uint
    time: U256
    prev_randao: Bytes32
    excess_blob_gas: U64          # read by BLOBBASEFEE opcode; used in fee calculations
    parent_beacon_block_root: Hash32
```

Accumulated blob gas across the block’s transactions is tracked separately in:

```python
@dataclass
class BlockOutput:
    block_gas_used: Uint
    ...
    blob_gas_used: U64 = U64(0)   # sum of calculate_total_blob_gas(tx) for all txs
```

### 6.3 `TransactionEnvironment`: Carrying Hashes Into the EVM

The versioned hashes are passed into the EVM via `TransactionEnvironment` in [`vm/__init__.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/__init__.py):

```python
@dataclass
class TransactionEnvironment:
    origin: Address
    gas_price: Uint
    gas: Uint
    access_list_addresses: Set[Address]
    access_list_storage_keys: Set[Tuple[Address, Bytes32]]
    transient_storage: TransientStorage
    blob_versioned_hashes: Tuple[VersionedHash, ...]
    authorizations: Tuple[Authorization, ...]
    index_in_block: Optional[Uint]
    tx_hash: Optional[Hash32]
```

This struct is populated in `process_transaction` in [`fork.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py):

```python
tx_env = vm.TransactionEnvironment(
    origin=sender,
    gas_price=effective_gas_price,
    gas=gas,
    access_list_addresses=access_list_addresses,
    access_list_storage_keys=access_list_storage_keys,
    transient_storage=TransientStorage(),
    blob_versioned_hashes=blob_versioned_hashes,
    authorizations=authorizations,
    index_in_block=index,
    tx_hash=get_transaction_hash(encode_transaction(tx)),
)
```

From here, `blob_versioned_hashes` flows through the EVM call tree as part of the message context, making it accessible to the `BLOBHASH` opcode at any call depth within the top-level transaction.

### 6.4 PeerDAS and the Osaka Networking Layer (EIP-7594)

Osaka introduces EIP-7594 (PeerDAS), which changes how blobs propagate on the consensus P2P network. In prior forks, consensus nodes downloaded complete blob sidecars. Under PeerDAS, each blob is split into 128 cells, and nodes only custody a random subset of columns, enabling data availability sampling without full downloads.

From the execution layer’s perspective, **nothing changes**. The `BlobTransaction` struct, versioned hashes, blob gas accounting, `BLOBHASH`, and the point-evaluation precompile are identical to Cancun. PeerDAS is a consensus-layer change, invisible to the EELS state transition code. The only execution-layer trace of PeerDAS is in the Engine API (`BlobsBundleV2`), which is the interface between consensus and execution clients — not part of the `ethereum/osaka` state transition module itself.

---

## 7. EVM Integration

### 7.1 `BLOBHASH` Opcode (`0x49`)

Defined in [`vm/instructions/environment.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/environment.py):

```python
GAS_BLOBHASH_OPCODE = Uint(3)

def blob_hash(evm: Evm) -> None:
    """
    BLOBHASH (0x49): Push versioned hash at index i onto the stack.
    Returns 32 zero bytes if index is out of range.
    """
    index = pop(evm.stack)
    charge_gas(evm, GAS_BLOBHASH_OPCODE)

    if int(index) < len(evm.message.tx_env.blob_versioned_hashes):
        blob_hash = evm.message.tx_env.blob_versioned_hashes[index]
    else:
        blob_hash = Bytes32(b"\x00" * 32)

    push(evm.stack, U256.from_be_bytes(blob_hash))
    evm.pc += Uint(1)
```

This is the EVM’s only window into blob space. It does not expose raw blob data — only the 32-byte versioned hash. Its canonical use is in an L2 inbox contract: when a sequencer submits a blob transaction calling the inbox, the contract executes `BLOBHASH(0)` to retrieve the versioned hash and stores it in contract storage. That stored value becomes the permanent on-chain anchor against which all future proof verifications are made.

### 7.2 Point-Evaluation Precompile (`0x0A`)

Defined in [`vm/precompiled_contracts/point_evaluation.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/point_evaluation.py):

```python
GAS_PRECOMPILE_POINT_EVALUATION = Uint(50000)
FIELD_ELEMENTS_PER_BLOB = 4096
BLS_MODULUS = 52435875175126190479447740508185965837690552500527637822603658699938581184513

def point_evaluation(evm: Evm) -> None:
    """
    Verify a KZG proof that p(z) == y for blob polynomial committed by C.

    Input (exactly 192 bytes):
        versioned_hash  [  0: 32] — 0x01 || sha256(commitment)[1:]
        z               [ 32: 64] — evaluation point (big-endian field element)
        y               [ 64: 96] — claimed value at z (big-endian field element)
        commitment      [ 96:144] — 48-byte KZG commitment (G1 point)
        proof           [144:192] — 48-byte KZG proof (G1 point)
    """
    data = evm.message.data
    if len(data) != 192:
        raise KZGProofError

    versioned_hash = data[:32]
    z = Bytes32(data[32:64])
    y = Bytes32(data[64:96])
    commitment = KZGCommitment(data[96:144])
    proof = Bytes48(data[144:192])

    charge_gas(evm, GAS_PRECOMPILE_POINT_EVALUATION)

    # Step 1: verify the commitment matches the on-chain versioned hash
    if kzg_commitment_to_versioned_hash(commitment) != versioned_hash:
        raise KZGProofError

    # Step 2: verify p(z) == y via BLS12-381 pairing check
    try:
        kzg_proof_verification = verify_kzg_proof(commitment, z, y, proof)
    except Exception as e:
        raise KZGProofError from e

    if not kzg_proof_verification:
        raise KZGProofError

    # Return FIELD_ELEMENTS_PER_BLOB || BLS_MODULUS (64 bytes)
    evm.output = Bytes(
        U256(FIELD_ELEMENTS_PER_BLOB).to_be_bytes32()
        + U256(BLS_MODULUS).to_be_bytes32()
    )
```

**Step 1 — commitment consistency.** `kzg_commitment_to_versioned_hash(commitment)` re-derives the versioned hash from the supplied 48-byte G1 point and checks it against the 32-byte `versioned_hash` argument. This cross-references the proof against what the inbox contract stored on-chain via `BLOBHASH` — ensuring the proof is for the correct blob.

**Step 2 — `verify_kzg_proof`.** This call delegates to the `c-kzg-4844` cryptographic library, which performs the BLS12-381 pairing check:

```
e(commitment - y·G₁, G₂) == e(proof, [τ]G₂ - z·G₂)
```

If this equation holds, the verifier has cryptographic certainty that the polynomial committed by `C` evaluates to `y` at point `z` — i.e., that a specific 32-byte chunk of the blob is exactly `y`. The check costs one pairing and is O(1) regardless of blob size, which is why the gas cost is a flat 50 000 regardless of the number of blobs in the transaction.

**Return values.** On success the precompile returns 64 bytes: `FIELD_ELEMENTS_PER_BLOB` (4096) and `BLS_MODULUS`. Returning the modulus allows smart contracts to read the active commitment parameters at runtime rather than hardcoding them — if a future commitment version uses a different modulus, the return value changes and calling contracts can adapt without being redeployed.

**How rollups use this.** An optimistic rollup fraud proof verifier calls `0x0A` with the versioned hash retrieved from the inbox contract, a commitment and proof for a specific field element `z`, and the claimed value `y` at that position. The precompile verifies the claim without the verifier ever accessing the full 128 KB blob. A ZK rollup uses the same mechanism to prove equivalence between its own internal commitment and the blob commitment.

The precompile is wired into the EVM dispatch table in [`precompiled_contracts/mapping.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/mapping.py). When the EVM encounters a `CALL` to `0x0A`, `process_message_call` in [`vm/interpreter.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/interpreter.py) routes directly to this function rather than interpreting bytecode.

---

## 8. End-to-End Execution Flow

```
1. CONSTRUCTION (off-chain)
   Sequencer packs L2 tx data into blob → calls blob_to_kzg_commitment()
   → derives versioned_hash = 0x01 || sha256(commitment)[1:]
   → constructs BlobTransaction with blob_versioned_hashes = (versioned_hash,)

2. P2P PROPAGATION
   Network wrapper: rlp([tx_payload_body, blobs, commitments, proofs])
   Receiving nodes: verify kzg_to_versioned_hash(commitments[i]) == blob_versioned_hashes[i]
                    verify blob matches commitment (verify_blob_kzg_proof)
   ← consensus layer; execution layer does not participate

3. BLOCK INCLUSION CHECK  [fork.py: check_transaction]
   ├── tx_blob_gas_used = GAS_PER_BLOB × len(blob_versioned_hashes) ≤ blob_gas_available?
   ├── each hash[0:1] == 0x01?
   ├── max_fee_per_blob_gas ≥ calculate_blob_gas_price(excess_blob_gas)?
   └── tx.to is not None?

4. FEE DEDUCTION  [fork.py: process_transaction]
   data_fee = calculate_total_blob_gas(tx) × calculate_blob_gas_price(excess_blob_gas)
   sender.balance -= execution_gas_fee + data_fee
   data_fee → burned (not sent to coinbase)

5. EVM CONTEXT SETUP  [fork.py → vm/__init__.py]
   TransactionEnvironment.blob_versioned_hashes = tx.blob_versioned_hashes
   (flows into every call frame; BLOBHASH reads from tx_env)

6. EVM EXECUTION  [vm/instructions/environment.py]
   inbox_contract executes BLOBHASH(0) → retrieves versioned_hash
   inbox_contract stores versioned_hash in storage  ← permanent on-chain anchor

7. BLOCK FINALISATION  [fork.py: apply_body]
   block_output.blob_gas_used += tx_blob_gas_used
   header.blob_gas_used  = block_output.blob_gas_used
   header.excess_blob_gas = calculate_excess_blob_gas(parent_header)
                          (EIP-7918 reserve price enforced here)

8. PROOF VERIFICATION  [precompiled_contracts/point_evaluation.py]
   (any time within ~18 days while blob sidecar exists)
   verifier calls 0x0A with (versioned_hash, z, y, commitment, proof):
   ├── kzg_commitment_to_versioned_hash(commitment) == versioned_hash?
   └── verify_kzg_proof(commitment, z, y, proof) → pairing check on BLS12-381
   ✓ proof: p(z) == y — field element z of the blob is confirmed to be y
```

---

## 9. Constants Reference

| Constant | Value | Location |
| --- | --- | --- |
| `GAS_PER_BLOB` | `131 072` (`2**17`) | [`vm/gas.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/gas.py) |
| `BLOB_SCHEDULE_TARGET` | `6` blobs/block | blobSchedule (EIP-7892) |
| `BLOB_SCHEDULE_MAX` | `9` blobs/block | blobSchedule (EIP-7892) |
| `MAX_BLOB_GAS_PER_BLOCK` | `1 179 648` (`9 × 131 072`) | derived at runtime |
| `BLOB_COUNT_LIMIT` | `6` per transaction | [`fork.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py) |
| `VERSIONED_HASH_VERSION_KZG` | `0x01` | [`fork.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py) |
| `MIN_BLOB_BASE_FEE` | `1` wei | [`vm/gas.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/gas.py) |
| `BLOB_BASE_FEE_UPDATE_FRACTION` | `5 007 716` | blobSchedule (EIP-7892) |
| `BLOB_BASE_COST` | `8 192` (`2**13`) | [`vm/gas.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/gas.py) — EIP-7918 |
| `GAS_BLOBHASH_OPCODE` | `3` | [`vm/gas.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/gas.py) |
| `GAS_PRECOMPILE_POINT_EVALUATION` | `50 000` | [`vm/gas.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/gas.py) |
| `FIELD_ELEMENTS_PER_BLOB` | `4 096` | [`vm/precompiled_contracts/point_evaluation.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/point_evaluation.py) |
| `BLS_MODULUS` | `52435875175126190479447740508185965837690552500527637822603658699938581184513` | [`vm/precompiled_contracts/point_evaluation.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/point_evaluation.py) |