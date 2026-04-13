---
title: EELS(6) Q&A Notes
published: 2026-04-04
pinned: false
description: Some QA that might be helpful.
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---


## Table of Contents

1. [How do the four context layers connect?](#1-how-do-the-four-context-layers-connect)
2. [How are messages passed between contexts? When does a new sub-Evm get activated?](#2-how-are-messages-passed-between-contexts-when-does-a-new-sub-evm-get-activated)
3. [Does `TransactionEnvironment` hold a reference to `BlockEnvironment`?](#3-does-transactionenvironment-hold-a-reference-to-blockenvironment)
4. [How does `process_message_call` work?](#4-how-does-process_message_call-work)
5. [How does `process_create_message` work?](#5-how-does-process_create_message-work)
6. [How does `process_message` work?](#6-how-does-process_message-work)
7. [How does the complete transaction flow work end-to-end?](#7-how-does-the-complete-transaction-flow-work-end-to-end)

---

## 1. How do the four context layers connect?

The EVM execution context is structured in four layers, each one holding references to the layers below it:

```
BlockEnvironment  (one per block)
└── TransactionEnvironment  (one per transaction)
    └── Message  (one per call or creation)
        └── Evm  (one execution frame)
            └── Evm  (nested, for sub-calls)
```

Each layer is a **container** that holds the layers below it. The connections are literally Python object references — one object holds another as a field.

---

### Layer 1: `BlockEnvironment`

Created **once per block** in `state_transition()`:

```python
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
```

This object is **immutable** and **shared**. Every transaction in the block, every call within every transaction, every nested sub-call — they all reference the **exact same** `BlockEnvironment` object. No copies.

---

### Layer 2: `TransactionEnvironment`

Created **once per transaction** in `process_transaction()`:

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

**Note:** `tx_env` does **not** hold a reference to `block_env`. They are independent objects created in the same function. The `origin` was recovered using `block_env.chain_id`, and the `gas_price` was computed using `block_env.base_fee_per_gas` — so `tx_env` is *derived from* `block_env`, but doesn't hold it as a field. The actual connection happens at the next layer.

---

### Layer 3: `Message`

Created by `prepare_message()` for the top-level tx, and by `generic_call()` for every nested `CALL`. It bridges the two:

```python
return Message(
    block_env=block_env,              # ← direct reference
    tx_env=tx_env,                    # ← direct reference
    caller=tx_env.origin,
    target=tx.to,
    gas=tx_env.gas,
    value=tx.value,
    data=msg_data,
    code=code,
    depth=Uint(0),
    current_target=current_target,
    code_address=code_address,
    should_transfer_value=True,
    is_static=False,
    accessed_addresses=accessed_addresses,
    accessed_storage_keys=set(tx_env.access_list_storage_keys),
    disable_precompiles=False,
    parent_evm=None,
)
```

**The connection is explicit:** `Message` has fields `block_env` and `tx_env` that hold **direct references** to the objects passed in. Not copies. The same objects.

So when you access `message.block_env.coinbase` anywhere deep in the call stack, you're reading the same coinbase that was set at the block level.

---

### Layer 4: `Evm`

Created fresh for each `Message` by `process_message()`:

```python
evm = Evm(
    pc=Uint(0),
    stack=[],
    memory=bytearray(),
    code=code,
    gas_left=message.gas,
    valid_jump_destinations=valid_jump_destinations,
    logs=(),
    refund_counter=0,
    running=True,
    message=message,                          # ← direct reference
    output=b"",
    accounts_to_delete=set(),
    return_data=b"",
    error=None,
    accessed_addresses=message.accessed_addresses,   # ← same set object
    accessed_storage_keys=message.accessed_storage_keys,
)
```

**The connection:** `evm.message = message` — the `Evm` holds a direct reference to the `Message`. And `evm.accessed_addresses` / `evm.accessed_storage_keys` are the **same set objects** as those on the `Message`. Not copies. This is deliberate — when a nested call warms an address, the parent sees it immediately because they share the same set.

---

## 2. How are messages passed between contexts? When does a new sub-Evm get activated?

A new sub-Evm is activated when the executing bytecode encounters a `CALL` (or `CALLCODE`, `DELEGATECALL`, `STATICCALL`, `CREATE`, `CREATE2`) instruction. Here's exactly how it works.

### The `call` opcode handler

When the EVM encounters a `CALL` instruction, the `call()` function in `vm/instructions/system.py` runs:

```python
def call(evm: Evm) -> None:
    # Pop stack values: gas, to, value, memory offsets...
    gas = Uint(pop(evm.stack))
    to = to_address_masked(pop(evm.stack))
    value = pop(evm.stack)
    # ... more stack pops for memory offsets ...

    # ... gas calculations, memory expansion, access checks ...

    generic_call(
        evm,
        message_call_gas.sub_call,
        value,
        evm.message.current_target,  # caller
        to,                           # target
        code_address,
        should_transfer_value=True,
        is_staticcall=False,
        # ... memory offsets, code, etc.
    )

    evm.pc += Uint(1)
```

### `generic_call` constructs the child Message

```python
def generic_call(evm, gas, value, caller, to, code_address, ...):
    # Read calldata from parent's memory
    call_data = memory_read_bytes(
        evm.memory, memory_input_start_position, memory_input_size
    )

    child_message = Message(
        block_env=evm.message.block_env,        # ← SAME block env reference
        tx_env=evm.message.tx_env,              # ← SAME tx env reference
        caller=caller,
        target=to,
        gas=gas,
        value=value,
        data=call_data,
        code=code,
        current_target=to,
        depth=evm.message.depth + Uint(1),      # ← incremented
        code_address=code_address,
        should_transfer_value=should_transfer_value,
        is_static=True if is_staticcall else evm.message.is_static,
        accessed_addresses=evm.accessed_addresses.copy(),
        accessed_storage_keys=evm.accessed_storage_keys.copy(),
        disable_precompiles=disable_precompiles,
        parent_evm=evm,                         # ← reference to parent Evm
    )
    child_evm = process_message(child_message)
```

**The child `Message` is constructed by copying references from the parent:**

| Field | Source | Shared or copied? |
|-------|--------|-------------------|
| `block_env` | `evm.message.block_env` | **Same reference** (one per block, never changes) |
| `tx_env` | `evm.message.tx_env` | **Same reference** (one per transaction, shared across all calls) |
| `caller` | Determined by opcode | New value |
| `target` | Determined by opcode | New value |
| `current_target` | `to` (or `current_target` for DELEGATECALL) | New value |
| `gas` | Calculated from parent's remaining gas | New value |
| `value` | From stack | New value |
| `data` | Read from parent's memory | **New bytes** (copied out of parent memory) |
| `code` | Fetched from `code_address` | New bytes |
| `depth` | `evm.message.depth + 1` | Incremented |
| `is_static` | Inherited from parent or forced by STATICCALL | Inherited or overridden |
| `accessed_addresses` | `evm.accessed_addresses.copy()` | **Shallow copy** of the set |
| `accessed_storage_keys` | `evm.accessed_storage_keys.copy()` | **Shallow copy** of the set |
| `parent_evm` | `evm` | **Same reference** (direct link to parent) |

### The parent-child communication

After `child_evm = process_message(child_message)` returns, the parent handles the result:

```python
if child_evm.error:
    incorporate_child_on_error(evm, child_evm)
    evm.return_data = child_evm.output
    push(evm.stack, U256(0))
else:
    incorporate_child_on_success(evm, child_evm)
    evm.return_data = child_evm.output
    push(evm.stack, U256(1))

# Write child's output into parent's memory
memory_write(
    evm.memory,
    memory_output_start_position,
    child_evm.output[:actual_output_size],
)
```

**On success:**

```python
def incorporate_child_on_success(evm: Evm, child_evm: Evm) -> None:
    evm.gas_left += child_evm.gas_left           # unused gas returned
    evm.logs += child_evm.logs                    # logs appended
    evm.refund_counter += child_evm.refund_counter
    evm.accounts_to_delete.update(child_evm.accounts_to_delete)
    evm.accessed_addresses.update(child_evm.accessed_addresses)
    evm.accessed_storage_keys.update(child_evm.accessed_storage_keys)
```

The child's remaining gas, logs, refunds, deletions, and warm sets are **merged into the parent**.

**On error:**

```python
def incorporate_child_on_error(evm: Evm, child_evm: Evm) -> None:
    evm.gas_left += child_evm.gas_left  # only unused gas returned
```

Everything else — logs, refunds, deletions, warm sets — is **discarded**. Only unused gas returns to the parent. This is the mechanism by which a reverted sub-call rolls back its own state changes while still returning unspent gas.

### The complete connection picture

```
[ SHARED GLOBALS ]
  ║
  ║  ┌──────────────────────────┐
  ╠══│BlockEnvironment (B)      │ (One per block)
  ║  └──────────────────────────┘
  ║  ┌──────────────────────────┐
  ╚══│TransactionEnvironment(T) │ (One per transaction)
     └──────────────────────────┘
           │
           │ (Initializes)
           ▼
 [ EXECUTION STACK ]
 ┌──────────────────────────────────────────────────┐
 │ MESSAGE (Depth 0)                                │
 │  ├─ block_env ─────► (B)                         │
 │  └─ tx_env    ─────► (T)                         │
 │                                                  │
 │  EVM FRAME (Depth 0)                             │
 │   │  (Stack, Memory, PC, Gas)                    │
 │   │                                              │
 │   └─── [ CALL / CREATE ] ──┐                     │
 └────────────────────────────│─────────────────────┘
                              │
           ┌──────────────────┘
           ▼
 ┌──────────────────────────────────────────────────┐
 │ MESSAGE (Depth 1)                                │
 │  ├─ block_env ─────► (B)                         │
 │  ├─ tx_env    ─────► (T)                         │
 │  └─ parent_evm ────► [Ref to EVM Depth 0]        │
 │                                                  │
 │  EVM FRAME (Depth 1)                             │
 │   │  (Stack, Memory, PC, Gas)                    │
 │   │                                              │
 │   └─── [ CALL ] ──┐                              │
 └───────────────────│──────────────────────────────┘
                     │
           ┌─────────┘
           ▼
 ┌──────────────────────────────────────────────────┐
 │ MESSAGE (Depth 2)                                │
 │  ├─ block_env ─────► (B)                         │
 │  ├─ tx_env    ─────► (T)                         │
 │  └─ parent_evm ────► [Ref to EVM Depth 1]        │
 │                                                  │
 │  EVM FRAME (Depth 2)                             │
 └──────────────────────────────────────────────────┘
```

**Key points:**

1. **`block_env` and `tx_env` flow down by reference.** Every `Message` at every depth points to the same `BlockEnvironment` and `TransactionEnvironment` objects. They are the "global" context shared across the entire call tree.

2. **`Message` is the "what"** — what to call, with what gas, value, data, code. It's a passive data structure.

3. **`Evm` is the "how"** — the mutable execution state: program counter, stack, memory, gas left, logs, errors. It's created fresh for each `Message` and discarded (or merged into parent) when execution finishes.

4. **`parent_evm` is the back-link.** The child `Message` holds a reference to the parent `Evm`. This is used by the interpreter to return results, but the child never mutates the parent directly — results flow back through `incorporate_child_on_success/error`.

5. **Warm sets are copied (not shared) between parent and child.** `accessed_addresses.copy()` creates a new set. On success, the child's warm set is merged back into the parent. This means warming an address in a child call doesn't immediately warm it in the parent — only if the child succeeds.

---

## 3. Does `TransactionEnvironment` hold a reference to `BlockEnvironment`?

**No.** The `TransactionEnvironment` dataclass has no `block_env` field:

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

None of these fields is a `BlockEnvironment`.

However, `tx_env` is **derived from** `block_env`. The `origin` was recovered using `block_env.chain_id`. The `gas_price` was computed using `block_env.base_fee_per_gas`. The two objects are always used together downstream. But at the Python object level, there's no `tx_env.block_env` attribute.

The actual connection between `block_env` and `tx_env` happens at the `Message` layer:

```python
return Message(
    block_env=block_env,      # ← direct reference to block_env
    tx_env=tx_env,            # ← direct reference to tx_env
    ...
)
```

`Message` is the object that **bridges** the two. It holds references to both, side by side. That's why every `Message` (and every `Evm` via `evm.message`) can access both the block-level and transaction-level context simultaneously.

---

## 4. How does `process_message_call` work?

**Purpose:** The single entry point for all EVM execution. It routes between contract creation and contract call, handles EIP-7702 delegation resolution, and packages the results.

```python
def process_message_call(message: Message) -> MessageCallOutput:
    block_env = message.block_env
    refund_counter = U256(0)
```

### Contract creation vs. contract call

```python
if message.target == Bytes0(b""):
    # Contract creation path
    is_collision = (
        account_has_code_or_nonce(block_env.state, message.current_target)
        or account_has_storage(block_env.state, message.current_target)
    )
    if is_collision:
        return MessageCallOutput(
            gas_left=Uint(0), refund_counter=U256(0), logs=tuple(),
            accounts_to_delete=set(), error=AddressCollision(),
            return_data=Bytes(b""),
        )
    else:
        evm = process_create_message(message)
```

**Address collision check.** Before creating a contract, check if the computed address (`current_target`) is already occupied — by an account with code, non-zero nonce, or existing storage. If so, fail immediately with `AddressCollision()`. All gas is consumed, no state changes occur.

### Contract call path — EIP-7702 delegation

```python
else:
    if message.tx_env.authorizations != ():
        refund_counter += set_delegation(message)

    delegated_address = get_delegated_code_address(message.code)
    if delegated_address is not None:
        message.disable_precompiles = True
        message.accessed_addresses.add(delegated_address)
        message.code = get_code(
            block_env.state,
            get_account(block_env.state, delegated_address).code_hash,
        )
        message.code_address = delegated_address

    evm = process_message(message)
```

**EIP-7702 authorization processing:** If the transaction carries `authorizations` (Type 4), `set_delegation` processes them first.

**EIP-7702 delegation resolution:** `get_delegated_code_address(message.code)` checks if the loaded code starts with `0xef0100`. If so:
1. `disable_precompiles = True` — prevents accidental precompile dispatch.
2. Warm the delegated address.
3. Resolve the delegation: fetch the **actual** bytecode from the target contract.
4. Update `code_address`.

### Package the results

```python
if evm.error:
    logs: Tuple[Log, ...] = ()
    accounts_to_delete = set()
else:
    logs = evm.logs
    accounts_to_delete = evm.accounts_to_delete
    refund_counter += U256(evm.refund_counter)

return MessageCallOutput(
    gas_left=evm.gas_left,
    refund_counter=refund_counter,
    logs=logs,
    accounts_to_delete=accounts_to_delete,
    error=evm.error,
    return_data=evm.output,
)
```

**On error:** Logs and `accounts_to_delete` are cleared. Only `gas_left`, `error`, and `return_data` are preserved.

**On success:** Logs, accounts-to-delete, and refund credits are propagated.

---

## 5. How does `process_create_message` work?

**Purpose:** Handles the contract-creation-specific setup before delegating to `process_message`.

### Step 1: Snapshot

```python
begin_transaction(state, transient_storage)
```

Push a deep copy of the entire state onto the snapshot stack. If contract creation fails, everything reverts to this state.

### Step 2: Mark the account as created

```python
mark_account_created(state, message.current_target)
```

Add the new contract address to `state.created_accounts`. Needed for:
1. **EIP-2200 (`get_storage_original`):** A newly created account has no pre-existing storage.
2. **EIP-6780 (`SELFDESTRUCT`):** A contract can only be fully destroyed if created in the same transaction.

### Step 3: Set nonce to 1

```python
increment_nonce(state, message.current_target)
```

Per EIP-161, the new contract's nonce starts at 1. Prevents address reuse attacks.

### Step 4: Execute the init code

```python
evm = process_message(message)
```

The `message.code` contains the init code. `process_message` runs it. The init code's job is to return the runtime bytecode via `RETURN`.

### Step 5: Validate and deposit the runtime code

```python
if not evm.error:
    contract_code = evm.output
    contract_code_gas = Uint(len(contract_code)) * GAS_CODE_DEPOSIT_PER_BYTE
    try:
        if len(contract_code) > 0:
            if contract_code[0] == 0xEF:
                raise InvalidContractPrefix
        charge_gas(evm, contract_code_gas)
        if len(contract_code) > MAX_CODE_SIZE:
            raise OutOfGasError
    except ExceptionalHalt as error:
        rollback_transaction(state, transient_storage)
        evm.gas_left = Uint(0)
        evm.output = b""
        evm.error = error
    else:
        set_code(state, message.current_target, contract_code)
        commit_transaction(state, transient_storage)
else:
    rollback_transaction(state, transient_storage)
```

**Validation checks:**
1. **`0xEF` prefix check** — rejected (reserved for EOF).
2. **Code deposit gas** — 200 gas per byte.
3. **Size limit** — max 24,576 bytes.

**On any validation failure:** Rollback everything. Zero out gas. Clear output. Set the error. The contract is NOT created.

**On success:** Store bytecode in `_code_store`, update account's `code_hash`, commit the snapshot.

**On init code failure:** Rollback the nonce increment, the `created_accounts` marker, and any storage changes.

---

## 6. How does `process_message` work?

**Purpose:** The core EVM execution engine. This is where the bytecode actually runs.

### Step 1: Stack depth check

```python
if message.depth > STACK_DEPTH_LIMIT:
    raise StackDepthLimitError("Stack depth limit reached")
# STACK_DEPTH_LIMIT = 1024
```

Prevents infinite recursion attacks.

### Step 2: Build the Evm frame

```python
evm = Evm(
    pc=Uint(0),
    stack=[],
    memory=bytearray(),
    code=code,
    gas_left=message.gas,
    valid_jump_destinations=get_valid_jump_destinations(code),
    logs=(),
    refund_counter=0,
    running=True,
    message=message,
    output=b"",
    accounts_to_delete=set(),
    return_data=b"",
    error=None,
    accessed_addresses=message.accessed_addresses,   # same set object
    accessed_storage_keys=message.accessed_storage_keys,
)
```

Every field is initialized from scratch. Notably, `accessed_addresses` and `accessed_storage_keys` are the **same set objects** as on the `Message` — not copies.

### Step 3: Snapshot

```python
begin_transaction(state, transient_storage)
```

Push another snapshot for this specific call frame.

### Step 4: Transfer value

```python
if message.should_transfer_value and message.value != 0:
    move_ether(state, message.caller, message.current_target, message.value)
```

Happens **before** code execution. `STATICCALL` and `DELEGATECALL` set `should_transfer_value = False`, so they skip this.

### Step 5: Precompile dispatch or bytecode loop

```python
try:
    if evm.message.code_address in PRE_COMPILED_CONTRACTS:
        if not message.disable_precompiles:
            PRE_COMPILED_CONTRACTS[evm.message.code_address](evm)
    else:
        while evm.running and evm.pc < ulen(evm.code):
            op = Ops(evm.code[evm.pc])
            op_implementation[op](evm)
        evm_trace(evm, EvmStop(Ops.STOP))
except ExceptionalHalt as error:
    evm.gas_left = Uint(0)
    evm.output = b""
    evm.error = error
except Revert as error:
    evm.error = error
```

**Precompile path:** If `code_address` is a precompile AND `disable_precompiles` is False, call the precompile directly. Precompiles don't run a bytecode loop — they're Python functions.

**Bytecode loop:** `while evm.running and evm.pc < len(code)`. Each opcode implementation mutates the `evm` — popping/pushing the stack, reading/writing memory, charging gas, advancing `pc`.

**Two exception classes:**
- **`ExceptionalHalt`** (`OutOfGasError`, `StackOverflowError`, `InvalidOpcode`, etc.): Zero out `gas_left`, clear `output`, set the error.
- **`Revert`** (from `REVERT` opcode): Keep `gas_left` (refundable), keep `output` (revert reason), set the error.

### Step 6: Commit or rollback

```python
if evm.error:
    rollback_transaction(state, transient_storage)
else:
    commit_transaction(state, transient_storage)
return evm
```

Error → rollback snapshot. Success → commit snapshot.

---

## 7. How does the complete transaction flow work end-to-end?

Here's what happens when a user sends a Type 2 tx calling `Uniswap.swap()`:

```
1. process_transaction (fork.py)
   ├── validate_transaction(tx)          → intrinsic_gas (e.g., 21,000)
   ├── check_transaction(block_env, tx)  → sender, effective_gas_price
   ├── [STATE] Deduct gas fee & Increment nonce
   └── build tx_env
       │
       ▼
2. prepare_message (utils/message.py)
   ├── [STATE] Warm addresses: {origin, precompiles, access_list}
   ├── [STATE] Resolve code: get_code(state, target.code_hash)
   └── build Message (Depth 0)
       │
       ▼
3. process_message_call (vm/interpreter.py)
   ├── [CHECK] Target address collision / EIP-7702 delegation
   └── [EXEC] Calls process_message(message)
       │
       ▼
4. process_message (ROOT CALL - Depth 0)
   ├── [SNAP] begin_transaction()  → Snapshot #1 (Root)
   ├── [STATE] move_ether(origin -> uniswap)
   │
   ├── [LOOP] While evm.running:
   │   │  ... Executes Uniswap Bytecode ...
   │   │
   │   │  ┌── CALL WETH (Depth 1) ───────────────────────────────────┐
   │   │  │ 5. process_message (Sub-call)                            │
   │   │  │    ├── [SNAP] begin_transaction() → Snapshot #2          │
   │   │  │    ├── [STATE] move_ether(uniswap -> weth)               │
   │   │  │    │                                                     │
   │   │  │    ├── [LOOP] while loop (WETH Bytecode)                 │
   │   │  │    │   │                                                 │
   │   │  │    │   │ ┌── CALL ERC20 (Depth 2) ─────────────────────┐ │
   │   │  │    │   │ │ 6. process_message (Sub-call)               │ │
   │   │  │    │   │ │    ├── [SNAP] begin_transaction() → Snap #3 │ │
   │   │  │    │   │ │    ├── [LOOP] while loop (ERC20 Bytecode)   │ │
   │   │  │    │   │ │    ├── [SNAP] commit_transaction() (Snap #3)│ │
   │   │  │    │   │ │    └── return child_evm                     │ │
   │   │  │    │   │ └─────────────────────────────────────────────┘ │
   │   │  │    │   │                                                 │
   │   │  │    │   └── incorporate_child_on_success(parent, child)   │
   │   │  │    │                                                     │
   │   │  │    ├── [SNAP] commit_transaction() → Snapshot #2         │
   │   │  │    └── return child_evm                                  │
   │   │  └──────────────────────────────────────────────────────────┘
   │   │
   │   └── incorporate_child_on_success(root, child)
   │
   ├── [SNAP] commit_transaction() → Snapshot #1 (Root)
   └── return evm
       │
       ▼
5. process_message_call (Teardown)
   ├── [RESULT] Propagate logs, refunds, and self-destructs
   └── return MessageCallOutput
       │
       ▼
6. process_transaction (Finalize)
   ├── [GAS] Calculate refund (max 20% of gas used)
   ├── [GAS] Refund unused gas + pay coinbase priority fee
   ├── [RECEIPT] Build receipt & insert into Trie
   └── return TransactionResult
```

Every nested call gets its own `Message` (the "what"), its own `Evm` frame (the "how"), and its own snapshot (the "undo button"). They all share the same `BlockEnvironment` and `TransactionEnvironment` — the global context that flows down through every level.

---

*End of notes.*
