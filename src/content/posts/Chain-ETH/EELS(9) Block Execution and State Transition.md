---
title: EELS(9) Block Execution and State Transition
published: 2026-04-10
pinned: false
description: Ethereum block execution pipeline, orchestrates all the things introduced previously
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

- [**Preface**](#preface)
- [**Introduction**](#introduction)
- [**1. The BlockChain Object**](#1-the-blockchain-object)
- [**2. Key Constants**](#2-key-constants-osaka)
- [**3. State Transition**](#3-state_transition--the-outer-gate)
- [**4. Header Validation**](#4-header-validation)
- [**5. Block Execution**](#5-apply_body--block-execution)
- [**6. How Tries Are Built During Execution**](#6-how-tries-are-built-during-execution)
- [**7. Output Validation**](#7-output-validation--cryptographic-commitments)

## Preface

This article is part of a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**.

**What is EELS?** EELS is a Python reference implementation of the Ethereum execution client, designed with a focus on readability and clarity. It serves as a programmer-friendly, up-to-date successor to the original Yellow Paper and is the primary tool for prototyping new Ethereum Improvement Proposals (EIPs). EELS provides complete protocol snapshots at each fork and rendered diffs between them.

**Scope and Implementation** Note that EELS does not implement the JSON-RPC API or P2P networking. To validate blocks, it requires an external RPC provider to fetch data, which EELS then processes and stores in a local database.

**Version Reference** The code analysis in this article is based on the **Osaka** fork:
[ethereum/execution-specs (Osaka)](https://github.com/ethereum/execution-specs/tree/forks/amsterdam/src/ethereum/forks/osaka)

---

## Introduction

This chapter brings together all previously introduced components — accounts, state, transactions, and the EVM — into the full block execution pipeline. It is centred on two functions in [`fork.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py): `apply_body`, which executes a block, and `state_transition`, which validates the result against the block header.

---

## 1. The BlockChain Object

[`BlockChain`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py) is the top-level object that both `state_transition` and `apply_body` operate on. It bundles the live world state with the recent block history needed by the `BLOCKHASH` opcode:

```python
# src/ethereum/forks/osaka/fork.py

@dataclass
class BlockChain:
    """History and current state of the block chain."""
    blocks: List[Block]   # recent blocks (last 255 retained)
    state: State          # live world state trie
    chain_id: U64         # EIP-155 network identifier
```

`chain.state` is the single mutable `State` object that all transactions in a block modify in place. When `state_transition` builds the `BlockEnvironment`, it passes `chain.state` by reference — so all EVM operations within `apply_body` write directly into the canonical world state.

---

## 2. Key Constants (Osaka)

```python
# src/ethereum/forks/osaka/fork.py

SYSTEM_ADDRESS               = hex_to_address("0xfffffffffffffffffffffffffffffffffffffffe")
BEACON_ROOTS_ADDRESS         = hex_to_address("0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02")
HISTORY_STORAGE_ADDRESS      = hex_to_address("0x0000F90827F1C53a10cb7A02335B175320002935")
WITHDRAWAL_REQUEST_PREDEPLOY_ADDRESS  = hex_to_address("0x00000961Ef480Eb55e80D19ad83579A64c007002")
CONSOLIDATION_REQUEST_PREDEPLOY_ADDRESS = hex_to_address("0x0000BBdDc7CE488642fb579F8B00f3a590007251")

SYSTEM_TRANSACTION_GAS  = Uint(30_000_000)

# EIP-7934 (Osaka): RLP-encoded block size limit
MAX_BLOCK_SIZE    = 10_485_760    # 10 MB
SAFETY_MARGIN     = 2_097_152     #  2 MB
MAX_RLP_BLOCK_SIZE = MAX_BLOCK_SIZE - SAFETY_MARGIN  # 8 MB effective limit

BLOB_COUNT_LIMIT  = 6
```

`MAX_RLP_BLOCK_SIZE` is a hard limit introduced in Osaka by EIP-7934: the RLP encoding of the entire block must not exceed 8 MB. `state_transition` enforces this before any other processing.

---

## 3. `state_transition` — The Outer Gate

[`state_transition(chain, block)`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py) is the highest-level function in the execution layer. It is the single point that takes an incoming block and either appends it to the canonical chain or raises `InvalidBlock`.

```python
# src/ethereum/forks/osaka/fork.py

def state_transition(chain: BlockChain, block: Block) -> None:
    # 1. Block size check (EIP-7934, Osaka)
    if len(rlp.encode(block)) > MAX_RLP_BLOCK_SIZE:
        raise InvalidBlock("Block rlp size exceeds MAX_RLP_BLOCK_SIZE")

    # 2. Header validation
    validate_header(chain, block.header)
    if block.ommers != ():
        raise InvalidBlock

    # 3. Build execution environment
    block_env = vm.BlockEnvironment(
        chain_id=chain.chain_id,
        state=chain.state,
        block_gas_limit=block.header.gas_limit,
        block_hashes=get_last_256_block_hashes(chain),
        coinbase=block.header.coinbase,
        number=block.header.number,
        base_fee_per_gas=block.header.base_fee_per_gas,
        time=block.header.timestamp,
        prev_randao=block.header.prev_randao,
        excess_blob_gas=block.header.excess_blob_gas,
        parent_beacon_block_root=block.header.parent_beacon_block_root,
    )

    # 4. Execute the block body
    block_output = apply_body(
        block_env=block_env,
        transactions=block.transactions,
        withdrawals=block.withdrawals,
    )

    # 5. Compute roots from execution output
    block_state_root   = state_root(block_env.state)
    transactions_root  = root(block_output.transactions_trie)
    receipt_root       = root(block_output.receipts_trie)
    block_logs_bloom   = logs_bloom(block_output.block_logs)
    withdrawals_root   = root(block_output.withdrawals_trie)
    requests_hash      = compute_requests_hash(block_output.requests)

    # 6. Verify roots against header commitments
    if block_output.block_gas_used != block.header.gas_used:
        raise InvalidBlock(
            f"{block_output.block_gas_used} !={block.header.gas_used}"
        )
    if transactions_root != block.header.transactions_root:
        raise InvalidBlock
    if block_state_root != block.header.state_root:
        raise InvalidBlock
    if receipt_root != block.header.receipt_root:
        raise InvalidBlock
    if block_logs_bloom != block.header.bloom:
        raise InvalidBlock
    if withdrawals_root != block.header.withdrawals_root:
        raise InvalidBlock
    if block_output.blob_gas_used != block.header.blob_gas_used:
        raise InvalidBlock
    if requests_hash != block.header.requests_hash:
        raise InvalidBlock

    # 7. Append to chain
    chain.blocks.append(block)
    if len(chain.blocks) > 255:
        # Protocol requires only the last 255 blocks (for BLOCKHASH opcode).
        # Production clients retain more for reorg handling.
        chain.blocks = chain.blocks[-255:]
```

The structure is deliberately sequential: pre-flight checks → execution → root computation → root verification → append. No state is committed until all roots match.

---

## 4. Header Validation

[`validate_header(chain, header)`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py) performs structural validation of the incoming header against the parent before any execution takes place. The key checks are:

- **Parent linkage**: `header.parent_hash == keccak256(rlp.encode(parent_header))`. Enforces that the block is a direct child of the current chain tip.
- **Block number**: `header.number == parent_header.number + 1`. Blocks must be strictly sequential.
- **Timestamp**: `header.timestamp > parent_header.timestamp`. Time must advance.
- **Gas limit**: checked by `check_gas_limit`, which ensures the new gas limit is within `parent_gas_limit ± (parent_gas_limit // GAS_LIMIT_ADJUSTMENT_FACTOR)` and above `GAS_LIMIT_MINIMUM`. This caps gas limit drift to 1/1024 per block.
- **Post-Merge invariants**: `difficulty == 0`, `nonce == b"\x00" * 8`, `ommers_hash == EMPTY_OMMER_HASH`. These fields were retired with the Merge.
- **Base fee (EIP-1559)**: `header.base_fee_per_gas` must match the value computed by `calculate_base_fee_per_gas(block_gas_limit, parent_gas_limit, parent_gas_used, parent_base_fee_per_gas)`. The base fee adjusts by up to 12.5% per block based on how much gas the parent used relative to its target (50% of the gas limit).
- **Excess blob gas (EIP-4844)**: `header.excess_blob_gas` must match the value computed from the parent’s `excess_blob_gas` and `blob_gas_used`.
- **Extra data**: `len(header.extra_data) <= 32`.

Header validation is intentionally kept independent of execution: it only verifies field consistency with the parent header, not the correctness of execution results.

---

## 5. `apply_body` — Block Execution

[`apply_body`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py) is responsible for executing the full block body and returning a [`BlockOutput`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/__init__.py) that accumulates all outputs.

> Executes a block. Many of the contents of a block are stored in data structures called tries. There is a transactions trie which is similar to a ledger of the transactions stored in the current block. There is also a receipts trie which stores the results of executing a transaction, like the post state and gas used. This function creates and executes the block that is to be added to the chain.
> 

```python
# src/ethereum/forks/osaka/fork.py

def apply_body(
    block_env: vm.BlockEnvironment,
    transactions: Tuple[Union[LegacyTransaction, Bytes], ...],
    withdrawals: Tuple[Withdrawal, ...],
) -> vm.BlockOutput:
```

`BlockOutput` is initialised with empty tries and zero counters, and is progressively filled by each of the four phases below:

```python
# src/ethereum/forks/osaka/vm/__init__.py

@dataclass
class BlockOutput:
    block_gas_used: Uint = Uint(0)
    transactions_trie: Trie[Bytes, Optional[Bytes | LegacyTransaction]] = ...
    receipts_trie: Trie[Bytes, Optional[Bytes | Receipt]] = ...
    receipt_keys: Tuple[Bytes, ...] = ...
    block_logs: Tuple[Log, ...] = ...
    withdrawals_trie: Trie[Bytes, Optional[Bytes | Withdrawal]] = ...
    blob_gas_used: U64 = U64(0)
    requests: List[Bytes] = ...
```

### Phase 1: System Transactions (EIP-4788 and EIP-2935)

Before any user transaction is processed, two system calls run against predeploy contracts. Both are dispatched via `process_unchecked_system_transaction`, which wraps `process_message_call` with `SYSTEM_ADDRESS` as both caller and origin, with `SYSTEM_TRANSACTION_GAS = 30_000_000` gas, and silently ignores any failure:

```python
# EIP-4788: store parent beacon block root in the ring buffer contract
process_unchecked_system_transaction(
    block_env=block_env,
    target_address=BEACON_ROOTS_ADDRESS,
    data=block_env.parent_beacon_block_root,
)

# EIP-2935: store parent block hash in the history storage contract
process_unchecked_system_transaction(
    block_env=block_env,
    target_address=HISTORY_STORAGE_ADDRESS,
    data=block_env.block_hashes[-1],  # the parent hash
)
```

These calls mutate `block_env.state` (the live world state) but do not contribute to `block_output.transactions_trie` or any of the output tries. They are purely state-modifying side effects that execute before the first user transaction.

### Phase 2: User Transactions

Transactions are decoded and executed strictly sequentially. There is no parallelism in the base protocol. Each transaction sees the world state as modified by all preceding transactions — including the system transactions above:

```python
for i, tx in enumerate(map(decode_transaction, transactions)):
    process_transaction(block_env, block_output, tx, Uint(i))
```

`decode_transaction` unwraps typed transactions encoded as raw bytes (EIP-2718 envelope). For each transaction, `process_transaction` performs validation, constructs an EVM `Message`, invokes `process_message_call`, applies gas refunds, pays the block proposer’s priority fee, and — critically — inserts both the transaction and its receipt into the respective tries inside `block_output`. This is covered in detail in [§6](about:blank#6-how-tries-are-built-during-execution).

### Phase 3: Validator Withdrawals

Consensus-layer withdrawals are applied as direct balance increments, bypassing EVM execution entirely:

```python
process_withdrawals(block_env, block_output, withdrawals)
```

Inside `process_withdrawals`, each withdrawal increases the recipient’s balance via `increase_balance(state, withdrawal.address, withdrawal.amount * GWEI_TO_WEI)` and inserts the withdrawal into `block_output.withdrawals_trie` keyed by its index. No code is executed, no gas is charged, and the withdrawal cannot fail — its validity was established by the consensus layer.

### Phase 4: General-Purpose Requests (EIP-7685)

```python
process_general_purpose_requests(
    block_env=block_env,
    block_output=block_output,
)
```

This phase collects execution-layer requests destined for the consensus layer, specifically:

- **Deposits (EIP-6110)**: scanned from `block_output.block_logs` emitted during user transactions.
- **Withdrawal requests (EIP-7002)**: collected by calling the `WITHDRAWAL_REQUEST_PREDEPLOY_ADDRESS` contract via `process_checked_system_transaction`. This variant raises `InvalidBlock` if the contract code is absent or the call fails.
- **Consolidation requests (EIP-7251)**: collected similarly from `CONSOLIDATION_REQUEST_PREDEPLOY_ADDRESS`.

All collected request bytes are appended to `block_output.requests`, which is later hashed into `requests_hash` by `state_transition`.

### Phase 5: Return

`apply_body` returns the fully populated `block_output`. At this point the world state (`block_env.state`) reflects all changes from all four phases.

---

## 6. How Tries Are Built During Execution

The tries inside `BlockOutput` are not computed after execution — they are built incrementally by `process_transaction` and `process_withdrawals` as the block executes. Understanding this is key to seeing how the execution layer actually uses MPT in practice.

### Transactions Trie

For each transaction at index `i`, `process_transaction` inserts the RLP-encoded transaction into `block_output.transactions_trie`:

```python
trie_set(
    block_output.transactions_trie,
    rlp.encode(Uint(index)),    # key: RLP-encoded transaction index
    encode_transaction(tx),     # value: type-prefixed RLP encoding of the tx
)
```

The key is the RLP encoding of the transaction’s position in the block. The value is the full typed transaction encoding. `root(block_output.transactions_trie)` at the end of `state_transition` is then compared against `block.header.transactions_root` — if they differ, the block was constructed with a different transaction set than what was executed.

### Receipts Trie

After `process_message_call` returns, `process_transaction` constructs a `Receipt` from the execution result and inserts it at the same index:

```python
trie_set(
    block_output.receipts_trie,
    rlp.encode(Uint(index)),
    encode_receipt(receipt),
)
```

A `Receipt` records the cumulative gas used after this transaction, the status (success or failure), and the logs emitted. The logs are also accumulated into `block_output.block_logs` for bloom filter construction. `root(block_output.receipts_trie)` verifies that receipts — and therefore execution outcomes — match the header’s `receipt_root` claim.

### Withdrawals Trie

`process_withdrawals` inserts each withdrawal at its sequential index:

```python
trie_set(
    block_output.withdrawals_trie,
    rlp.encode(Uint(index)),
    rlp.encode(withdrawal),
)
```

### World State Trie

The world state trie (`block_env.state._main_trie`) is modified live by every transaction. Account mutations — balance changes, nonce increments, code deployments, storage writes — are applied atomically within each `process_transaction` call via the snapshot mechanism. After all four phases complete, `state_root(block_env.state)` traverses the updated trie to produce `block_state_root`, which is checked against `block.header.state_root`.

Because `block_env.state` is `chain.state` (passed by reference in `state_transition`), a successfully validated block leaves the world state permanently updated — there is no separate “apply” step.

---

## 7. Output Validation — Cryptographic Commitments

After `apply_body` returns, `state_transition` independently recomputes six roots and two scalar values, and verifies all eight against the block header. The block proposer must have pre-computed these same values when constructing the block; any discrepancy indicates either a proposer error or a fraud attempt.

```python
# Recompute
block_state_root  = state_root(block_env.state)
transactions_root = root(block_output.transactions_trie)
receipt_root      = root(block_output.receipts_trie)
block_logs_bloom  = logs_bloom(block_output.block_logs)
withdrawals_root  = root(block_output.withdrawals_trie)
requests_hash     = compute_requests_hash(block_output.requests)

# Verify all eight
if block_output.block_gas_used != block.header.gas_used:       raise InvalidBlock(...)
if transactions_root != block.header.transactions_root:        raise InvalidBlock
if block_state_root  != block.header.state_root:               raise InvalidBlock
if receipt_root      != block.header.receipt_root:             raise InvalidBlock
if block_logs_bloom  != block.header.bloom:                    raise InvalidBlock
if withdrawals_root  != block.header.withdrawals_root:         raise InvalidBlock
if block_output.blob_gas_used != block.header.blob_gas_used:   raise InvalidBlock
if requests_hash     != block.header.requests_hash:            raise InvalidBlock
```

The table below explains what each commitment represents and why it is necessary:

| Field | Computed from | What it commits to | Consequence if removed |
| --- | --- | --- | --- |
| `gas_used` | Sum of all transaction gas costs | Total computational work | Proposers could lie about gas consumed, breaking fee markets and gas limit enforcement. |
| `transactions_root` | `root(block_output.transactions_trie)` | The exact set and order of included transactions | Proposers could include different transactions than the ones executed, enabling double-inclusion or omission attacks. |
| `state_root` | `state_root(block_env.state)` | The entire world state after applying the block | Any undetected state corruption (wrong balances, wrong storage, wrong code) would be accepted. This is the most critical commitment. |
| `receipt_root` | `root(block_output.receipts_trie)` | Execution outcomes and cumulative gas for every transaction | Clients could not verify whether a transaction succeeded or what logs it emitted, breaking EVM finality guarantees for dApps. |
| `bloom` | `logs_bloom(block_output.block_logs)` | A Bloom filter over all log topics and addresses | Light clients rely on the bloom for efficient log filtering without downloading full receipts. |
| `withdrawals_root` | `root(block_output.withdrawals_trie)` | The set of validator withdrawals applied | Consensus-layer withdrawals could be omitted or fabricated, breaking validator exit flows. |
| `blob_gas_used` | Sum of blob gas across blob transactions | Total blob gas consumed in the block | Blob fee market (EIP-4844) accounting would be broken; `excess_blob_gas` for the next block would be computed incorrectly. |
| `requests_hash` | `compute_requests_hash(block_output.requests)` | The set of EL→CL requests (deposits, withdrawals, consolidations) | Deposit processing, validator exits, and validator consolidations on the consensus layer could be manipulated. |

**The underlying guarantee**: every verifying node independently re-executes the block and recomputes all eight values. Because the EVM is deterministic — given the same prior state and the same transactions, every node produces the same outputs — consensus is achieved without any node trusting another’s results. The block header acts as a cryptographic summary: proposers commit to execution results before the block is broadcast, and verifiers confirm these commitments independently.