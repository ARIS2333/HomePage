---
title: EELS(3) Q&A Notes
published: 2026-04-01
pinned: false
description: Some QA that might be helpful.
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

1. [What is `get_storage_original` and why does `sstore` need it?](#14-what-is-get_storage_original-and-why-does-sstore-need-it)
2. [What does `state._snapshots[0]` return?](#15-what-does-state_snapshots0-return)
3. [How does `state._snapshots` work?](#16-how-does-state_snapshots-work)
4. [Why does `commit_transaction` call `state._snapshots.pop()`?](#17-why-does-commit_transaction-call-state_snapshotspop)
5. [Isn't copying the entire trie into memory every time impractical?](#18-isnt-copying-the-entire-trie-into-memory-every-time-impractical)
6. [Is `commit_transaction` called only when the outer call succeeds?](#19-is-commit_transaction-called-only-when-the-outer-call-succeeds)

---

## 1. What is `get_storage_original` and why does `sstore` need it?

### The Problem: SSTORE Gas Pricing Depends on Three Values

In modern Ethereum (post EIP-2200), the gas cost of an `SSTORE` depends on **three** values, not just the current one:

| Value | What it means |
|-------|--------------|
| `original_value` | The value in this slot **before the current transaction started** |
| `current_value` | The value in this slot **right now** (possibly modified by earlier SSTOREs in this tx) |
| `new_value` | The value we're **about to write** |

This is necessary because a single transaction can call `SSTORE` on the same slot multiple times, and the protocol needs to charge/refund gas fairly based on the **net effect** of all those writes.

### How `get_storage_original` Works

The implementation:

```python
def get_storage_original(state: State, address: Address, key: Bytes32) -> U256:
    """
    Get the original value in a storage slot i.e. the value before the current
    transaction began. This function reads the value from the snapshots taken
    before executing the transaction.
    """
    # In the transaction where an account is created, its preexisting storage
    # is ignored.
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

**Step 1: Check if the account was created in this transaction.**

```python
if address in state.created_accounts:
    return U256(0)
```

If the contract was just created in this transaction, it couldn't have had any pre-existing storage. Return 0 immediately.

**Step 2: Look at the outermost snapshot.**

```python
_, original_trie = state._snapshots[0]
```

Recall the snapshot stack. When `begin_transaction` is called at the start of a transaction, it pushes a deep copy of all storage tries onto `state._snapshots`. Index `[0]` is always the **outermost** snapshot — the state as it existed before this transaction began.

```
state._snapshots = [
    snapshot_0,   # ← this is the pre-transaction state (what we want)
    snapshot_1,   # snapshot from an inner CALL frame
    snapshot_2,   # snapshot from a deeper CALL frame
    ...
]
```

**Step 3: Find the storage trie for this address in the snapshot.**

```python
original_account_trie = original_trie.get(address)
```

The snapshot contains a full copy of `_storage_tries` — the dict mapping addresses to storage tries. Look up the address.

**Step 4: Read the slot value from the snapshot's trie.**

```python
if original_account_trie is None:
    original_value = U256(0)
else:
    original_value = trie_get(original_account_trie, key)
```

If the contract had no storage trie in the snapshot, the slot was never set → 0. Otherwise, read the slot directly from the snapshot's trie (not from the live, possibly-modified trie).

---

### How `sstore` Uses All Three Values

Look at the `sstore` function:

```python
original_value = get_storage_original(state, evm.message.current_target, key)
current_value  = get_storage(state, evm.message.current_target, key)
# new_value comes from the stack
```

Here's a concrete scenario to see why all three are needed:

#### Scenario: A transaction toggles a slot three times

```
Slot starts at: 100

SSTORE(key, 200)    # original=100, current=100, new=200
SSTORE(key, 0)      # original=100, current=200, new=0
SSTORE(key, 100)    # original=100, current=0,   new=100
```

**First SSTORE** — setting a changed value for the first time:
- `original=100, current=100, new=200`
- `original == current != new` and `original != 0`
- Cost: `GAS_COLD_STORAGE_WRITE - GAS_COLD_STORAGE_ACCESS = 2900`
- No refund yet

**Second SSTORE** — clearing the slot:
- `original=100, current=200, new=0`
- `original != 0, current != 0, new == 0` → first clear
- Cost: `GAS_WARM_ACCESS = 100` (slot was already accessed)
- Refund: `+4800`

**Third SSTORE** — restoring to original value:
- `original=100, current=0, new=100`
- `original != 0, current == 0` → reversing an earlier clear
- Refund: `-4800` (cancel the previous refund)
- `original == new` → slot restored to original
- Refund: `+2800` (5000 - 2100 - 100)
- Cost: `GAS_WARM_ACCESS = 100`

**Net result:** The transaction wrote three times but ended up with the original value. Without `original_value`, the protocol couldn't distinguish "restored to original" from "normal write." The refund logic ensures you only pay for the **net change** to state, not the intermediate churn.

---

### Without `get_storage_original`: The Gas Exploit

Before EIP-2200, SSTORE had a flat cost. An attacker could do:

```
SSTORE(key, 0)    # clear slot → get 15000 gas refund
SSTORE(key, 100)  # set it back → pay 20000 gas
```

Net gas: +5000. Net state change: nothing. But the attacker got 15000 refund credits to offset gas spent elsewhere in the transaction — effectively **free state manipulation**.

EIP-2200 fixes this by tracking `original_value`. In the second SSTORE above, the protocol sees `original == new`, recognizes the slot is being restored, and adjusts refunds accordingly. The net refund for a no-op is zero.

---

### Summary

| Function | Returns | Source |
|----------|---------|--------|
| `get_storage()` | Current value (possibly modified this tx) | Live `_storage_tries` |
| `get_storage_original()` | Value before this tx started | `state._snapshots[0]` — the outermost snapshot |
| `new_value` | Value being written | Stack operand |

`get_storage_original` is the memory mechanism for SSTORE gas pricing. It lets the protocol compare where a slot started, where it is now, and where it's going — and charge gas based on the net effect, not the intermediate noise.

---

## 2. What does `state._snapshots[0]` return?

`state._snapshots` is a **list of tuples**, where each tuple contains a deep copy of the entire state at a specific moment.

```python
# From the State dataclass definition:
_snapshots: List[
    Tuple[
        Trie[Address, Optional[Account]],          # copy of _main_trie
        Dict[Address, Trie[Bytes32, U256]],        # copy of _storage_tries
    ]
] = field(default_factory=list)
```

So `state._snapshots[0]` returns:

```python
(
    Trie(secured=True, default=None, _data={...}),   # a COPY of the main trie as it was at snapshot[0]
    {                                                 # a COPY of _storage_tries as it was at snapshot[0]
        Address(0x1234...): Trie(secured=True, default=U256(0), _data={...}),
        Address(0x5678...): Trie(secured=True, default=U256(0), _data={...}),
    }
)
```

In `get_storage_original`:

```python
_, original_trie = state._snapshots[0]
```

This unpacks the tuple, ignoring the first element (the copied main trie) and keeping the second element (the copied storage tries dict). That `original_trie` is a snapshot of `_storage_tries` as it existed when `begin_transaction` was first called at the start of the transaction.

---

## 3. How does `state._snapshots` work?

The snapshot stack supports **nested atomic transactions**. Every time the EVM enters a call frame (`CALL`, `DELEGATECALL`, `STATICCALL`, `CREATE`, `CREATE2`), it calls `begin_transaction`. When the call returns successfully, it calls `commit_transaction`. When the call fails/reverts, it calls `rollback_transaction`.

### `begin_transaction` — push a snapshot

```python
def begin_transaction(state: State, transient_storage: TransientStorage) -> None:
    state._snapshots.append(
        (
            copy_trie(state._main_trie),                                  # deep copy of account trie
            {k: copy_trie(t) for (k, t) in state._storage_tries.items()}, # deep copy of every storage trie
        )
    )
    transient_storage._snapshots.append(
        {k: copy_trie(t) for (k, t) in transient_storage._tries.items()}
    )
```

This does a **deep copy** of:
1. The entire main trie (all accounts)
2. Every storage trie (all contract storage)

The copies are independent objects. Any subsequent mutation to the live tries does **not** affect the copies in `_snapshots`.

### `commit_transaction` — discard a snapshot

```python
def commit_transaction(state: State, transient_storage: TransientStorage) -> None:
    state._snapshots.pop()          # throw away the snapshot
    if not state._snapshots:
        state.created_accounts.clear()

    transient_storage._snapshots.pop()
```

The call succeeded. The changes are permanent. We **discard** the most recent snapshot. We don't need it anymore.

### `rollback_transaction` — restore from a snapshot

```python
def rollback_transaction(state: State, transient_storage: TransientStorage) -> None:
    state._main_trie, state._storage_tries = state._snapshots.pop()  # restore the snapshot
    if not state._snapshots:
        state.created_accounts.clear()

    transient_storage._tries = transient_storage._snapshots.pop()
```

The call failed or reverted. We **replace** the live state with the snapshot. All changes made since `begin_transaction` are discarded.

---

### Concrete walkthrough: nested calls

Let's trace a real execution:

```
Transaction from Alice → Contract A → Contract B
```

#### Before the transaction starts

```
state._main_trie = {
    alice: Account(nonce=5, balance=1ETH, ...),
    contractA: Account(nonce=10, balance=0ETH, codeHash=0xabcd...),
}

state._storage_tries = {
    contractA: Trie(_data={
        slot0: U256(100),
        slot1: U256(200),
    }),
}

state._snapshots = []     # empty
```

#### Step 1: `begin_transaction` — outer transaction

The block execution code calls `begin_transaction` before processing Alice's transaction:

```python
begin_transaction(state, transient_storage)
```

After this call:

```
state._snapshots = [
    (
        copy of main_trie = {alice: ..., contractA: ...},
        {contractA: copy of its storage trie = {slot0: 100, slot1: 200}},
    ),
]
```

This is `state._snapshots[0]` — the outermost snapshot.

#### Step 2: Alice's tx modifies storage

Alice calls `transfer()` on contractA, which does `SSTORE(slot0, 50)`:

```
state._main_trie = {
    alice: Account(nonce=5, balance=1ETH, ...),
    contractA: Account(nonce=10, balance=0ETH, codeHash=0xabcd...),
}
# (no change to main trie in this case)

state._storage_tries = {
    contractA: Trie(_data={
        slot0: U256(50),     # ← changed from 100
        slot1: U256(200),
    }),
}
```

The snapshot at `state._snapshots[0]` is **unaffected** because it was a deep copy:

```
state._snapshots[0] = (
    copy of main_trie = {alice: ..., contractA: ...},
    {contractA: Trie(_data={slot0: 100, slot1: 200})},  # ← still 100!
)
```

#### Step 3: `begin_transaction` — inner call (Contract A → Contract B)

Contract A's code calls Contract B. Before entering, `begin_transaction` is called again:

```python
begin_transaction(state, transient_storage)
```

Now the stack has two snapshots:

```
state._snapshots = [
    snapshot_0: {contractA: {slot0: 100, slot1: 200}},   # pre-transaction state
    snapshot_1: {contractA: {slot0: 50, slot1: 200}},    # state before the inner call
]
```

Notice `snapshot_1` captures the **current** state (after Alice's SSTORE), not the pre-transaction state. It's a copy of the live tries at this exact moment.

#### Step 4: Contract B modifies storage, then reverts

Contract B does `SSTORE(some_key, 999)` then hits a `REVERT`.

```
live state: {contractA: {slot0: 50, slot1: 200, some_key: 999}}
snapshot_0: {contractA: {slot0: 100, slot1: 200}}
snapshot_1: {contractA: {slot0: 50, slot1: 200}}
```

`rollback_transaction` is called:

```python
state._main_trie, state._storage_tries = state._snapshots.pop()
# → restores snapshot_1
```

Live state is now:

```
state._storage_tries = {contractA: {slot0: 50, slot1: 200}}
# some_key: 999 is gone
```

The stack:

```
state._snapshots = [
    snapshot_0: {contractA: {slot0: 100, slot1: 200}},
]
```

#### Step 5: `commit_transaction` — outer transaction completes

Alice's transaction finishes successfully. `commit_transaction` is called:

```python
state._snapshots.pop()  # discard snapshot_0
```

The stack is now empty:

```
state._snapshots = []
```

Live state retains all changes: `slot0: 50, slot1: 200`.

---

### Why `get_storage_original` uses `[0]`

The key insight: **`state._snapshots[0]` is always the snapshot from before the current transaction began**, regardless of how many nested `begin_transaction` calls have happened inside it.

```
state._snapshots = [
    snapshot_0,   # ← index [0]: always the pre-transaction state
    snapshot_1,   # snapshot from inner call level 1
    snapshot_2,   # snapshot from inner call level 2
    ...
]
```

If `get_storage_original` used `state._snapshots[-1]` (the most recent snapshot), it would get the value from before the **most recent** call frame, not from before the **transaction**. That would be useless for EIP-2200, which specifically needs the value at the **start of the transaction**.

### The `created_accounts` edge case

```python
if address in state.created_accounts:
    return U256(0)
```

When a contract is created mid-transaction, `begin_transaction` already ran. So `snapshot_0` contains the state **before** the new contract existed. If `get_storage_original` tried to look up the new contract's storage in `snapshot_0`, it would find nothing and return `U256(0)` anyway. But the explicit check is clearer and avoids the unnecessary trie lookup.

---

### Summary: The Snapshot Stack

| Concept | Explanation |
|---------|-------------|
| `state._snapshots` | A **stack** (Python list) of deep copies of the entire state |
| `_snapshots[0]` | The **outermost** snapshot — state as it was before the current transaction started |
| `_snapshots[-1]` | The **most recent** snapshot — state as it was before the most recent inner call |
| `begin_transaction` | **Pushes** a deep copy onto the stack |
| `commit_transaction` | **Discards** the most recent snapshot (changes are permanent) |
| `rollback_transaction` | **Restores** the most recent snapshot (changes are undone) |
| `get_storage_original` | Reads from `_snapshots[0]` to get the value before any SSTORE in this transaction |

---

## 4. Why does `commit_transaction` call `state._snapshots.pop()`?

Because the snapshot's only purpose is to enable **rollback**. Once the transaction succeeds, there's nothing to rollback to anymore — the changes are permanent.

If we didn't pop, the snapshot stack would accumulate forever. But the deeper problem is that `get_storage_original` uses `state._snapshots[0]` to find the pre-transaction value.

### With the pop (correct behavior)

```
Transaction 1 starts:  begin_transaction → snapshots = [snap_0]
Transaction 1 commits: commit_transaction → snapshots.pop() → snapshots = []

Transaction 2 starts:  begin_transaction → snapshots = [snap_0]
Transaction 2 commits: commit_transaction → snapshots.pop() → snapshots = []
```

When `get_storage_original` reads `snapshots[0]` during Transaction 2, it gets the snapshot pushed **at the start of Transaction 2**. Correct.

### Without the pop (broken)

```
Transaction 1 starts:  begin_transaction → snapshots = [snap_0]
Transaction 1 commits: (no pop)         → snapshots = [snap_0]

Transaction 2 starts:  begin_transaction → snapshots = [snap_0, snap_1]
Transaction 2 commits: (no pop)         → snapshots = [snap_0, snap_1]

Transaction 3 starts:  begin_transaction → snapshots = [snap_0, snap_1, snap_2]
```

Now during Transaction 3, `get_storage_original` reads `snapshots[0]`. But `snapshots[0]` is `snap_0` — the state from **before Transaction 1 started**, two transactions ago. That's completely wrong. It should be reading the state from before Transaction 3 started, which would be `snap_2`.

The protocol would miscalculate SSTORE gas prices using stale values from old transactions. An attacker could exploit this: a slot that was cleared in Transaction 1 would still appear as having its original value when Transaction 3 runs, triggering incorrect refund logic.

### The invariant

`state._snapshots` should **always be empty between transactions**. It only contains entries while a transaction (or nested call frame within a transaction) is in progress.

```
Between transactions:    snapshots = []
During outer tx:         snapshots = [snap_0]
  During inner call:     snapshots = [snap_0, snap_1]
    During deeper call:  snapshots = [snap_0, snap_1, snap_2]
  Inner call commits:    snapshots = [snap_0, snap_1].pop() → [snap_0]
Inner call commits:      snapshots = [snap_0].pop() → []
Transaction done:        snapshots = []
```

Every `begin_transaction` pushes. Every `commit_transaction` or `rollback_transaction` pops. The stack always stays balanced.

---

## 5. Isn't copying the entire trie into memory every time impractical?

You're absolutely right to question this. **This is completely impractical for real Ethereum state.**

### The Scale Problem

Ethereum's current mainnet state:

- **~500 million accounts** in the state trie
- **~2.5 billion storage slots** across all contracts
- **Total state size: ~300+ GB on disk**

Now look at what `begin_transaction` does:

```python
def begin_transaction(state, transient_storage):
    state._snapshots.append(
        (
            copy_trie(state._main_trie),                                   # copies the ENTIRE account trie
            {k: copy_trie(t) for (k, t) in state._storage_tries.items()},  # copies EVERY storage trie
        )
    )
```

If there are 200 transactions per block, and each transaction calls `begin_transaction` once, that's **200 full copies of the entire world state per block**. At 300 GB each, that's **60 TB of memory per block**. Obviously impossible.

### Why EELS Does It This Way

**EELS is a specification, not a production client.** It's designed to be correct, readable, and easy to verify. It is **never** run against full mainnet state.

EELS is used for:
- **Test vectors** — small, curated test cases with a handful of accounts
- **EIP prototyping** — validating that a protocol change produces correct results on small examples
- **Reference behavior** — "this is what the correct answer looks like"

For these purposes, copying the entire trie is fine. The tests involve maybe 10 accounts and a few storage slots. The code is simple and obviously correct. There's no performance concern because the data is tiny.

### How Geth Actually Handles This

Geth does **not** copy the entire state on every transaction. It uses a completely different approach:

**Diff layers, not full copies.** Instead of deep-copying the entire trie, Geth records **only the nodes that changed**. A transaction modifies maybe hundreds of nodes out of billions. Geth stores just those differences in a lightweight in-memory structure:

```
Disk layer (300 GB, on disk)
    │
    └── Diff layer for Transaction N
         ├── Account trie changes: 5 modified nodes
         ├── Contract A storage: 3 modified nodes
         └── Contract B storage: 2 modified nodes
```

Each diff layer is maybe a few KB, not hundreds of GB. When the transaction commits, the diff is merged into the parent. When it reverts, the diff is simply discarded.

**Path-based storage.** Modern Geth also uses path-based keys, which makes diff tracking even cheaper. Instead of copying a trie, Geth just records:

```python
{
    "a/<hashed_address>": new_account_data,      # modified account
    "h/<hashed_address>/<hashed_slot>": new_val, # modified storage slot
}
```

A few dozen key-value pairs per transaction. That's it. No trie copying, no deep copies.

**Nested call frames.** For inner calls (`CALL` → `CALL` → `CALL`), Geth doesn't even create separate snapshots for every call frame. Instead, it uses a **journal** — a sequential log of mutations:

```
Journal:
  1. balance[alice] -= 100
  2. balance[bob]   += 100
  3. storage[contractA][slot0] = 50
  ...
```

On revert, Geth walks the journal backwards and undoes each mutation. No copying, no snapshots, just a list of "what changed" that gets discarded on commit.

### The Fundamental Distinction

| | EELS | Geth / Production Clients |
|---|---|---|
| **Data size** | Small test cases (10-100 accounts) | Full mainnet state (500M+ accounts) |
| **Snapshot strategy** | Deep copy entire tries | Diff layers + journal of changes |
| **Memory per transaction** | O(entire state) | O(number of modified slots) — typically a few KB |
| **Priority** | Correctness and readability | Performance and memory efficiency |
| **Question it answers** | "What is the correct result?" | "How do we compute it without running out of memory?" |

---

## 6. Is `commit_transaction` called only when the outer call succeeds?

No — **every `process_message` call gets its own `begin_transaction` / `commit_transaction` pair**, regardless of nesting depth.

### Every `process_message` gets its own begin/commit pair

Look at `process_message`:

```python
def process_message(message: Message) -> Evm:
    state = message.block_env.state
    transient_storage = message.tx_env.transient_storage

    # ... stack depth check ...

    # ← EVERY call frame pushes a snapshot
    begin_transaction(state, transient_storage)

    # ... execute code ...

    if evm.error:
        rollback_transaction(state, transient_storage)   # on failure
    else:
        commit_transaction(state, transient_storage)     # on success ← EVERY call frame pops
    return evm
```

**Every single `process_message` call** — the top-level transaction AND every nested CALL/DELEGATECALL/CREATE — gets its own `begin_transaction` at the start and its own `commit_transaction` or `rollback_transaction` at the end.

### Concrete example: three nested calls

```
Alice's tx → Contract A → Contract B
```

#### Step 1: Top-level transaction enters

```python
process_transaction(...)          # fork.py
  → process_message(message_0)     # Alice's tx
    → begin_transaction(state, transient_storage)
      → snapshots = [snap_0]       # stack depth: 1
    → execute Alice's tx code
      → CALL Contract A            # triggers a nested process_message
```

#### Step 2: CALL Contract A

```python
CALL instruction (system.py)
  → child_message = build_child_message(...)
  → child_evm = process_message(child_message)  # Contract A
    → begin_transaction(state, transient_storage)
      → snapshots = [snap_0, snap_1]             # stack depth: 2
    → execute Contract A code
      → CALL Contract B                          # triggers another nested process_message
```

#### Step 3: CALL Contract B

```python
CALL instruction (system.py)
  → child_message = build_child_message(...)
  → child_message = process_message(child_message)  # Contract B
    → begin_transaction(state, transient_storage)
      → snapshots = [snap_0, snap_1, snap_2]        # stack depth: 3
    → execute Contract B code
    → Contract B finishes successfully

    → commit_transaction(state, transient_storage)  # pops snap_2
      → snapshots = [snap_0, snap_1]                # stack depth: 2
    → returns child_evm to Contract A
```

Contract B's snapshot is popped. Its changes are permanent.

#### Step 4: Contract A finishes

```
    → commit_transaction(state, transient_storage)  # pops snap_1
      → snapshots = [snap_0]                        # stack depth: 1
    → returns child_evm to Alice's tx
```

Contract A's snapshot is popped. Its changes are permanent.

#### Step 5: Alice's tx finishes

```
    → commit_transaction(state, transient_storage)  # pops snap_0
      → snapshots = []                              # stack depth: 0
      → not state._snapshots → True → state.created_accounts.clear()
    → returns to process_transaction
```

Now the stack is empty, and `created_accounts` is cleared.

### The `created_accounts.clear()` condition

```python
def commit_transaction(state: State, transient_storage: TransientStorage) -> None:
    state._snapshots.pop()
    if not state._snapshots:           # ← only True when stack is EMPTY
        state.created_accounts.clear()
    transient_storage._snapshots.pop()
```

`created_accounts.clear()` only fires when the snapshot stack becomes **empty** — i.e., when the outermost transaction completes. It does NOT fire for inner call frames.

This is correct because `created_accounts` tracks which accounts were created during the **entire transaction**, not during a single call frame. It needs to persist across all nested calls and only reset at the very end.

### Summary

| Event | Snapshot action | Stack depth after | `created_accounts.clear()`? |
|-------|----------------|-------------------|----------------------------|
| Top-level tx enters | push snap_0 | 1 | No |
| CALL Contract A | push snap_1 | 2 | No |
| CALL Contract B | push snap_2 | 3 | No |
| Contract B succeeds | **commit** → pop snap_2 | 2 | No (stack not empty) |
| Contract A succeeds | **commit** → pop snap_1 | 1 | No (stack not empty) |
| Top-level tx succeeds | **commit** → pop snap_0 | 0 | **Yes** (stack is empty) |

Every call frame pushes on `begin_transaction` and pops on `commit_transaction` or `rollback_transaction`. The outermost transaction is special only in that it's the one that empties the stack and triggers `created_accounts.clear()`.
