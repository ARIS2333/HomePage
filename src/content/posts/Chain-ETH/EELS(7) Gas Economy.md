---
title: EELS(7) Gas Economy
published: 2026-04-04
pinned: false
description: Gas accounting mechanisms of EVM
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## **Table of Contents**

- [**Preface**](#preface)
- [**Introduction**](#introduction)
- [**1. Gas Constants**](#1-gas-constants)
- [**2. Gas Accounting**](#2-gas-accounting--charge_gas)
- [**3. Gas Initialization and Propagation**](#3-gas-initialization-and-propagation)
- [**4. Memory Expansion Costs**](#4-memory-expansion-costs)
- [**5. Storage Access Costs**](#5-storage-access-costs--eip-2929)
- [**6. SSTORE Gas Rules**](#6-sstore-gas-rules--eip-2200--eip-3529)
- [**7. Transient Storage Gas**](#7-transient-storage-gas--eip-1153)
- [**8. Sub-call Gas**](#8-sub-call-gas--the-6364-rule)
- [**9. Gas Refunds**](#9-gas-refunds)


## Preface

This article is part of a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**.

**What is EELS?** EELS is a Python reference implementation of the Ethereum execution client, designed with a focus on readability and clarity. It serves as a programmer-friendly, up-to-date successor to the original Yellow Paper and is the primary tool for prototyping new Ethereum Improvement Proposals (EIPs). EELS provides complete protocol snapshots at each fork and rendered diffs between them.

**Scope and Implementation** Note that EELS does not implement the JSON-RPC API or P2P networking. To validate blocks, it requires an external RPC provider to fetch data, which EELS then processes and stores in a local database.

**Version Reference** The code analysis in this article is based on the **Osaka** fork:
[ethereum/execution-specs (Osaka)](https://github.com/ethereum/execution-specs/tree/forks/amsterdam/src/ethereum/forks/osaka)

---

## Introduction

Gas is Ethereum’s fundamental resource-pricing and anti-spam mechanism. It ensures that every computation has a cost, preventing infinite loops and denial-of-service attacks. This chapter covers how gas is defined, initialized, consumed, and refunded during EVM execution in the Osaka fork.

---

## 1. Gas Constants

All gas constants for the Osaka fork are defined in [`vm/gas.py`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/gas.py.html).

### 1.1 Opcode Tier Constants

```python
GAS_JUMPDEST = Uint(1)
GAS_BASE     = Uint(2)        # PUSH, DUP, SWAP, …
GAS_VERY_LOW = Uint(3)        # ADD, SUB, NOT, LT, GT, …
GAS_LOW      = Uint(5)        # MUL, DIV, …
GAS_MID      = Uint(8)        # ADDMOD, MULMOD, JUMP
GAS_HIGH     = Uint(10)       # JUMPI
GAS_FAST_STEP = Uint(5)
```

### 1.2 Storage Constants

```python
GAS_STORAGE_SET        = Uint(20000)  # First write to a zero slot
GAS_COLD_STORAGE_WRITE = Uint(5000)   # First write to a non-zero slot
GAS_COLD_STORAGE_ACCESS = Uint(2100)  # Cold SLOAD (EIP-2929)
GAS_COLD_ACCOUNT_ACCESS = Uint(2600)  # Cold address access (EIP-2929)
GAS_WARM_ACCESS        = Uint(100)    # Warm SLOAD/account (EIP-2929)
REFUND_STORAGE_CLEAR   = 4800         # Refund: nonzero slot cleared (EIP-3529)
```

### 1.3 Memory and Copy Constants

```python
GAS_MEMORY             = Uint(3)      # Per-word linear memory cost
GAS_COPY               = Uint(3)      # Per-word copy cost (CALLDATACOPY, etc.)
GAS_KECCAK256          = Uint(30)     # KECCAK256 base cost
GAS_KECCAK256_PER_WORD = Uint(6)      # KECCAK256 per 32-byte word
GAS_RETURN_DATA_COPY   = Uint(3)      # RETURNDATACOPY per word
```

### 1.4 Call and Creation Constants

```python
GAS_CREATE             = Uint(32000)  # CREATE / CREATE2 base
GAS_CODE_DEPOSIT_PER_BYTE = Uint(200) # Per-byte code deposit after CREATE
GAS_CODE_INIT_PER_WORD = Uint(2)      # EIP-3860: init code size cost per word
GAS_NEW_ACCOUNT        = Uint(25000)  # CALL to a new account
GAS_CALL_VALUE         = Uint(9000)   # CALL with non-zero value
GAS_CALL_STIPEND       = Uint(2300)   # Stipend added to callee when value > 0
GAS_SELF_DESTRUCT      = Uint(5000)   # SELFDESTRUCT base
GAS_SELF_DESTRUCT_NEW_ACCOUNT = Uint(25000)  # SELFDESTRUCT to new recipient
```

### 1.5 Miscellaneous

```python
GAS_LOG              = Uint(375)   # LOGx base
GAS_LOG_DATA_PER_BYTE = Uint(8)   # LOGx data
GAS_LOG_TOPIC        = Uint(375)  # LOGx per topic
GAS_BLOCK_HASH       = Uint(20)   # BLOCKHASH
GAS_EXPONENTIATION   = Uint(10)   # EXP base
GAS_EXPONENTIATION_PER_BYTE = Uint(50)  # EXP per byte of exponent
GAS_BLOBHASH_OPCODE  = Uint(3)    # BLOBHASH
```

### 1.6 Osaka-Specific: P256VERIFY

Osaka introduces the secp256r1 precompile ([EIP-7951](https://eips.ethereum.org/EIPS/eip-7951)):

```python
GAS_PRECOMPILE_P256VERIFY = Uint(6900)
```

---

## 2. Gas Accounting — `charge_gas`

All gas consumption in the EVM flows through a single function, [`charge_gas`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/gas.py.html#ethereum.forks.osaka.vm.gas.charge_gas:0):

```python
def charge_gas(evm: Evm, amount: Uint) -> None:
    evm_trace(evm, GasAndRefund(int(amount)))

    if evm.gas_left < amount:
        raise OutOfGasError
    else:
        evm.gas_left -= amount
```

Two details are significant. First, `OutOfGasError` is raised as a class reference, not an instance — it is an `ExceptionalHalt` subclass, so catching it in `process_message` zeros `evm.gas_left` and records the error. Second, `gas_left` is **not** zeroed before the raise; that happens in the `ExceptionalHalt` handler inside `process_message`. The `evm_trace` call has no semantic effect and exists purely for debugging.

Every instruction calls `charge_gas` before performing its operation. If the charge fails, execution halts immediately and all state changes for the frame are rolled back.

---

## 3. Gas Initialization and Propagation

### 3.1 Frame Initialization

When `process_message` constructs an `Evm` frame, `gas_left` is set directly from `message.gas`:

```python
evm = Evm(
    ...
    gas_left=message.gas,
    ...
)
```

`message.gas` is the gas allocated to this call frame after the intrinsic gas has been deducted at the transaction level. There is no additional deduction at frame construction time.

### 3.2 Warm Access Pre-population

Before the bytecode loop runs, the warm access sets are initialized from the `Message`. Two sets are established at frame construction:

```python
evm = Evm(
    ...
    accessed_addresses=message.accessed_addresses,
    accessed_storage_keys=message.accessed_storage_keys,
    ...
)
```

These are the **same set objects** as those on the `Message` — they are not copied. The warm sets are pre-populated at the transaction level from the following sources:

- The transaction sender and recipient (always warm)
- All precompile addresses (always warm)
- All addresses in the EIP-2930 access list, and their declared storage slots
- The coinbase address (always warm per EIP-3651)

Any address or storage slot added to the warm set during a child frame’s execution is visible to the parent immediately, because both share the same set object.

### 3.3 Gas at Transaction Level

Intrinsic gas is deducted from the total gas limit before any EVM frame runs. The remaining gas is then passed as `message.gas` into the first call frame. Intrinsic gas calculation (covered in the Transactions chapter) uses constants from [`transactions.py`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/transactions.py.html) — `GAS_TX_BASE = Uint(21000)`, `GAS_TX_DATA_TOKEN_STANDARD = Uint(4)`, `GAS_TX_DATA_TOKEN_FLOOR = Uint(10)`, and so on.

---

## 4. Memory Expansion Costs

### 4.1 Memory Cost Formula

EVM memory has a **quadratic cost** to deter unbounded allocation. The total cost for a given memory size is computed by [`calculate_memory_gas_cost`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/gas.py.html#ethereum.forks.osaka.vm.gas.calculate_memory_gas_cost:0):

```python
def calculate_memory_gas_cost(size_in_bytes: Uint) -> Uint:
    size_in_words = ceil32(size_in_bytes) // Uint(32)
    linear_cost   = size_in_words * GAS_MEMORY           # 3 per word
    quadratic_cost = size_in_words ** Uint(2) // Uint(512)
    total_gas_cost = linear_cost + quadratic_cost
    try:
        return total_gas_cost
    except ValueError as e:
        raise OutOfGasError from e
```

The formula is: **3 × words + words² / 512**, where `words = ⌈size_in_bytes / 32⌉`. The `ceil32` helper rounds up to the nearest 32-byte boundary before dividing.

### 4.2 Incremental Expansion Cost

Instructions never pay the full memory cost from scratch. They pay only the **difference** between the new cost and the cost already paid. This is computed by [`calculate_gas_extend_memory`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/gas.py.html#ethereum.forks.osaka.vm.gas.calculate_gas_extend_memory:0), which returns an [`ExtendMemory`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/gas.py.html#ethereum.forks.osaka.vm.gas.ExtendMemory:0) dataclass:

```python
@dataclass
class ExtendMemory:
    cost: Uint        # Gas to charge for the expansion
    expand_by: Uint   # Number of bytes to append to evm.memory
```

The function accepts the current memory contents and a list of `(start_position, size)` pairs representing all memory regions the instruction will access:

```python
def calculate_gas_extend_memory(
    memory: bytearray,
    extensions: List[Tuple[U256, U256]],
) -> ExtendMemory:
    size_to_extend = Uint(0)
    to_be_paid     = Uint(0)
    current_size   = Uint(len(memory))

    for start_position, size in extensions:
        if size == 0:
            continue
        before_size = ceil32(current_size)
        after_size  = ceil32(Uint(start_position) + Uint(size))
        if after_size <= before_size:
            continue

        size_to_extend += after_size - before_size
        already_paid    = calculate_memory_gas_cost(before_size)
        total_cost      = calculate_memory_gas_cost(after_size)
        to_be_paid     += total_cost - already_paid
        current_size    = after_size

    return ExtendMemory(to_be_paid, size_to_extend)
```

The caller charges `extend_memory.cost` via `charge_gas`, then extends `evm.memory` by `extend_memory.expand_by` bytes.

### 4.3 Cost Reference

| Memory (bytes) | Words | Total cost | Marginal cost per extra word |
| --- | --- | --- | --- |
| 32 | 1 | 3 | 3 |
| 320 | 10 | 30 | 3 |
| 1,024 | 32 | 98 | ~3 |
| 32,768 | 1,024 | 5,120 | ~9 |
| 1,048,576 | 32,768 | 116,608 | ~195 |

The quadratic term grows slowly at typical contract sizes but makes megabyte-scale allocation prohibitively expensive.

---

## 5. Storage Access Costs — EIP-2929

### 5.1 SLOAD

[`sload`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/instructions/storage.py.html#ethereum.forks.osaka.vm.instructions.storage.sload:0) implements the EIP-2929 warm/cold distinction inline:

```python
def sload(evm: Evm) -> None:
    key = pop(evm.stack).to_be_bytes32()

    # GAS: cold or warm?
    if (evm.message.current_target, key) in evm.accessed_storage_keys:
        charge_gas(evm, GAS_WARM_ACCESS)          # 100
    else:
        evm.accessed_storage_keys.add((evm.message.current_target, key))
        charge_gas(evm, GAS_COLD_STORAGE_ACCESS)  # 2100

    value = get_storage(
        evm.message.block_env.state, evm.message.current_target, key
    )
    push(evm.stack, value)
    evm.pc += Uint(1)
```

Two points to note. The stack value is a `U256`; it is converted to `Bytes32` via `.to_be_bytes32()` before being used as a storage key. The slot is added to the warm set **before** the gas is charged, so the charge itself cannot fail on the set-mutation side.

### 5.2 Warm Set Membership

The membership check uses the tuple `(current_target, key)` — both the account address and the slot must match. A slot pre-declared in the transaction’s EIP-2930 access list will be in `evm.accessed_storage_keys` from the start, making its first access warm.

| Access type | Gas cost |
| --- | --- |
| Cold SLOAD (first access to slot this transaction) | 2,100 |
| Warm SLOAD (slot already accessed) | 100 |

---

## 6. SSTORE Gas Rules — EIP-2200 / EIP-3529

SSTORE has the most complex gas logic in the EVM. Its cost depends on three values: `original_value` (the slot’s value at the start of the transaction), `current_value` (the slot’s current value in state), and `new_value` (the value being written).

The full implementation in [`sstore`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/instructions/storage.py.html#ethereum.forks.osaka.vm.instructions.storage.sstore:0):

```python
def sstore(evm: Evm) -> None:
    key       = pop(evm.stack).to_be_bytes32()
    new_value = pop(evm.stack)

    # Guard: must have more than the call stipend remaining (EIP-2200)
    if evm.gas_left <= GAS_CALL_STIPEND:
        raise OutOfGasError

    state = evm.message.block_env.state
    original_value = get_storage_original(state, evm.message.current_target, key)
    current_value  = get_storage(state, evm.message.current_target, key)

    gas_cost = Uint(0)

    # Cold slot surcharge (EIP-2929)
    if (evm.message.current_target, key) not in evm.accessed_storage_keys:
        evm.accessed_storage_keys.add((evm.message.current_target, key))
        gas_cost += GAS_COLD_STORAGE_ACCESS  # 2100

    # Base SSTORE cost
    if original_value == current_value and current_value != new_value:
        if original_value == 0:
            gas_cost += GAS_STORAGE_SET                              # 20000
        else:
            gas_cost += GAS_COLD_STORAGE_WRITE - GAS_COLD_STORAGE_ACCESS  # 2900
    else:
        gas_cost += GAS_WARM_ACCESS                                  # 100

    # Refund calculation
    if current_value != new_value:
        if original_value != 0 and current_value != 0 and new_value == 0:
            # Clearing a slot for the first time this transaction
            evm.refund_counter += REFUND_STORAGE_CLEAR              # +4800

        if original_value != 0 and current_value == 0:
            # Reversing an earlier clear — cancel the previously issued refund
            evm.refund_counter -= REFUND_STORAGE_CLEAR              # -4800

        if original_value == new_value:
            # Slot restored to its transaction-start value
            if original_value == 0:
                evm.refund_counter += int(GAS_STORAGE_SET - GAS_WARM_ACCESS)
                # 20000 - 100 = 19900
            else:
                evm.refund_counter += int(
                    GAS_COLD_STORAGE_WRITE - GAS_COLD_STORAGE_ACCESS - GAS_WARM_ACCESS
                )
                # 5000 - 2100 - 100 = 2800

    charge_gas(evm, gas_cost)

    if evm.message.is_static:
        raise WriteInStaticContext

    set_storage(state, evm.message.current_target, key, new_value)
    evm.pc += Uint(1)
```

### 6.1 Base Cost Logic

The base cost (excluding the cold surcharge) is determined by comparing the slot’s state at the beginning of the transaction (`original_value`) against its current in-memory value:

| Condition | Base cost | Interpretation |
| --- | --- | --- |
| `original == current != new`, `original == 0` | 20,000 (`GAS_STORAGE_SET`) | New slot being set for the first time |
| `original == current != new`, `original != 0` | 2,900 (`GAS_COLD_STORAGE_WRITE − GAS_COLD_STORAGE_ACCESS`) | Existing slot being changed for the first time this tx |
| All other cases | 100 (`GAS_WARM_ACCESS`) | Slot already modified this transaction, or value unchanged |

The cold-slot surcharge of 2,100 (`GAS_COLD_STORAGE_ACCESS`) is added on top of the base cost if the slot is being accessed for the first time this transaction.

### 6.2 Stipend Guard

Before any other work, `sstore` checks:

```python
if evm.gas_left <= GAS_CALL_STIPEND:   # <= 2300
    raise OutOfGasError
```

This prevents a contract from executing `SSTORE` with only the 2,300-gas stipend passed on value-bearing calls, ensuring the caller always retains enough gas to handle the sub-call result.

### 6.3 Refund Logic

`refund_counter` is an `int` (signed) that accumulates across the entire transaction. It can go negative mid-execution (if a reversal occurs) but must be ≥ 0 by the time the transaction finalizes. The three refund events are described in the table below:

| Event | Effect on `refund_counter` |
| --- | --- |
| Slot cleared (`original != 0`, `current != 0`, `new == 0`) | +4,800 |
| Earlier clear reversed (`original != 0`, `current == 0`) | −4,800 |
| Slot restored to `original == 0` | +19,900 |
| Slot restored to `original != 0` | +2,800 |

---

## 7. Transient Storage Gas — EIP-1153

Transient storage (`TLOAD`/`TSTORE`) always costs `GAS_WARM_ACCESS = Uint(100)`, regardless of whether the slot has been accessed before. There is no cold/warm distinction for transient storage, and no refunds.

```python
# tload (storage.py)
def tload(evm: Evm) -> None:
    key = pop(evm.stack).to_be_bytes32()
    charge_gas(evm, GAS_WARM_ACCESS)   # always 100
    value = get_transient_storage(
        evm.message.tx_env.transient_storage, evm.message.current_target, key
    )
    push(evm.stack, value)
    evm.pc += Uint(1)

# tstore (storage.py)
def tstore(evm: Evm) -> None:
    key       = pop(evm.stack).to_be_bytes32()
    new_value = pop(evm.stack)
    charge_gas(evm, GAS_WARM_ACCESS)   # always 100
    if evm.message.is_static:
        raise WriteInStaticContext
    set_transient_storage(
        evm.message.tx_env.transient_storage,
        evm.message.current_target, key, new_value,
    )
    evm.pc += Uint(1)
```

---

## 8. Sub-call Gas — The 63/64 Rule

### 8.1 `max_message_call_gas`

EIP-150 limits the gas a caller can forward to a sub-call to at most 63/64 of its remaining gas. This is implemented in [`max_message_call_gas`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/gas.py.html#ethereum.forks.osaka.vm.gas.max_message_call_gas:0):

```python
def max_message_call_gas(gas: Uint) -> Uint:
    return gas - (gas // Uint(64))
```

### 8.2 `calculate_message_call_gas`

The full gas budget for a call opcode is computed by [`calculate_message_call_gas`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/gas.py.html#ethereum.forks.osaka.vm.gas.calculate_message_call_gas:0), which returns a [`MessageCallGas`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/gas.py.html#ethereum.forks.osaka.vm.gas.MessageCallGas:0):

```python
@dataclass
class MessageCallGas:
    cost: Uint      # Gas charged to the current frame (opcode cost)
    sub_call: Uint  # Gas forwarded to the child frame
```

```python
def calculate_message_call_gas(
    value: U256,
    gas: Uint,
    gas_left: Uint,
    memory_cost: Uint,
    extra_gas: Uint,
    call_stipend: Uint,
) -> MessageCallGas:
    call_stipend = Uint(0) if value == 0 else call_stipend

    if gas_left < extra_gas + memory_cost:
        return MessageCallGas(gas + extra_gas, gas + call_stipend)

    gas = min(gas, max_message_call_gas(gas_left - memory_cost - extra_gas))

    return MessageCallGas(gas + extra_gas, gas + call_stipend)
```

The parameters are:

| Parameter | Meaning |
| --- | --- |
| `value` | ETH value being transferred (determines whether stipend applies) |
| `gas` | Gas requested by the instruction operand |
| `gas_left` | Remaining gas in the parent frame before memory cost |
| `memory_cost` | Gas to pay for memory expansion in the parent |
| `extra_gas` | Additional per-call costs (value transfer, new account) |
| `call_stipend` | `GAS_CALL_STIPEND` (2,300) added to callee on value transfer |

The 63/64 rule is applied **after** deducting `memory_cost` and `extra_gas` from `gas_left`. The stipend is added to `sub_call` only — it inflates what the child receives but is not charged to the parent.

### 8.3 Why 63/64?

The rule guarantees that after any call, the parent retains at least 1/64 of its remaining gas. Applied recursively across 1,024 call frames, this bounds the minimum gas at the deepest frame and prevents call-stack attacks that exhaust gas to a state where the parent cannot execute its cleanup code.

---

## 9. Gas Refunds

### 9.1 Accumulation

`evm.refund_counter` (type `int`, not `Uint`) accumulates refund credits during execution. It is updated directly — there is no separate API. Only `SSTORE` adds to it in normal bytecode execution.

### 9.2 Application

At the end of transaction processing (in `process_transaction` within `fork.py`), the refund is applied:

```python
tx_gas_used = tx.gas - output.gas_left

# EIP-3529: cap refund at 1/5 of gas used
refund = min(output.refund_counter, tx_gas_used // 5)
tx_gas_used_after_refund = tx_gas_used - refund

# EIP-7623: apply calldata floor (Osaka)
tx_gas_used_after_refund = max(
    tx_gas_used_after_refund,
    calldata_floor_gas_cost,
)
```

The refund is bounded by `tx_gas_used // 5` (EIP-3529, London). After the refund, the EIP-7623 calldata floor is applied: if the final gas used falls below the floor computed from calldata token count, it is raised to the floor. This is specific to Osaka and prevents execution gas from effectively subsidizing data-heavy transactions.

### 9.3 Child Frame Refund Propagation

When a child frame succeeds, its `refund_counter` is merged into the parent via [`incorporate_child_on_success`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/__init__.py.html#ethereum.forks.osaka.vm.incorporate_child_on_success:0):

```python
def incorporate_child_on_success(evm: Evm, child_evm: Evm) -> None:
    evm.gas_left      += child_evm.gas_left
    evm.logs          += child_evm.logs
    evm.refund_counter += child_evm.refund_counter   # propagated
    evm.accounts_to_delete.update(child_evm.accounts_to_delete)
    evm.accessed_addresses.update(child_evm.accessed_addresses)
    evm.accessed_storage_keys.update(child_evm.accessed_storage_keys)
```

On error, [`incorporate_child_on_error`](https://ethereum.github.io/execution-specs/src/ethereum/forks/osaka/vm/__init__.py.html#ethereum.forks.osaka.vm.incorporate_child_on_error:0) only returns unused gas — refunds from the failed child are discarded:

```python
def incorporate_child_on_error(evm: Evm, child_evm: Evm) -> None:
    evm.gas_left += child_evm.gas_left
```

### 9.4 Refund Reference

| Event | Amount |
| --- | --- |
| SSTORE: nonzero slot cleared (`original != 0`, `current != 0`, `new == 0`) | +4,800 |
| SSTORE: earlier clear reversed (`original != 0`, `current == 0`) | −4,800 |
| SSTORE: slot restored to original (was 0) | +19,900 |
| SSTORE: slot restored to original (was nonzero) | +2,800 |
| Maximum total refund | min(refund, gas_used / 5) |
| SELFDESTRUCT refund | 0 (removed by EIP-3529) |