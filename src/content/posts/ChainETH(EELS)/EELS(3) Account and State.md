---
title: EELS(3) Account and State
published: 2026-03-31
pinned: false
description: Examine how Ethereum manages state data under the hood
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

1. [**Preface**](#preface)
2. [**Introduction**](#introduction)
3. [**Account Structure**](#1-account-structure)
4. [**World State**](#2-world-state)
5. [**Account Operations**](#3-account-operations)
6. [**Contract Storage**](#4-contract-storage)
7. [**Code Storage**](#5-code-storage)
8. [**Snapshot Mechanism**](#6-snapshot-mechanism)
9. [**Transient Storage**](#7-transient-storage)

---

## Preface

This article is part of a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**.

**What is EELS?** EELS is a Python reference implementation of the Ethereum execution client, designed with a focus on readability and clarity. It serves as a programmer-friendly, up-to-date successor to the original Yellow Paper and is the primary tool for prototyping new Ethereum Improvement Proposals (EIPs). EELS provides complete protocol snapshots at each fork and rendered diffs between them.

**Scope and Implementation** Note that EELS does not implement the JSON-RPC API or P2P networking. To validate blocks, it requires an external RPC provider to fetch data, which EELS then processes and stores in a local database.

**Version Reference** The code analysis in this article is based on the **Osaka** fork:
[ethereum/execution-specs (Osaka)](https://github.com/ethereum/execution-specs/tree/forks/amsterdam/src/ethereum/forks/osaka)

---

## Introduction

This chapter describes how Ethereum stores and manages state in the Osaka fork of the Execution Layer Specification (EELS). It covers the account model, the world state trie, contract storage, code storage, the snapshot mechanism for atomic reverts, and transient storage.

---

## 1. Account Structure

The [`Account`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/state.py) dataclass is defined in the shared `ethereum` package and reused across all forks:

```python
# src/ethereum/state.py

@slotted_freezable
@dataclass
class Account:
    """
    State associated with an address.
    """
    nonce: Uint
    balance: U256
    code_hash: Hash32
```

Notice that `storage_root` is **not** a field of `Account`. Storage is maintained separately in per-account storage tries (see [§2](about:blank#2-world-state)), and the storage root is only computed and injected at RLP-encoding time.

The sentinel value for accounts with no code is:

```python
EMPTY_CODE_HASH = keccak256(b"")
# = 0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470
```

The zero-value account is:

```python
EMPTY_ACCOUNT = Account(
    nonce=Uint(0),
    balance=U256(0),
    code_hash=EMPTY_CODE_HASH,
)
```

### EIP-7702: EOA Code Delegation

After EIP-7702, an EOA may delegate its execution to a smart contract. In this case, the account’s raw code is set to the byte string `0xef0100 || address`, where `0xef0100` is the delegation marker and `address` is the 20-byte target contract address. The `code_hash` stored on the account therefore becomes `keccak256(0xef0100 || address)`.

### Account Encoding

Although `storage_root` is not stored in the `Account` struct, it is included when the account is RLP-encoded for insertion into the state trie. This is handled by [`encode_account`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork_types.py) in the Osaka fork:

```python
# src/ethereum/forks/osaka/fork_types.py

def encode_account(raw_account_data: Account, storage_root: Bytes) -> Bytes:
    """
    Encode `Account` dataclass.

    Storage is not stored in the `Account` dataclass, so `Accounts` cannot be
    encoded without providing a storage root.
    """
    return rlp.encode(
        (
            raw_account_data.nonce,
            raw_account_data.balance,
            storage_root,
            raw_account_data.code_hash,
        )
    )
```

The RLP-encoded tuple follows the canonical order: `(nonce, balance, storage_root, code_hash)`.

---

## 2. World State

### 2.1 The `State` Dataclass

The global state is defined in [`state.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) of the Osaka fork. It is a collection of Merkle Patricia Tries (MPTs) and auxiliary stores:

```python
# src/ethereum/forks/osaka/state.py

@dataclass
class State:
    """
    Contains all information that is preserved between transactions.
    """
    _main_trie: Trie[Address, Optional[Account]] = field(
        default_factory=lambda: Trie(secured=True, default=None)
    )
    _storage_tries: Dict[Address, Trie[Bytes32, U256]] = field(
        default_factory=dict
    )
    _snapshots: List[
        Tuple[
            Trie[Address, Optional[Account]],
            Dict[Address, Trie[Bytes32, U256]],
        ]
    ] = field(default_factory=list)
    created_accounts: Set[Address] = field(default_factory=set)
    _code_store: Dict[Hash32, Bytes] = field(
        default_factory=dict, compare=False
    )
```

| Field | Type | Purpose |
| --- | --- | --- |
| `_main_trie` | `Trie[Address, Optional[Account]]` | Maps every address to its account. Uses a secured trie, meaning keys are hashed with `keccak256`. |
| `_storage_tries` | `Dict[Address, Trie[Bytes32, U256]]` | Per-contract storage. Each entry is its own secured trie mapping slot keys to `U256` values. |
| `_snapshots` | `List[Tuple[...]]` | Stack of (main trie, storage tries) snapshots for transaction-level rollback. |
| `created_accounts` | `Set[Address]` | Accounts created in the current transaction. Used by `get_storage_original()` (EIP-2200) and `SELFDESTRUCT` (EIP-6780). |
| `_code_store` | `Dict[Hash32, Bytes]` | Content-addressed bytecode store. Keyed by `keccak256(bytecode)`. Not part of the MPT. |

### 2.2 State Root

The root of `_main_trie` is the `state_root` committed to in the block header. It is computed by [`state_root`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py):

```python
def state_root(state: State) -> Root:
    assert not state._snapshots  # must not be called mid-transaction

    def get_storage_root(address: Address) -> Root:
        return storage_root(state, address)

    return root(state._main_trie, get_storage_root=get_storage_root)
```

Computing the state root requires all open transactions to be committed or rolled back first, since the trie is in a mutable intermediate state while snapshots exist. For each account in the main trie, the trie’s `encode_node` callback invokes `encode_account(account, storage_root)`, which is how the per-account `storage_root` is dynamically computed and embedded at serialization time.

---

## 3. Account Operations

### 3.1 Reading Accounts

There are two account read functions with different semantics:

[**`get_account`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) — always returns an `Account`, substituting `EMPTY_ACCOUNT` for non-existent addresses:

```python
def get_account(state: State, address: Address) -> Account:
    account = get_account_optional(state, address)
    if isinstance(account, Account):
        return account
    else:
        return EMPTY_ACCOUNT
```

[**`get_account_optional`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) — returns `None` for non-existent addresses:

```python
def get_account_optional(state: State, address: Address) -> Optional[Account]:
    account = trie_get(state._main_trie, address)
    return account
```

Use `get_account_optional` when the distinction between a non-existent account and `EMPTY_ACCOUNT` matters (e.g., existence checks). Use `get_account` when a default zero-valued account is an acceptable substitute (e.g., balance reads).

### 3.2 Writing Accounts

[**`set_account`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) sets or deletes an account entry in the main trie. Passing `None` removes the account record, but does **not** delete storage; use `destroy_account` for full removal:

```python
def set_account(
    state: State, address: Address, account: Optional[Account]
) -> None:
    trie_set(state._main_trie, address, account)
```

[**`modify_state`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) applies a mutation function to an account in place, then auto-destroys the account if it becomes empty (zero nonce, empty code, zero balance):

```python
def modify_state(
    state: State, address: Address, f: Callable[[Account], None]
) -> None:
    """
    Modify an `Account` in the `State`. If, after modification, the account
    exists and has zero nonce, empty code, and zero balance, it is destroyed.
    """
    set_account(state, address, modify(get_account(state, address), f))

    account = get_account_optional(state, address)
    account_exists_and_is_empty = (
        account is not None
        and account.nonce == Uint(0)
        and account.code_hash == EMPTY_CODE_HASH
        and account.balance == 0
    )

    if account_exists_and_is_empty:
        destroy_account(state, address)
```

### 3.3 Account Existence Predicates

The Osaka state module exposes several predicates for checking account state:

| Function | Returns `True` when |
| --- | --- |
| [`account_exists`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) | `get_account_optional` returns a non-`None` value. |
| [`is_account_alive`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) | Account exists in the trie and is not equal to `EMPTY_ACCOUNT`. |
| [`account_has_code_or_nonce`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) | `nonce != 0` or `code_hash != EMPTY_CODE_HASH`. |
| [`account_has_storage`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) | Address has an entry in `_storage_tries`. |

```python
def account_exists(state: State, address: Address) -> bool:
    return get_account_optional(state, address) is not None

def is_account_alive(state: State, address: Address) -> bool:
    account = get_account_optional(state, address)
    return account is not None and account != EMPTY_ACCOUNT

def account_has_code_or_nonce(state: State, address: Address) -> bool:
    account = get_account(state, address)
    return account.nonce != Uint(0) or account.code_hash != EMPTY_CODE_HASH

def account_has_storage(state: State, address: Address) -> bool:
    return address in state._storage_tries
```

### 3.4 ETH Transfers

[**`move_ether`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) transfers a `U256` amount of wei from one address to another:

```python
def move_ether(
    state: State,
    sender_address: Address,
    recipient_address: Address,
    amount: U256,
) -> None:
    def reduce_sender_balance(sender: Account) -> None:
        if sender.balance < amount:
            raise AssertionError
        sender.balance -= amount

    def increase_recipient_balance(recipient: Account) -> None:
        recipient.balance += amount

    modify_state(state, sender_address, reduce_sender_balance)
    modify_state(state, recipient_address, increase_recipient_balance)
```

The balance check is performed inside the closure; if `sender.balance < amount`, the `AssertionError` propagates up and the enclosing transaction is expected to be rolled back by the caller.

---

## 4. Contract Storage

### 4.1 Structure

Each contract has an independent storage trie in `state._storage_tries`. Keys and values are both 32-byte quantities: keys are `Bytes32` (hashed by the secured trie) and values are `U256`. Unset slots implicitly hold `U256(0)`.

### 4.2 Reading Storage

[**`get_storage`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) returns `U256(0)` for any address without a storage trie, or for any unset slot within an existing trie:

```python
def get_storage(state: State, address: Address, key: Bytes32) -> U256:
    trie = state._storage_tries.get(address)
    if trie is None:
        return U256(0)

    value = trie_get(trie, key)

    assert isinstance(value, U256)
    return value
```

### 4.3 Writing Storage

[**`set_storage`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) creates the storage trie on first write. Setting a slot to `U256(0)` deletes that entry from the trie, and if the trie becomes entirely empty it is removed from `_storage_tries`:

```python
def set_storage(
    state: State, address: Address, key: Bytes32, value: U256
) -> None:
    assert trie_get(state._main_trie, address) is not None

    trie = state._storage_tries.get(address)
    if trie is None:
        trie = Trie(secured=True, default=U256(0))
        state._storage_tries[address] = trie
    trie_set(trie, key, value)
    if trie._data == {}:
        del state._storage_tries[address]
```

The leading `assert` enforces that storage can only be written to accounts that exist in the main trie.

### 4.4 Storage Root

[**`storage_root`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) computes the MPT root for a given account’s storage trie. It returns `EMPTY_TRIE_ROOT` for accounts with no storage:

```python
def storage_root(state: State, address: Address) -> Root:
    assert not state._snapshots
    if address in state._storage_tries:
        return root(state._storage_tries[address])
    else:
        return EMPTY_TRIE_ROOT
```

This function is called internally by `state_root` to embed each account’s storage root into its RLP encoding.

### 4.5 Pre-Transaction Original Values

For SSTORE gas pricing (EIP-2200), the EVM must know the value of a slot *before* the current transaction began. [**`get_storage_original`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) reads this from the outermost snapshot:

```python
def get_storage_original(state: State, address: Address, key: Bytes32) -> U256:
    # For accounts created in the current transaction, preexisting storage is ignored.
    if address in state.created_accounts:
        return U256(0)

    _, original_trie = state._snapshots[0]
    original_account_trie = original_trie.get(address)

    if original_account_trie is None:
        original_value = U256(0)
    else:
        original_value = trie_get(original_account_trie, key)

    assert isinstance(original_value, U256)
    return original_value
```

`state._snapshots[0]` is the snapshot taken at the start of the transaction (the outermost frame). Accounts in `created_accounts` return `U256(0)` unconditionally, because no storage could have existed before the account was created.

---

## 5. Code Storage

### 5.1 Separation from Account Trie

Contract bytecode is stored in `State._code_store`, a plain `Dict[Hash32, Bytes]`, separate from the MPT. The `Account` struct holds only the `keccak256` hash of the bytecode. This design has several benefits:

- **Size**: Keeps account trie nodes small; bytecode can be up to 24 KB.
- **Deduplication**: Multiple contracts with identical bytecode share a single `_code_store` entry.
- **Performance**: Hash table lookups are O(1), compared to O(log n) for trie reads.

### 5.2 Storing and Reading Code

[**`store_code`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) writes bytecode to `_code_store` and returns its hash. Empty bytecode is not inserted:

```python
def store_code(state: State, code: Bytes) -> Hash32:
    code_hash = keccak256(code)
    if code_hash != EMPTY_CODE_HASH:
        state._code_store[code_hash] = code
    return code_hash
```

[**`get_code`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) retrieves bytecode by hash, returning `b""` for `EMPTY_CODE_HASH` without a store lookup:

```python
def get_code(state: State, code_hash: Hash32) -> Bytes:
    if code_hash == EMPTY_CODE_HASH:
        return b""
    return state._code_store[code_hash]
```

`get_code` is also available as a method on the `State` object (`state.get_code(code_hash)`), with identical behaviour.

### 5.3 Setting Code on an Account

[**`set_code`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) stores bytecode and updates the account’s `code_hash` atomically via `modify_state`:

```python
def set_code(state: State, address: Address, code: Bytes) -> None:
    code_hash = keccak256(code)
    if code_hash != EMPTY_CODE_HASH:
        state._code_store[code_hash] = code

    def write_code(sender: Account) -> None:
        sender.code_hash = code_hash

    modify_state(state, address, write_code)
```

---

## 6. Snapshot Mechanism

### 6.1 Purpose

State modifications during a transaction must be atomic: if execution fails or reverts, all changes must be undone. The EELS achieves this by maintaining a **snapshot stack** inside `State._snapshots`. Every call to `begin_transaction` pushes a deep copy of the current trie state; `rollback_transaction` restores it.

Snapshots can be nested, enabling inner call frames (e.g., `CALL`, `CREATE`) to revert independently of the outer frame.

### 6.2 Transaction Lifecycle

The three snapshot operations are defined in [`state.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py). Each takes both `state` and `transient_storage` as arguments, because both must be snapshotted together:

**`begin_transaction`** — push a snapshot of both the account trie and all storage tries, and independently snapshot transient storage:

```python
def begin_transaction(
    state: State, transient_storage: TransientStorage
) -> None:
    state._snapshots.append(
        (
            copy_trie(state._main_trie),
            {k: copy_trie(t) for (k, t) in state._storage_tries.items()},
        )
    )
    transient_storage._snapshots.append(
        {k: copy_trie(t) for (k, t) in transient_storage._tries.items()}
    )
```

**`commit_transaction`** — discard the most recent snapshot, making all changes since `begin_transaction` permanent. Clears `created_accounts` when the outermost transaction completes:

```python
def commit_transaction(
    state: State, transient_storage: TransientStorage
) -> None:
    state._snapshots.pop()
    if not state._snapshots:
        state.created_accounts.clear()

    transient_storage._snapshots.pop()
```

**`rollback_transaction`** — restore state from the most recent snapshot, discarding all changes since `begin_transaction`:

```python
def rollback_transaction(
    state: State, transient_storage: TransientStorage
) -> None:
    state._main_trie, state._storage_tries = state._snapshots.pop()
    if not state._snapshots:
        state.created_accounts.clear()

    transient_storage._tries = transient_storage._snapshots.pop()
```

### 6.3 Nested Snapshots

The snapshot stack supports arbitrary nesting. A revert at an inner frame only restores to the snapshot pushed by that frame’s `begin_transaction`, leaving outer frames unaffected:

```
begin_transaction()          # snapshot[0] pushed  (transaction start)
  begin_transaction()        # snapshot[1] pushed  (inner CALL)
    ...
  rollback_transaction()     # snapshot[1] restored (inner CALL reverted)
commit_transaction()         # snapshot[0] discarded (transaction committed)
```

### 6.4 `created_accounts` and EIP-6780

[**`mark_account_created`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) adds an address to `State.created_accounts`:

```python
def mark_account_created(state: State, address: Address) -> None:
    state.created_accounts.add(address)
```

This marker is used in two places:

1. **`get_storage_original`**: Returns `U256(0)` for any account created in the current transaction, because no pre-transaction storage can logically exist.
2. **EIP-6780 (`SELFDESTRUCT`)**: The opcode fully destroys a contract only if it was created in the same transaction; otherwise it only transfers its balance.

Notably, the marker is not removed if account creation reverts. This is safe because a reverted account cannot have had code before creation and therefore cannot call `get_storage_original`.

---

## 7. Transient Storage

### 7.1 Overview

Transient storage (introduced in EIP-1153 at Cancun) provides per-address key-value storage that is cleared at the end of every transaction. It is accessed via the `TLOAD` and `TSTORE` opcodes and costs 100 gas per access (always warm). Unlike persistent storage, it is not included in the state root.

### 7.2 The `TransientStorage` Dataclass

[`TransientStorage`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) mirrors the structure of `State` for its domain:

```python
@dataclass
class TransientStorage:
    """
    Contains all information that is preserved between message calls
    within a transaction.
    """
    _tries: Dict[Address, Trie[Bytes32, U256]] = field(default_factory=dict)
    _snapshots: List[Dict[Address, Trie[Bytes32, U256]]] = field(
        default_factory=list
    )
```

`TransientStorage` is a **separate object** from `State`. It is not a field of `State`; instead, it is threaded through the `begin_transaction`, `commit_transaction`, and `rollback_transaction` calls alongside `State` (see [§6.2](about:blank#62-transaction-lifecycle)).

### 7.3 Operations

[**`get_transient_storage`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py):

```python
def get_transient_storage(
    transient_storage: TransientStorage, address: Address, key: Bytes32
) -> U256:
    trie = transient_storage._tries.get(address)
    if trie is None:
        return U256(0)

    value = trie_get(trie, key)

    assert isinstance(value, U256)
    return value
```

[**`set_transient_storage`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/state.py) creates the per-address trie on first write and removes it when the last entry is deleted:

```python
def set_transient_storage(
    transient_storage: TransientStorage,
    address: Address,
    key: Bytes32,
    value: U256,
) -> None:
    trie = transient_storage._tries.get(address)
    if trie is None:
        trie = Trie(secured=True, default=U256(0))
        transient_storage._tries[address] = trie
    trie_set(trie, key, value)
    if trie._data == {}:
        del transient_storage._tries[address]
```

### 7.4 Comparison with Persistent Storage

| Property | Persistent Storage | Transient Storage |
| --- | --- | --- |
| Lifetime | Until explicitly deleted | Cleared after each transaction |
| Opcodes | `SLOAD` / `SSTORE` | `TLOAD` / `TSTORE` |
| Gas cost | 2100 (cold) / 100 (warm) | 100 (always warm) |
| Included in state root | Yes | No |
| Snapshot support | Yes | Yes |