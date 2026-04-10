---
title: EELS(6) Message and EVM Execution
published: 2026-04-03
pinned: false
description: EVM architecture and execution loop
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

- [**Preface**](#preface)
- [**Introduction**](#introduction)
- [**1. Execution Environments**](#1-execution-environments)
- [**2. The Message Abstraction**](#2-the-message-abstraction)
- [**3. The Evm Frame**](#3-the-evm-frame)
- [**4. MessageCallOutput**](#4-messagecalloutput)
- [**5. Interpreter Entry Points**](#5-interpreter-entry-points)
- [**6. The Instruction Dispatch Loop**](#6-the-instruction-dispatch-loop)
- [**7. Jump Destination Analysis**](#7-jump-destination-analysis)
- [**8. Precompiled Contracts**](#8-precompiled-contracts)
- [**9. Call Instruction Semantics**](#9-call-instruction-semantics)
- [**10. Error Handling**](#10-error-handling)

## Preface

This article is part of a series analyzing the core components of Ethereum through the **Ethereum Execution Layer Specification (EELS)**.

**What is EELS?** EELS is a Python reference implementation of the Ethereum execution client, designed with a focus on readability and clarity. It serves as a programmer-friendly, up-to-date successor to the original Yellow Paper and is the primary tool for prototyping new Ethereum Improvement Proposals (EIPs). EELS provides complete protocol snapshots at each fork and rendered diffs between them.

**Scope and Implementation** Note that EELS does not implement the JSON-RPC API or P2P networking. To validate blocks, it requires an external RPC provider to fetch data, which EELS then processes and stores in a local database.

**Version Reference** The code analysis in this article is based on the **Osaka** fork:
[ethereum/execution-specs (Osaka)](https://github.com/ethereum/execution-specs/tree/forks/amsterdam/src/ethereum/forks/osaka)

---

## Introduction

This chapter covers the Ethereum Virtual Machine from the ground up: the context structures it operates within, the message abstraction that encapsulates every unit of computation, the `Evm` frame that holds live execution state, and the mechanics of the instruction dispatch loop — including jump destination analysis, precompiled contracts, call instruction semantics, and error handling.

---

## 1. Execution Environments

`Message` does not receive a separate environment argument. Instead, it embeds two environment structures defined in [`vm/__init__.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/__init__.py): `BlockEnvironment` and `TransactionEnvironment`. This keeps all execution context self-contained and eliminates the need to thread environment objects through every call frame.

```
BlockEnvironment  (one per block)
└── TransactionEnvironment  (one per transaction)
    └── Message  (one per call or creation)
        └── Evm  (one execution frame)
            └── Evm  (nested, for sub-calls)
```

### 1.1 BlockEnvironment

[`BlockEnvironment`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/__init__.py) holds block-level constants that are fixed for the duration of an entire block. They are read-only from the EVM’s perspective and correspond to the `BLOCK*` family of opcodes.

```python
# src/ethereum/forks/osaka/vm/__init__.py

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
    excess_blob_gas: U64
    parent_beacon_block_root: Hash32
```

| Field | Opcode | Description |
| --- | --- | --- |
| `chain_id` | `CHAINID` | Network identifier (EIP-155 replay protection) |
| `state` | — | The world state, accessed during execution |
| `block_gas_limit` | — | Maximum gas allowed in this block |
| `block_hashes` | `BLOCKHASH` | Last 256 block hashes |
| `coinbase` | `COINBASE` | Block proposer’s address |
| `number` | `NUMBER` | Current block number |
| `base_fee_per_gas` | `BASEFEE` | EIP-1559 base fee |
| `time` | `TIMESTAMP` | Block timestamp (Unix seconds, as `U256`) |
| `prev_randao` | `PREVRANDAO` | Beacon chain randomness value |
| `excess_blob_gas` | `BLOBBASEFEE` | EIP-4844 blob gas accumulator |
| `parent_beacon_block_root` | — | EIP-4788 parent beacon block root |

### 1.2 TransactionEnvironment

[`TransactionEnvironment`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/__init__.py) holds transaction-level values shared across all call frames spawned by a single transaction.

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

| Field | Opcode | Description |
| --- | --- | --- |
| `origin` | `ORIGIN` | Transaction signer (`tx.origin`) — constant across all frames |
| `gas_price` | `GASPRICE` | Effective gas price paid |
| `gas` | — | Gas allocated at the transaction level |
| `access_list_addresses` | — | Pre-warmed addresses from the EIP-2930 access list |
| `access_list_storage_keys` | — | Pre-warmed storage slots from the EIP-2930 access list |
| `transient_storage` | `TLOAD`/`TSTORE` | EIP-1153 per-transaction transient storage |
| `blob_versioned_hashes` | `BLOBHASH` | EIP-4844 blob commitments |
| `authorizations` | — | EIP-7702 delegation authorizations |
| `index_in_block` | — | Transaction position within the block |
| `tx_hash` | — | Transaction hash |

`origin` is fixed for the lifetime of a transaction and never changes across call frames. `Message.caller`, by contrast, identifies the immediate sender of each individual call and changes with every `CALL`, `DELEGATECALL`, etc. For example:

```
Transaction from Alice → Contract A → Contract B

tx_env.origin  = Alice  (constant across all frames)
Message 1: caller = Alice,      target = Contract A
Message 2: caller = Contract A, target = Contract B
```

---

## 2. The Message Abstraction

[`Message`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/__init__.py) represents a single unit of EVM computation. Every `CALL`, `STATICCALL`, `DELEGATECALL`, `CALLCODE`, `CREATE`, and `CREATE2` instruction constructs a new `Message` for the child frame. The top-level message for a transaction is built by [`prepare_message`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/utils/message.py).

```python
# src/ethereum/forks/osaka/vm/__init__.py

@dataclass
class Message:
    block_env: BlockEnvironment
    tx_env: TransactionEnvironment
    caller: Address
    target: Union[Bytes0, Address]
    current_target: Address
    gas: Uint
    value: U256
    data: Bytes
    code_address: Optional[Address]
    code: Bytes
    depth: Uint
    should_transfer_value: bool
    is_static: bool
    accessed_addresses: Set[Address]
    accessed_storage_keys: Set[Tuple[Address, Bytes32]]
    disable_precompiles: bool
    parent_evm: Optional["Evm"]
```

### Field Reference

**`block_env` / `tx_env`**: Embedded references to block and transaction contexts. Child frames copy these references from their parent, so all frames in a transaction share the same block-level and transaction-level data without duplication.

**`caller`**: The address that initiated this specific call — `msg.sender` in Solidity. For the outermost transaction message, `caller == tx_env.origin`. For `DELEGATECALL`, `caller` is preserved from the parent frame, not set to the current contract’s address.

**`target`**: The destination address provided by the sender. The union type distinguishes the two fundamental execution modes:
- `Bytes0(b"")` — contract creation. No target exists yet; the deployment address is computed before execution and stored in `current_target`.
- `Address` — a regular call. Code at `target` is loaded and executed.

**`current_target`**: The address whose storage context is active for this frame. In normal calls, `current_target == target`. In `DELEGATECALL`, it remains the calling contract’s address so that the callee’s code runs against the caller’s storage.

**`code_address`**: Where the bytecode is loaded from. For most calls, `code_address == current_target`. For `DELEGATECALL` and EIP-7702 delegation, `code_address` points to the contract whose code is being borrowed while `current_target` retains the storage context.

**`code`**: The resolved bytecode that will be executed. In `process_message_call`, the code is fetched from state using `code_address` before the `Evm` frame is constructed, including resolving any EIP-7702 delegation chain (see [§5.1](about:blank#51-process_message_call)).

**`data`**: Serves a dual role — for calls it is calldata (`msg.data`); for contract creation (`target == Bytes0`) it is the initialization bytecode.

**`depth`**: Current call stack depth. The interpreter enforces `STACK_DEPTH_LIMIT = Uint(1024)`; any message that would exceed this limit fails immediately without entering execution.

**`should_transfer_value`**: Controls whether `value` wei is actually moved from `caller` to `current_target`. It is `False` for `STATICCALL` and `DELEGATECALL`.

**`is_static`**: Marks a read-only execution context. When `True`, any state-modifying opcode (`SSTORE`, `LOG*`, `CREATE`, `SELFDESTRUCT`, non-zero-value `CALL`) raises an exception. `STATICCALL` sets this flag, and child frames inherit it.

**`accessed_addresses` / `accessed_storage_keys`**: The EIP-2929 warm sets for this frame. Initialised by copying the parent frame’s warm sets at call time, so addresses warmed in a parent remain warm in the child. Precompile addresses and the transaction origin are pre-warmed at the transaction level.

**`disable_precompiles`**: Set to `True` when the target is an EIP-7702 delegated account whose delegation points to an address in the precompile range. In this case the code at the delegation target is executed as a regular contract rather than dispatching to the precompile.

**`parent_evm`**: Reference to the calling `Evm` frame. Used by `incorporate_child_on_success` and `incorporate_child_on_error` to propagate `return_data`, gas changes, and accumulated state (logs, accounts to delete) back to the parent after the child frame completes.

### Execution Variants

The same `Message` type covers every variant of EVM computation:

| Variant | `target` | `current_target` | `code_address` | `should_transfer_value` | `is_static` |
| --- | --- | --- | --- | --- | --- |
| `CALL` | destination | destination | destination | `True` | inherited |
| `STATICCALL` | destination | destination | destination | `False` | `True` |
| `DELEGATECALL` | destination | calling contract | destination | `False` | inherited |
| `CALLCODE` | destination | calling contract | destination | `True` | inherited |
| `CREATE` / `CREATE2` | `Bytes0(b"")` | new address | `None` | `True` | `False` |
| System transaction | system contract | system contract | system contract | `False` | `False` |

For system transactions, both `caller` and `tx_env.origin` are set to `SYSTEM_ADDRESS = 0xfffffffffffffffffffffffffffffffffffffffe`. System transactions are discussed in a later chapter.

---

## 3. The Evm Frame

[`Evm`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/__init__.py) is the complete mutable state of a single call frame. It is created fresh at the start of every `process_message` invocation and discarded — or merged into the parent — when that frame returns.

```python
# src/ethereum/forks/osaka/vm/__init__.py

@dataclass
class Evm:
    """The internal state of the virtual machine."""
    pc: Uint
    stack: List[U256]
    memory: bytearray
    code: Bytes
    gas_left: Uint
    valid_jump_destinations: Set[Uint]
    logs: Tuple[Log, ...]
    refund_counter: int
    running: bool
    message: Message
    output: Bytes
    accounts_to_delete: Set[Address]
    return_data: Bytes
    error: Optional[EthereumException]
    accessed_addresses: Set[Address]
    accessed_storage_keys: Set[Tuple[Address, Bytes32]]
```

| Field | Description |
| --- | --- |
| `pc` | Program counter — index of the next byte to decode in `code`. Every instruction advances `pc`; control-flow instructions (`JUMP`, `JUMPI`) set it directly, all others increment by 1 plus any immediate data. |
| `stack` | Operand stack holding `U256` values. The list grows toward the end; `push` appends, `pop` removes from the end. Max depth 1024. |
| `memory` | Byte-addressable volatile memory local to this frame. Zero-initialised, starts empty, expands on demand. Does not persist across frames or transactions. |
| `code` | Bytecode being executed (copied from `message.code` at frame creation). |
| `gas_left` | Remaining gas for this frame. `charge_gas` decrements this; if it would go negative, `OutOfGasError` is raised. Set to `Uint(0)` on any `ExceptionalHalt`. |
| `valid_jump_destinations` | Pre-computed set of valid `JUMPDEST` offsets. Computed once per frame by [`get_valid_jump_destinations`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/runtime.py) before the loop starts. |
| `logs` | Ordered tuple of `Log` objects accumulated by `LOG0`–`LOG4` in this frame and successful child frames. Discarded on any error. |
| `refund_counter` | Accumulated gas-refund credits (type `int`). Credits originate from `SSTORE` operations. Capped and applied during transaction post-processing. |
| `running` | Loop sentinel. `True` while execution continues; set to `False` by `STOP`, `RETURN`, `REVERT`, or reaching the end of code. |
| `message` | The `Message` that created this frame — provides context to opcodes (`CALLER`, `CALLVALUE`, `CALLDATALOAD`, etc.). |
| `output` | Output buffer populated by `RETURN` or `REVERT`. For contract creation, holds the init code’s return value (the new runtime bytecode). Empty on an exceptional halt. |
| `accounts_to_delete` | Addresses marked by `SELFDESTRUCT` in this frame and successful child frames. Discarded on error. |
| `return_data` | Raw output of the most recently completed sub-call. Updated after every `CALL`-family and `CREATE`-family instruction. Exposed via `RETURNDATASIZE` and `RETURNDATACOPY`. |
| `error` | `None` during execution or on success; set to an `EthereumException` subclass on failure. |
| `accessed_addresses` / `accessed_storage_keys` | EIP-2929 warm sets. Initialised directly from `message.accessed_addresses` and `message.accessed_storage_keys` — the **same objects**, not copies. Opcodes that warm new addresses mutate these sets on the `Evm` directly, which is why child-frame message construction copies the parent’s warm sets at call time. |

### Child Frame Merge Helpers

Two helper functions defined in [`vm/__init__.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/__init__.py) propagate frame state back to the parent after a sub-call completes.

[**`incorporate_child_on_success`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/__init__.py) merges gas, logs, refund credits, accounts-to-delete, and both warm sets into the parent:

```python
def incorporate_child_on_success(evm: Evm, child_evm: Evm) -> None:
    evm.gas_left += child_evm.gas_left
    evm.logs += child_evm.logs
    evm.refund_counter += child_evm.refund_counter
    evm.accounts_to_delete.update(child_evm.accounts_to_delete)
    evm.accessed_addresses.update(child_evm.accessed_addresses)
    evm.accessed_storage_keys.update(child_evm.accessed_storage_keys)
```

[**`incorporate_child_on_error`**](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/__init__.py) returns only unused gas to the parent; all other child state is discarded:

```python
def incorporate_child_on_error(evm: Evm, child_evm: Evm) -> None:
    evm.gas_left += child_evm.gas_left
```

This asymmetry is the mechanism by which a reverted sub-call rolls back its own state changes, logs, and deletions while still returning any unspent gas to the caller.

---

## 4. MessageCallOutput

[`MessageCallOutput`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/interpreter.py) is returned by the top-level interpreter entry points and carries the complete result of a message execution:

```python
# src/ethereum/forks/osaka/vm/interpreter.py

@dataclass
class MessageCallOutput:
    gas_left: Uint
    refund_counter: U256
    logs: Tuple[Log, ...]
    accounts_to_delete: Set[Address]
    error: Optional[EthereumException]
    return_data: Bytes
```

**`gas_left`**: Remaining gas after execution, to be refunded to the sender.

**`refund_counter`**: EIP-3529 gas refund credits accumulated during execution, primarily from `SSTORE` operations that clear storage. Type is `U256`. The caller in `process_transaction` caps the effective refund to `gas_used // MAX_REFUND_QUOTIENT` before crediting it back.

**`logs`**: Ordered sequence of `Log` objects emitted during execution. On error, this is the empty tuple — logs from a reverted execution are discarded entirely.

**`accounts_to_delete`**: Set of addresses marked for deletion by `SELFDESTRUCT`. Like logs, this is the empty set on error. Actual account deletion is deferred and happens in `process_transaction` after the message call returns.

**`error`**: `None` on success, an `EthereumException` subclass on failure. The type encodes the failure mode: `Revert` for an explicit `REVERT` opcode, `OutOfGasError` for gas exhaustion, `StackOverflowError`/`StackUnderflowError` for stack violations, `InvalidOpcode` for an unrecognised opcode, and so on.

**`return_data`**: The byte string produced by execution — the ABI-encoded return value for successful calls, the deployed bytecode for successful contract creation, or the revert reason data for `REVERT`.

---

## 5. Interpreter Entry Points

All EVM execution flows through [`vm/interpreter.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/interpreter.py). The key constants are:

```python
# src/ethereum/forks/osaka/vm/interpreter.py

STACK_DEPTH_LIMIT  = Uint(1024)
MAX_CODE_SIZE      = 0x6000        # 24576 bytes
MAX_INIT_CODE_SIZE = 2 * MAX_CODE_SIZE  # 49152 bytes (EIP-3860)
```

### 5.1 `process_message_call`

[`process_message_call(message: Message) -> MessageCallOutput`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/interpreter.py) is the single entry point for all EVM execution, called once per transaction by `process_transaction` in [`fork.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/fork.py).

```python
def process_message_call(message: Message) -> MessageCallOutput:
    block_env = message.block_env
    refund_counter = U256(0)

    if message.target == Bytes0(b""):
        # Contract creation path
        is_collision = (
            account_has_code_or_nonce(block_env.state, message.current_target)
            or account_has_storage(block_env.state, message.current_target)
        )
        if is_collision:
            return MessageCallOutput(
                gas_left=Uint(0),
                refund_counter=U256(0),
                logs=tuple(),
                accounts_to_delete=set(),
                error=AddressCollision(),
                return_data=Bytes(b""),
            )
        else:
            evm = process_create_message(message)
    else:
        # Call path
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

The function operates in three distinct phases before delegating to a lower-level entry point:

**Address collision check (creation path)**: Before entering `process_create_message`, the function checks whether the pre-computed deployment address is occupied — by an account with code or non-zero nonce, or by an account with existing storage. A collision immediately returns `AddressCollision()` with all gas consumed and no state changes.

**EIP-7702 delegation setup (call path)**: If the transaction carries `authorizations`, `set_delegation` processes them first, potentially rewriting target account code to a delegation pointer and accumulating refunds. If the code loaded for the message target is itself a delegation pointer, the function resolves it: it loads the delegated address’s code, updates `message.code` and `message.code_address`, and sets `disable_precompiles = True`.

**Error suppression**: On error, logs and `accounts_to_delete` from the `Evm` frame are cleared. Only `gas_left`, `error`, and `return_data` (the revert payload) are preserved.

### 5.2 `process_create_message`

[`process_create_message(message: Message) -> Evm`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/interpreter.py) handles the contract-creation-specific setup before delegating to the core execution loop:

```python
def process_create_message(message: Message) -> Evm:
    state = message.block_env.state
    transient_storage = message.tx_env.transient_storage

    begin_transaction(state, transient_storage)
    mark_account_created(state, message.current_target)
    increment_nonce(state, message.current_target)

    evm = process_message(message)

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

    return evm
```

Key steps in order:

1. **Snapshot**: `begin_transaction` pushes a rollback point covering both world state and transient storage.
2. **Mark created**: `mark_account_created` records the new address so that `get_storage_original` (EIP-2200) treats its pre-existing storage as empty, and so that `SELFDESTRUCT` (EIP-6780) can check whether destruction is permitted.
3. **Set nonce to 1**: Per EIP-161, `increment_nonce` is called on an account that starts at nonce 0. This prevents address reuse attacks where a destroyed contract could be recreated at the same address.
4. **Execute init code**: `process_message` runs `message.code` (the init code). The `RETURN` output of this execution becomes the deployed runtime bytecode.
5. **Validate prefix**: Bytecode beginning with `0xEF` is rejected via `InvalidContractPrefix` — this byte range is reserved for the EVM Object Format (EOF) container format.
6. **Charge and size-check**: The code deposit cost `GAS_CODE_DEPOSIT_PER_BYTE × len(code)` is charged. If the result exceeds `MAX_CODE_SIZE` (24576 bytes), `OutOfGasError` is raised. Both errors are caught by the same `except ExceptionalHalt` block, which zeroes all gas and rolls back state.
7. **Commit or rollback**: On success, `set_code` stores the runtime bytecode and `commit_transaction` makes all changes permanent. On any error, `rollback_transaction` restores the pre-creation state.

### 5.3 `process_message`

[`process_message(message: Message) -> Evm`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/interpreter.py) is the shared core used by both the call and creation paths. It initialises an `Evm` frame, takes a state snapshot, optionally transfers value, and then dispatches to either a precompile or the main opcode decode-execute loop.

```python
def process_message(message: Message) -> Evm:
    state = message.block_env.state

    # 1. Depth check before any state changes
    if message.depth > STACK_DEPTH_LIMIT:
        raise StackDepthLimitError("Stack depth limit reached")

    transient_storage = message.tx_env.transient_storage
    code = message.code
    valid_jump_destinations = get_valid_jump_destinations(code)

    # 2. Initialise the Evm frame
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
        message=message,
        output=b"",
        accounts_to_delete=set(),
        return_data=b"",
        error=None,
        accessed_addresses=message.accessed_addresses,
        accessed_storage_keys=message.accessed_storage_keys,
    )

    # 3. Snapshot state before any mutations
    begin_transaction(state, transient_storage)

    # 4. Value transfer
    if message.should_transfer_value and message.value != 0:
        move_ether(state, message.caller, message.current_target, message.value)

    try:
        # 5. Execution dispatch
        if evm.message.code_address in PRE_COMPILED_CONTRACTS:
            if not message.disable_precompiles:
                evm_trace(evm, PrecompileStart(evm.message.code_address))
                PRE_COMPILED_CONTRACTS[evm.message.code_address](evm)
                evm_trace(evm, PrecompileEnd())
        else:
            # Main opcode decode-execute loop
            while evm.running and evm.pc < ulen(evm.code):
                try:
                    op = Ops(evm.code[evm.pc])
                except ValueError as e:
                    raise InvalidOpcode(evm.code[evm.pc]) from e

                evm_trace(evm, OpStart(op))
                op_implementation[op](evm)
                evm_trace(evm, OpEnd())

            evm_trace(evm, EvmStop(Ops.STOP))

    # 6. Exception handling
    except ExceptionalHalt as error:
        evm_trace(evm, OpException(error))
        evm.gas_left = Uint(0)
        evm.output = b""
        evm.error = error
    except Revert as error:
        evm_trace(evm, OpException(error))
        evm.error = error

    # 7. Commit or rollback
    if evm.error:
        rollback_transaction(state, transient_storage)
    else:
        commit_transaction(state, transient_storage)

    return evm
```

Key design points:

**Depth check first.** The call-stack depth is validated before the `Evm` frame is created and before any snapshot is taken. `StackDepthLimitError` is an `ExceptionalHalt` subclass, so the calling instruction (e.g. a `CALL` in the parent frame) sees it as a failed sub-call and pushes `U256(0)` to the parent stack.

**Snapshot before execution.** `begin_transaction` takes a checkpoint of the world state and transient storage before value transfer or bytecode execution begins. This ensures that even an error raised during value transfer rolls back to a clean pre-frame state.

**Value transfer before bytecode.** `move_ether` runs before the opcode loop or precompile call. If the recipient is a contract, ETH is already credited by the time the first opcode executes.

**Symmetric commit/rollback.** Whether the frame completes normally, executes `RETURN`/`REVERT`, or hits an exception, the final `if evm.error` branch ensures that exactly one of `commit_transaction` or `rollback_transaction` is called.

**Loop termination.** The loop exits normally when `evm.pc >= len(evm.code)`, which is treated identically to a `STOP` instruction — the frame halts successfully with an empty output.

---

## 6. The Instruction Dispatch Loop

Opcodes are defined in the [`Ops` enum](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/__init__.py) and their implementations are organised into categorised modules under [`vm/instructions/`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/):

| Module | Opcodes |
| --- | --- |
| [`arithmetic.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/arithmetic.py) | `ADD`, `MUL`, `SUB`, `DIV`, `SDIV`, `MOD`, `SMOD`, `ADDMOD`, `MULMOD`, `EXP`, `SIGNEXTEND` |
| [`comparison.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/comparison.py) | `LT`, `GT`, `SLT`, `SGT`, `EQ`, `ISZERO` |
| [`bitwise.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/bitwise.py) | `AND`, `OR`, `XOR`, `NOT`, `BYTE`, `SHL`, `SHR`, `SAR`, `CLZ` (Osaka, EIP-7939) |
| [`keccak.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/keccak.py) | `KECCAK256` |
| [`environment.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/environment.py) | `ADDRESS`, `BALANCE`, `ORIGIN`, `CALLER`, `CALLVALUE`, `CALLDATALOAD`, `CALLDATASIZE`, `CALLDATACOPY`, `CODESIZE`, `CODECOPY`, `GASPRICE`, `EXTCODESIZE`, `EXTCODECOPY`, `RETURNDATASIZE`, `RETURNDATACOPY`, `EXTCODEHASH`, `BLOBHASH`, `BLOBBASEFEE` |
| [`block.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/block.py) | `BLOCKHASH`, `COINBASE`, `TIMESTAMP`, `NUMBER`, `PREVRANDAO`, `GASLIMIT`, `CHAINID`, `SELFBALANCE`, `BASEFEE` |
| [`memory.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/memory.py) | `MLOAD`, `MSTORE`, `MSTORE8`, `MSIZE`, `MCOPY` |
| [`storage.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/storage.py) | `SLOAD`, `SSTORE`, `TLOAD`, `TSTORE` |
| [`control_flow.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/control_flow.py) | `STOP`, `JUMP`, `JUMPI`, `PC`, `GAS`, `JUMPDEST`, `PUSH0` |
| [`stack.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/stack.py) | `PUSH1`–`PUSH32`, `DUP1`–`DUP16`, `SWAP1`–`SWAP16`, `POP` |
| [`log.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/log.py) | `LOG0`–`LOG4` |
| [`system.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/system.py) | `CREATE`, `CREATE2`, `CALL`, `CALLCODE`, `DELEGATECALL`, `STATICCALL`, `RETURN`, `REVERT`, `SELFDESTRUCT` |

The `op_implementation` dictionary (defined in [`instructions/__init__.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/__init__.py)) maps each `Ops` member to its function. The loop in `process_message` resolves a single byte to an `Ops` variant; an unrecognised byte raises `InvalidOpcode` before any gas is charged.

### Instruction Structure

Every instruction function takes a single `Evm` argument and mutates it in place. The canonical pattern is:

1. **Pop inputs** from `evm.stack`.
2. **Charge gas** (raises `OutOfGasError` if `gas_left` is insufficient).
3. **Perform the operation** — compute a result, read/write state or memory, or initiate a sub-call.
4. **Push outputs** to `evm.stack` if the instruction produces a value.
5. **Advance `evm.pc`** — typically `evm.pc += Uint(1)`, except for `JUMP`/`JUMPI` (which set `pc` directly) and halting instructions (`STOP`, `RETURN`, `REVERT`, `SELFDESTRUCT`) which set `evm.running = False` or raise.

---

## 7. Jump Destination Analysis

Before the execution loop starts, [`get_valid_jump_destinations`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/runtime.py) performs a single linear scan over the bytecode to build the set of offsets that `JUMP` and `JUMPI` are allowed to target:

```python
# src/ethereum/forks/osaka/vm/runtime.py

def get_valid_jump_destinations(code: Bytes) -> Set[Uint]:
    valid_jump_destinations = set()
    pc = Uint(0)

    while pc < ulen(code):
        try:
            current_opcode = Ops(code[pc])
        except ValueError:
            # Skip invalid opcodes — they are caught at runtime if reached.
            pc += Uint(1)
            continue

        if current_opcode == Ops.JUMPDEST:
            valid_jump_destinations.add(pc)
        elif Ops.PUSH1.value <= current_opcode.value <= Ops.PUSH32.value:
            # Skip the N immediate data bytes that follow PUSH-N
            push_data_size = current_opcode.value - Ops.PUSH1.value + 1
            pc += Uint(push_data_size)

        pc += Uint(1)

    return valid_jump_destinations
```

A byte offset is a valid jump destination if and only if it falls within the bytecode boundary, its value is `0x5B` (`JUMPDEST`), and it is **not** part of the data payload of a preceding `PUSH-N` opcode. The scan enforces the third condition by advancing `pc` past each `PUSH`’s immediate bytes:

```
Offset  0: PUSH1  (0x60)
Offset  1: 0x5B   ← immediate data byte, skipped by the scanner
Offset  2: JUMPDEST (0x5B) ← valid
```

Because the scanner skips offset 1, it is never added to the set. Attempting to `JUMP` to offset 1 raises `InvalidJumpDestError`.

The pre-computation is O(n) in bytecode length and done once per frame. All subsequent `JUMP`/`JUMPI` validations are O(1) set lookups.

---

## 8. Precompiled Contracts

Precompiled contracts are native Python implementations of computationally intensive operations exposed at fixed addresses. They are called through the same `CALL`-family opcodes as regular contracts, but the interpreter short-circuits directly to a native function rather than running bytecode.

### 8.1 Address Mapping

Precompile addresses and implementations are collected into [`PRE_COMPILED_CONTRACTS`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/mapping.py) in `mapping.py`:

```python
# src/ethereum/forks/osaka/vm/precompiled_contracts/mapping.py

PRE_COMPILED_CONTRACTS: Dict[Address, Callable] = {
    ECRECOVER_ADDRESS:               ecrecover,              # 0x01
    SHA256_ADDRESS:                  sha256,                 # 0x02
    RIPEMD160_ADDRESS:               ripemd160,              # 0x03
    IDENTITY_ADDRESS:                identity,               # 0x04
    MODEXP_ADDRESS:                  modexp,                 # 0x05
    ALT_BN128_ADD_ADDRESS:           alt_bn128_add,          # 0x06
    ALT_BN128_MUL_ADDRESS:           alt_bn128_mul,          # 0x07
    ALT_BN128_PAIRING_CHECK_ADDRESS: alt_bn128_pairing_check, # 0x08
    BLAKE2F_ADDRESS:                 blake2f,                # 0x09
    POINT_EVALUATION_ADDRESS:        point_evaluation,       # 0x0a
    BLS12_G1_ADD_ADDRESS:            bls12_g1_add,           # 0x0b
    BLS12_G1_MSM_ADDRESS:            bls12_g1_msm,           # 0x0c
    BLS12_G2_ADD_ADDRESS:            bls12_g2_add,           # 0x0d
    BLS12_G2_MSM_ADDRESS:            bls12_g2_msm,           # 0x0e
    BLS12_PAIRING_ADDRESS:           bls12_pairing,          # 0x0f
    BLS12_MAP_FP_TO_G1_ADDRESS:      bls12_map_fp_to_g1,     # 0x10
    BLS12_MAP_FP2_TO_G2_ADDRESS:     bls12_map_fp2_to_g2,    # 0x11
    P256VERIFY_ADDRESS:              p256verify,             # 0x100
}
```

### 8.2 Complete Precompile Table

| Address | Name | Purpose | Introduced |
| --- | --- | --- | --- |
| `0x01` | [`ecrecover`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/ecrecover.py) | secp256k1 ECDSA signature recovery | Frontier |
| `0x02` | [`sha256`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/sha256.py) | SHA-256 hash | Frontier |
| `0x03` | [`ripemd160`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/ripemd160.py) | RIPEMD-160 hash | Frontier |
| `0x04` | [`identity`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/identity.py) | Copy input to output | Frontier |
| `0x05` | [`modexp`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/modexp.py) | Modular exponentiation | Byzantium |
| `0x06` | [`alt_bn128_add`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/alt_bn128.py) | BN254 elliptic curve point addition | Byzantium |
| `0x07` | [`alt_bn128_mul`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/alt_bn128.py) | BN254 scalar multiplication | Byzantium |
| `0x08` | [`alt_bn128_pairing_check`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/alt_bn128.py) | BN254 pairing check | Byzantium |
| `0x09` | [`blake2f`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/blake2f.py) | BLAKE2b F compression | Istanbul |
| `0x0a` | [`point_evaluation`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/point_evaluation.py) | KZG polynomial commitment verification | Cancun (EIP-4844) |
| `0x0b` | `bls12_g1_add` | BLS12-381 G1 point addition | Prague (EIP-2537) |
| `0x0c` | `bls12_g1_msm` | BLS12-381 G1 multi-scalar multiplication | Prague |
| `0x0d` | `bls12_g2_add` | BLS12-381 G2 point addition | Prague |
| `0x0e` | `bls12_g2_msm` | BLS12-381 G2 multi-scalar multiplication | Prague |
| `0x0f` | `bls12_pairing` | BLS12-381 pairing check | Prague |
| `0x10` | `bls12_map_fp_to_g1` | BLS12-381 field element → G1 | Prague |
| `0x11` | `bls12_map_fp2_to_g2` | BLS12-381 field element → G2 | Prague |
| `0x100` | [`p256verify`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/precompiled_contracts/p256verify.py) | secp256r1 (P-256) ECDSA verification | **Osaka** (EIP-7951) |

### 8.3 Dispatch Mechanism

Precompile dispatch happens inside `process_message` after the `Evm` frame is initialised and value transfer is complete:

```python
if evm.message.code_address in PRE_COMPILED_CONTRACTS:
    if not message.disable_precompiles:
        evm_trace(evm, PrecompileStart(evm.message.code_address))
        PRE_COMPILED_CONTRACTS[evm.message.code_address](evm)
        evm_trace(evm, PrecompileEnd())
```

Each precompile function receives the live `Evm` object and mutates it in place — charging gas against `evm.gas_left`, writing its result to `evm.output`, and setting `evm.error` on failure. An `ExceptionalHalt` (e.g., `OutOfGasError`) propagates to the enclosing `try/except` in `process_message` and is handled identically to a failed bytecode frame.

### 8.4 The `disable_precompiles` Flag

`Message.disable_precompiles` is set to `True` by `process_message_call` when the call target’s code is an EIP-7702 delegation designator (prefix `0xef0100`). In that case, `message.code` is rewritten to the delegated contract’s code, and the precompile shortcut is skipped — even if the delegation happens to point at a precompile address — to prevent the delegated code from being treated as a precompile.

---

## 9. Call Instruction Semantics

Call and creation opcodes are implemented in [`vm/instructions/system.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/instructions/system.py). Each opcode constructs a new `Message`, calls the appropriate interpreter entry point, then incorporates the child result back into the parent frame via `incorporate_child_on_success` or `incorporate_child_on_error`.

### 9.1 CREATE and CREATE2

Both creation opcodes share a `generic_create` helper. The key difference is how the deployment address is computed beforehand:

```python
# CREATE: address from sender's current nonce
contract_address = compute_contract_address(
    evm.message.current_target,
    get_account(evm.message.block_env.state, evm.message.current_target).nonce,
)

# CREATE2: address from sender, salt, and init code hash
contract_address = compute_create2_contract_address(
    evm.message.current_target,
    salt,
    memory_read_bytes(evm.memory, memory_start_position, memory_size),
)
```

Inside `generic_create`, the child `Message` is constructed with `target=Bytes0()` and passed to `process_create_message`:

```python
# src/ethereum/forks/osaka/vm/instructions/system.py

child_message = Message(
    block_env=evm.message.block_env,
    tx_env=evm.message.tx_env,
    caller=evm.message.current_target,
    target=Bytes0(),
    gas=create_message_gas,
    value=endowment,
    data=b"",
    code=call_data,             # init code from memory
    current_target=contract_address,
    depth=evm.message.depth + Uint(1),
    code_address=None,
    should_transfer_value=True,
    is_static=False,
    accessed_addresses=evm.accessed_addresses.copy(),
    accessed_storage_keys=evm.accessed_storage_keys.copy(),
    disable_precompiles=False,
    parent_evm=evm,
)
child_evm = process_create_message(child_message)

if child_evm.error:
    incorporate_child_on_error(evm, child_evm)
    evm.return_data = child_evm.output
    push(evm.stack, U256(0))
else:
    incorporate_child_on_success(evm, child_evm)
    evm.return_data = b""
    push(evm.stack, U256.from_be_bytes(child_evm.message.current_target))
```

Several pre-flight conditions abort `generic_create` before entering `process_create_message`: insufficient sender balance, nonce overflow (`nonce == 2^64 - 1`), stack depth exceeded, init code exceeding `MAX_INIT_CODE_SIZE` (EIP-3860), or address collision. In the collision and depth-exceeded cases, the creation gas is refunded to the parent frame.

### 9.2 CALL Family

`CALL`, `STATICCALL`, `DELEGATECALL`, and `CALLCODE` each construct a child `Message` through a shared helper and pass it to `process_message`. The differences between them are reflected directly in the `Message` fields:

| Opcode | `caller` | `current_target` | `code_address` | `should_transfer_value` | `is_static` |
| --- | --- | --- | --- | --- | --- |
| `CALL` | `evm.message.current_target` | `to` | `to` | `True` | inherited |
| `STATICCALL` | `evm.message.current_target` | `to` | `to` | `False` | `True` |
| `DELEGATECALL` | `evm.message.caller` | `evm.message.current_target` | `to` | `False` | inherited |
| `CALLCODE` | `evm.message.current_target` | `evm.message.current_target` | `to` | `True` | inherited |

`DELEGATECALL` preserves the parent’s `caller` (not the current contract) and routes all storage access through the parent’s `current_target`. This is the mechanism behind proxy contracts: the implementation’s logic executes, but all `SLOAD`/`SSTORE` operations land in the proxy’s own storage. `CALLCODE` is a legacy variant superseded by `DELEGATECALL`; it differs in that `caller` is set to `current_target` rather than the parent’s caller.

---

## 10. Error Handling

Execution can terminate in three ways: **normal halt**, **revert**, or **exceptional halt**.

### 10.1 Exception Types

All EVM exceptions are defined in [`vm/exceptions.py`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/osaka/vm/exceptions.py):

```python
# src/ethereum/forks/osaka/vm/exceptions.py

class ExceptionalHalt(EthereumException):
    """Indicates an exceptional halt; all remaining gas is consumed."""

class Revert(EthereumException):
    """Raised by the REVERT opcode; remaining gas is preserved."""

# ExceptionalHalt subclasses:
class OutOfGasError(ExceptionalHalt): ...        # gas_left would go negative
class StackUnderflowError(ExceptionalHalt): ...  # pop from empty stack
class StackOverflowError(ExceptionalHalt): ...   # push to a 1024-item stack
class InvalidOpcode(ExceptionalHalt): ...        # unrecognised byte
class InvalidJumpDestError(ExceptionalHalt): ... # JUMP to non-JUMPDEST
class StackDepthLimitError(ExceptionalHalt): ... # depth > 1024
class WriteInStaticContext(ExceptionalHalt): ... # state write in STATICCALL
class InvalidContractPrefix(ExceptionalHalt): ...# deployed code starts with 0xEF
class AddressCollision(ExceptionalHalt): ...     # CREATE to occupied address
class OutOfBoundsRead(ExceptionalHalt): ...      # read beyond buffer boundary
class InvalidParameter(ExceptionalHalt): ...     # invalid precompile parameter
class KZGProofError(ExceptionalHalt): ...        # KZG verification failure
```

### 10.2 Halt Modes Compared

| Aspect | Normal halt | `REVERT` | `ExceptionalHalt` |
| --- | --- | --- | --- |
| Gas consumed | Up to last instruction | Up to `REVERT` instruction | **All remaining gas zeroed** |
| `evm.output` | May be non-empty | Revert reason payload | Empty (`b""`) |
| State changes | **Committed** | **Rolled back** | **Rolled back** |
| Logs | **Kept** | **Discarded** | **Discarded** |

### 10.3 Error Propagation

Exceptions raised inside instruction functions bubble up to the `try/except` block in `process_message`:

```python
except ExceptionalHalt as error:
    evm.gas_left = Uint(0)    # Consume all gas
    evm.output = b""
    evm.error = error
except Revert as error:
    evm.error = error         # gas_left unchanged; output preserved
```

In both cases `evm.error` is set and `rollback_transaction` is subsequently called. The calling instruction (e.g. `CALL` in the parent frame) detects the non-`None` error via `incorporate_child_on_error`, returns only unspent gas to the parent, and pushes `U256(0)` (failure) to the parent stack instead of `U256(1)`.

At the top level, `process_message_call` collects `evm.error` into `MessageCallOutput.error`, which `process_transaction` uses to decide whether to commit or roll back the full transaction.

### 10.4 Stack Depth Limit

The call-stack depth is enforced at the entry of `process_message` before any frame is set up or state is touched:

```python
STACK_DEPTH_LIMIT = Uint(1024)

if message.depth > STACK_DEPTH_LIMIT:
    raise StackDepthLimitError("Stack depth limit reached")
```

`StackDepthLimitError` is an `ExceptionalHalt` subclass, so the calling instruction sees it as a failed sub-call. The parent frame continues running with its own remaining gas. Combined with EIP-150’s 63/64 gas forwarding rule — which ensures each nested call frame receives at most 63/64 of the remaining gas — deeply nested calls run out of gas long before reaching depth 1024 in practice, providing defense-in-depth against call-depth attacks.
