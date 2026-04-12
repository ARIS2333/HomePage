---
title: EELS(8) System Transactions
published: 2026-04-05
pinned: false
description: System transaction and EL->CL request
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

- [**Preface**](#preface)
- [**Introduction**](#introduction)
- [**What Is a System Transaction?**](#1-what-is-a-system-transaction)
- [**Execution Semantics**](#2-execution-semantics)
- [**Implementation**](#3-implementation--process_system_transaction)
- [**Checked vs. Unchecked Variants**](#4-checked-vs-unchecked-variants)
- [**System Contracts in Osaka**](#5-system-contracts-in-osaka)
- [**Block-Level Execution Order**](#6-block-level-execution-order--apply_body)
- [**EL → CL Requests**](#7-el--cl-requests-eip-7685)
- [**Deposit Requests**](#8-deposit-requests-eip-6110)
- [**Withdrawal Requests**](#9-withdrawal-requests-eip-7002)
- [**Consolidation Requests**](#10-consolidation-requests-eip-7251)
- [**Request Hashing**](#11-request-hashing--compute_requests_hash)
- [**Block Header Commitment**](#12-block-header-commitment)

## Preface

This article is part of a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**.

**What is EELS?** EELS is a Python reference implementation of the Ethereum execution client, designed with a focus on readability and clarity. It serves as a programmer-friendly, up-to-date successor to the original Yellow Paper and is the primary tool for prototyping new Ethereum Improvement Proposals (EIPs). EELS provides complete protocol snapshots at each fork and rendered diffs between them.

**Scope and Implementation** Note that EELS does not implement the JSON-RPC API or P2P networking. To validate blocks, it requires an external RPC provider to fetch data, which EELS then processes and stores in a local database.

**Version Reference** The code analysis in this article is based on the **Osaka** fork:
[ethereum/execution-specs (Osaka)](https://github.com/ethereum/execution-specs/tree/forks/amsterdam/src/ethereum/forks/osaka)

---

## Introduction

This chapter covers how the Ethereum Execution Layer initiates privileged calls to system contracts during block processing, and how the results of those calls — together with deposit contract logs — are assembled into the EL → CL request pipeline for the Osaka fork.

---

## 1. What Is a System Transaction?

A **system transaction** is a call to a smart contract that is made by the protocol itself during block processing — not submitted by any external user. System transactions implement protocol-level behaviour through deployed contracts rather than hardcoded EVM logic, making the behaviour upgradable and auditable on-chain.

System transactions differ from user transactions in every semantically significant way:

| Property | User transaction | System transaction |
| --- | --- | --- |
| Origin / caller | Sender EOA, recovered from signature | `SYSTEM_ADDRESS = 0xfffffffffffffffffffffffffffffffffffffffe` |
| Counted against block gas limit | Yes | No |
| Fee mechanics | EIP-1559 base fee + tip apply | No ETH moves; gas price is `block_env.base_fee_per_gas` but no payment is collected |
| Nonce requirement | Must match sender nonce | None |
| Receipt generated | Yes | No |
| Execution failure | Reverts sender, gas charged | Either silent no-op or `InvalidBlock` depending on variant |
| Timing within block | Among user transactions | Before user transactions (BeaconRoots, HistoryStorage), or after (request contracts) |

---

## 2. Execution Semantics

A system transaction is an ordinary EVM call constructed to pass through the same `process_message_call` path as user calls. What makes it privileged is the identity of its caller. The EVM itself imposes no special semantics on `SYSTEM_ADDRESS`; the privilege is purely in the *inputs* provided — specifically, that `origin` and `caller` are both set to `SYSTEM_ADDRESS`, and that the execution is not bounded by the block gas limit.

Key properties of the constructed call:

- **`caller = SYSTEM_ADDRESS`**: Smart contracts that check `msg.sender` can grant or restrict capabilities based on this value.
- **`should_transfer_value = False`**: No ETH is transferred even if `value` is non-zero (it is always `U256(0)`).
- **`is_static = False`**: System contracts may write state.
- **`gas = SYSTEM_TRANSACTION_GAS = Uint(30_000_000)`**: A generous fixed limit. Consumed gas is not deducted from the block’s available gas.
- **No access list pre-warming**: `accessed_addresses` and `accessed_storage_keys` both start as empty sets.
- **`index_in_block = None` and `tx_hash = None`**: System transactions have no position in the block’s transaction list and produce no receipt.

---

## 3. Implementation — `process_system_transaction`

[`process_system_transaction`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html#ethereum.forks.osaka.fork.process_system_transaction:0) is the core primitive defined in [`fork.py`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html). It takes the code to execute explicitly as a parameter — callers are responsible for resolving the contract’s code from state before calling it:

```python
def process_system_transaction(
    block_env: BlockEnvironment,
    target_address: Address,
    system_contract_code: Bytes,
    data: Bytes,
) -> MessageCallOutput:
    tx_env = vm.TransactionEnvironment(
        origin=SYSTEM_ADDRESS,
        gas_price=block_env.base_fee_per_gas,
        gas=SYSTEM_TRANSACTION_GAS,
        access_list_addresses=set(),
        access_list_storage_keys=set(),
        transient_storage=TransientStorage(),
        blob_versioned_hashes=(),
        authorizations=(),
        index_in_block=None,
        tx_hash=None,
    )

    system_tx_message = Message(
        block_env=block_env,
        tx_env=tx_env,
        caller=SYSTEM_ADDRESS,
        target=target_address,
        gas=SYSTEM_TRANSACTION_GAS,
        value=U256(0),
        data=data,
        code=system_contract_code,
        depth=Uint(0),
        current_target=target_address,
        code_address=target_address,
        should_transfer_value=False,
        is_static=False,
        accessed_addresses=set(),
        accessed_storage_keys=set(),
        disable_precompiles=False,
        parent_evm=None,
    )

    system_tx_output = process_message_call(system_tx_message)

    return system_tx_output
```

The result is a `MessageCallOutput`. Callers examine `system_tx_output.error` and `system_tx_output.return_data` according to the variant (checked or unchecked) they used.

---

## 4. Checked vs. Unchecked Variants

The two public-facing wrappers differ in how they handle missing code or execution failure.

### `process_unchecked_system_transaction`

[`process_unchecked_system_transaction`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html#ethereum.forks.osaka.fork.process_unchecked_system_transaction:0) reads the contract code from state and passes it directly to `process_system_transaction`. If the account has no code, the call proceeds with empty bytecode — the EVM will execute an immediate `STOP` and return an empty output. No error is raised; the block remains valid. This is the appropriate variant for system contracts that may not yet be deployed (e.g., before their introducing fork activates) or whose absence should be tolerated.

### `process_checked_system_transaction`

[`process_checked_system_transaction`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html#ethereum.forks.osaka.fork.process_checked_system_transaction:0) additionally:

1. Verifies the target account has code. If `len(system_contract_code) == 0`, it raises `InvalidBlock`.
2. Checks the execution output for errors. If `output.error` is set, it raises `InvalidBlock`.

This variant is used for contracts whose presence and successful execution are protocol invariants — the block is invalid if they are absent or fail.

| Variant | Missing code | Execution failure | Use case |
| --- | --- | --- | --- |
| Unchecked | Silent no-op | Ignored | BeaconRoots (EIP-4788), HistoryStorage (EIP-2935) |
| Checked | `InvalidBlock` | `InvalidBlock` | WithdrawalRequest (EIP-7002), ConsolidationRequest (EIP-7251) |

---

## 5. System Contracts in Osaka

All system contract addresses are constants defined in [`fork.py`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html):

```python
SYSTEM_ADDRESS = hex_to_address("0xfffffffffffffffffffffffffffffffffffffffe")

BEACON_ROOTS_ADDRESS = hex_to_address(
    "0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02"
)
HISTORY_STORAGE_ADDRESS = hex_to_address(
    "0x0000F90827F1C53a10cb7A02335B175320002935"
)
WITHDRAWAL_REQUEST_PREDEPLOY_ADDRESS = hex_to_address(
    "0x00000961Ef480Eb55e80D19ad83579A64c007002"
)
CONSOLIDATION_REQUEST_PREDEPLOY_ADDRESS = hex_to_address(
    "0x0000BBdDc7CE488642fb579F8B00f3a590007251"
)
```

| EIP | Contract | Address | Call variant | Purpose |
| --- | --- | --- | --- | --- |
| EIP-4788 | BeaconRoots | `0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02` | Unchecked | Store parent beacon block root in ring buffer |
| EIP-2935 | HistoryStorage | `0x0000F90827F1C53a10cb7A02335B175320002935` | Unchecked | Store parent block hash for `BLOCKHASH` extension |
| EIP-7002 | WithdrawalRequest | `0x00000961Ef480Eb55e80D19ad83579A64c007002` | Checked | Accumulate and return validator withdrawal requests |
| EIP-7251 | ConsolidationRequest | `0x0000BBdDc7CE488642fb579F8B00f3a590007251` | Checked | Accumulate and return validator consolidation requests |

All four contracts are written in EVM assembly and deployed at their addresses via Nick’s method — a keyless deployment scheme that produces a deterministic contract address without requiring a funded deployer.

---

## 6. Block-Level Execution Order — `apply_body`

System transactions are called inside `apply_body` in [`fork.py`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html), which is itself called by [`state_transition`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/fork.py.html#ethereum.forks.osaka.fork.state_transition:0). The execution order within a block is:

```
1.  process_unchecked_system_transaction → BEACON_ROOTS_ADDRESS
      calldata = block_env.parent_beacon_block_root (32 bytes)

2.  process_unchecked_system_transaction → HISTORY_STORAGE_ADDRESS
      calldata = keccak256(rlp(parent_header)) — the parent block hash

3.  For each user transaction in block.transactions:
        process_transaction(block_env, block_output, tx, index)

4.  For each withdrawal in block.withdrawals:
        process_withdrawal(block_env.state, withdrawal)

5.  Collect EL→CL requests:
    a. deposit_requests  = parse_deposit_requests(block_output)
    b. withdrawal_data   = process_checked_system_transaction(
                               WITHDRAWAL_REQUEST_PREDEPLOY_ADDRESS, data=b"")
    c. consolidation_data = process_checked_system_transaction(
                               CONSOLIDATION_REQUEST_PREDEPLOY_ADDRESS, data=b"")

    block_output.requests = [
        DEPOSIT_REQUEST_TYPE + deposit_requests,      # if non-empty
        WITHDRAWAL_REQUEST_TYPE + withdrawal_data,    # if non-empty
        CONSOLIDATION_REQUEST_TYPE + consolidation_data,  # if non-empty
    ]
```

The request-collecting calls in step 5 happen **after** all user transactions, ensuring they see the full block’s state — in particular, the full set of deposits that occurred during user transactions.

---

## 7. EL → CL Requests (EIP-7685)

[EIP-7685](https://eips.ethereum.org/EIPS/eip-7685) establishes a general-purpose framework for the execution layer to signal structured information to the consensus layer. Before this framework, each cross-layer mechanism (deposits, exits, etc.) had its own ad-hoc signalling channel. EIP-7685 unifies them under a single abstraction: a typed, ordered list of **requests** produced per block.

A request is a byte string with a one-byte type prefix followed by its payload. The type prefixes active in Osaka are defined in [`requests.py`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/requests.py.html):

```python
DEPOSIT_REQUEST_TYPE      = b"\x00"   # EIP-6110
WITHDRAWAL_REQUEST_TYPE   = b"\x01"   # EIP-7002
CONSOLIDATION_REQUEST_TYPE = b"\x02"  # EIP-7251
```

Requests are collected into `block_output.requests: List[Bytes]` during `apply_body`. Each entry in the list is `type_byte + raw_payload_bytes` for a given request type — all individual requests of the same type are concatenated into a single entry. The list is then hashed and committed to the block header as `requests_hash`.

---

## 8. Deposit Requests (EIP-6110)

Deposit requests originate from the **deposit contract** at:

```python
DEPOSIT_CONTRACT_ADDRESS = hex_to_address(
    "0x00000000219ab540356cbb839cbe05303d7705fa"
)
```

Unlike the other two request types, deposits are not collected via a system call. Instead, [`parse_deposit_requests`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/requests.py.html#ethereum.forks.osaka.requests.parse_deposit_requests:0) in [`requests.py`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/requests.py.html) scans the receipts trie of the completed block for matching log entries:

```python
def parse_deposit_requests(block_output: BlockOutput) -> Bytes:
    deposit_requests: Bytes = b""
    for key in block_output.receipt_keys:
        receipt = trie_get(block_output.receipts_trie, key)
        assert receipt is not None
        decoded_receipt = decode_receipt(receipt)
        for log in decoded_receipt.logs:
            if log.address == DEPOSIT_CONTRACT_ADDRESS:
                if (
                    len(log.topics) > 0
                    and log.topics[0] == DEPOSIT_EVENT_SIGNATURE_HASH
                ):
                    request = extract_deposit_data(log.data)
                    deposit_requests += request
    return deposit_requests
```

For each matching log, [`extract_deposit_data`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/requests.py.html#ethereum.forks.osaka.requests.extract_deposit_data:0) decodes the ABI-encoded event data and validates its structure. The event log data is 576 bytes (`DEPOSIT_EVENT_LENGTH`), containing five ABI-encoded dynamic fields with explicit offsets and sizes:

| Field | Offset constant | Size |
| --- | --- | --- |
| `pubkey` (BLS public key) | `PUBKEY_OFFSET = 160` | 48 bytes |
| `withdrawal_credentials` | `WITHDRAWAL_CREDENTIALS_OFFSET = 256` | 32 bytes |
| `amount` (in Gwei, little-endian) | `AMOUNT_OFFSET = 320` | 8 bytes |
| `signature` (BLS signature) | `SIGNATURE_OFFSET = 384` | 96 bytes |
| `index` (deposit index) | `INDEX_OFFSET = 512` | 8 bytes |

If any offset or size is incorrect, `extract_deposit_data` raises `InvalidBlock`. On success it returns the five fields concatenated in order: `pubkey + withdrawal_credentials + amount + signature + index` (192 bytes per deposit). This raw byte string is directly suitable for CL consumption.

---

## 9. Withdrawal Requests (EIP-7002)

Withdrawal requests allow a validator’s withdrawal credential owner — an EL address — to trigger a partial or full withdrawal from the beacon chain via an on-chain contract, without requiring the validator’s BLS key.

After all user transactions execute, `apply_body` calls the WithdrawalRequest predeploy with empty calldata:

```python
withdrawal_output = process_checked_system_transaction(
    block_env=block_env,
    target_address=WITHDRAWAL_REQUEST_PREDEPLOY_ADDRESS,
    data=b"",
)
withdrawal_data = withdrawal_output.return_data
```

The contract accumulated withdrawal requests submitted by users during the block’s user transactions (users call the contract directly, paying a fee that the contract enforces). The system call with empty calldata triggers the contract to dequeue and return the accumulated requests as a raw byte string. Each individual withdrawal request is 76 bytes:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 20 bytes | `source_address` (EL address that triggered the request) |
| 20 | 48 bytes | `validator_pubkey` (BLS48 public key) |
| 68 | 8 bytes | `amount` in Gwei (`0` = full exit) |

Because this is a **checked** call, any failure — including the contract not being deployed — raises `InvalidBlock`.

---

## 10. Consolidation Requests (EIP-7251)

Consolidation requests allow the owner of a validator’s withdrawal credentials to merge two validators into one, combining their effective balances.

Identically to withdrawals, `apply_body` calls the ConsolidationRequest predeploy after all user transactions:

```python
consolidation_output = process_checked_system_transaction(
    block_env=block_env,
    target_address=CONSOLIDATION_REQUEST_PREDEPLOY_ADDRESS,
    data=b"",
)
consolidation_data = consolidation_output.return_data
```

Each individual consolidation request is 116 bytes:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 20 bytes | `source_address` (EL address that triggered the request) |
| 20 | 48 bytes | `source_pubkey` (validator to consolidate from) |
| 68 | 48 bytes | `target_pubkey` (validator to consolidate into) |

Again, this is a **checked** call; missing code or execution failure raises `InvalidBlock`.

---

## 11. Request Hashing — `compute_requests_hash`

After the three request byte strings are assembled, they are hashed by [`compute_requests_hash`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/requests.py.html#ethereum.forks.osaka.requests.compute_requests_hash:0) defined in [`requests.py`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/requests.py.html):

```python
def compute_requests_hash(requests: List[Bytes]) -> Bytes:
    m = sha256()
    for request in requests:
        m.update(sha256(request).digest())
    return m.digest()
```

The algorithm is: hash each entry individually with SHA-256, then feed all the resulting 32-byte digests sequentially into a single outer SHA-256 hasher, and return its digest. In pseudocode:

```
requests_hash = SHA256(SHA256(request_0) || SHA256(request_1) || SHA256(request_2))
```

Several design properties follow from this construction. SHA-256 is used rather than Keccak-256 because the consensus layer natively uses SHA-256 for all its hashing, making cross-layer verification consistent and efficient. The two-level structure (hash-then-chain) prevents length-extension attacks and ensures that any change to any single request, or any change to the set ordering, produces a different root. An empty request list (no entries in `block_output.requests`) produces the SHA-256 of the empty string.

The return type is `Bytes` (a 32-byte digest), not `Hash32` — this matches the block header field type.

---

## 12. Block Header Commitment

After `apply_body` returns, `state_transition` computes:

```python
requests_hash = compute_requests_hash(block_output.requests)
```

and verifies it against the block header:

```python
if requests_hash != block.header.requests_hash:
    raise InvalidBlock
```

`requests_hash` is a field of the block header introduced by EIP-7685. It commits the entire ordered set of EL → CL requests for this block. The CL reads the requests themselves through the Engine API’s `engine_getPayloadV*` family of endpoints, which include the request list alongside the execution payload. The CL then processes each request type according to its own rules — verifying BLS signatures, checking validator indices, enforcing rate limits, and so on — using `requests_hash` in the header as a binding commitment.

The three request types always appear in type-byte order within `block_output.requests` when non-empty: deposits (`0x00`) first, then withdrawals (`0x01`), then consolidations (`0x02`). Empty request types produce no entry in the list — they are omitted entirely rather than contributing a zero-length payload.