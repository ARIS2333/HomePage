---
title: Geth(7) The EVM Deep Dive
published: 2026-04-16
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

In Chapter 06, we followed a transaction through the `stateTransition` pipeline. At the "EVM dispatch" stage, the pipeline handed off to `evm.Call()` or `evm.Create()` and received back a result. This chapter opens that black box: we'll trace how geth's EVM loads bytecode, fetches opcodes one at a time, charges gas, and mutates the stack, memory, and world state.

Here is the lifecycle of a single EVM execution:

```
stateTransition.execute()
│
├─ evm.Call(sender, to, input, gas, value)
│   ├─ depth check (> 1024?)
│   ├─ balance check
│   ├─ snapshot state
│   ├─ precompile?  ──→  RunPrecompiledContract()
│   └─ resolveCode(addr) ──→ NewContract() ──→ evm.Run()
│
└─ evm.Run(contract, input, readOnly)
    ├─ depth++
    ├─ allocate Stack + Memory + ScopeContext
    │
    └─ for { ──────────────────────────── main loop
         ├─ op = contract.GetOp(pc)
         ├─ operation = jumpTable[op]
         ├─ validate stack (min/max)
         ├─ charge constantGas
         ├─ if dynamicGas: compute + charge
         ├─ if memorySize: expand memory
         ├─ operation.execute(&pc, evm, scope)
         └─ pc++
       }
```

The top half (`Call`) sets up the execution context. The bottom half (`Run`) is the interpreter loop that executes bytecode instruction by instruction. We'll cover both, starting from the outside in.

---

## The `EVM` Struct

The `EVM` struct in `core/vm/evm.go` is the central object for all bytecode execution within a block. A single `EVM` instance is created once per block and reused across all transactions — only the transaction-level context is swapped between calls.

```go
type EVM struct {
    Context BlockContext
    TxContext

    StateDB     StateDB
    table       *JumpTable
    depth       int
    chainConfig *params.ChainConfig
    chainRules  params.Rules
    Config      Config
    abort       atomic.Bool
    callGasTemp uint64

    precompiles map[common.Address]PrecompiledContract
    jumpDests   JumpDestCache
    hasher      crypto.KeccakState
    hasherBuf   common.Hash
    readOnly    bool
    returnData  []byte
}
```

Key fields:

- **`Context`** (`BlockContext`): block-level data — `Coinbase`, `BlockNumber`, `Time`, `Difficulty`, `BaseFee`, `BlobBaseFee`, `GasLimit`, `Random`, plus function pointers for `CanTransfer`, `Transfer`, and `GetHash`. These feed opcodes like `COINBASE`, `NUMBER`, `BASEFEE`, `BLOBBASEFEE`.
- **`TxContext`** (embedded): per-transaction data — `Origin` (the EOA that signed), `GasPrice`, `BlobHashes`, `BlobFeeCap`, and `AccessEvents`. Swapped via `SetTxContext()` between transactions.
- **`StateDB`**: the `vm.StateDB` interface for all state reads and writes (balances, storage, code). The EVM never imports `core/state` directly — it only sees this interface.
- **`table`**: a pointer to the current fork's `JumpTable` — the 256-entry array that maps each opcode to its handler, gas cost, and stack bounds.
- **`depth`**: the current call stack depth, incremented by `Run()` and checked against `params.CallCreateDepth` (1024).
- **`readOnly`**: set to `true` during `StaticCall`. Any opcode that modifies state (SSTORE, LOG, CREATE, SELFDESTRUCT) checks this flag and returns `ErrWriteProtection`.
- **`returnData`**: the raw return bytes from the most recent sub-call, accessible via `RETURNDATASIZE` / `RETURNDATACOPY`.
- **`precompiles`**: the map of precompiled contracts active for the current fork.
- **`callGasTemp`**: a temporary variable used to pass the gas amount calculated by `gasCall` (in dynamic gas) to `opCall` (in execute). This exists because gas calculation and opcode execution are separate steps.

### `NewEVM` and Fork Selection

`NewEVM` constructs the EVM for a block. Its most important job is selecting the correct instruction set for the current fork:

```go
func NewEVM(blockCtx BlockContext, statedb StateDB, chainConfig *params.ChainConfig, config Config) *EVM {
    evm := &EVM{
        Context:     blockCtx,
        StateDB:     statedb,
        Config:      config,
        chainConfig: chainConfig,
        chainRules:  chainConfig.Rules(blockCtx.BlockNumber, blockCtx.Random != nil, blockCtx.Time),
        jumpDests:   newMapJumpDests(),
        hasher:      crypto.NewKeccakState(),
    }
    evm.precompiles = activePrecompiledContracts(evm.chainRules)

    switch {
    case evm.chainRules.IsOsaka:
        evm.table = &osakaInstructionSet
    case evm.chainRules.IsVerkle:
        evm.table = &verkleInstructionSet
    case evm.chainRules.IsPrague:
        evm.table = &pragueInstructionSet
    case evm.chainRules.IsCancun:
        evm.table = &cancunInstructionSet
    // ... Shanghai, Merge, London, Berlin, Istanbul,
    //     Constantinople, Byzantium, SpuriousDragon,
    //     TangerineWhistle, Homestead ...
    default:
        evm.table = &frontierInstructionSet
    }
    // ...
}
```

The switch statement falls through from the newest fork to the oldest. A Cancun block gets the `cancunInstructionSet`, which includes all opcodes from Frontier through Cancun. The `chainRules` struct — derived from `ChainConfig` at the block's number and timestamp — drives every fork-dependent decision throughout the EVM.

---

## Call Variants

The `stateTransition` dispatches into the EVM through one of these entry points. From Solidity, the `CALL`, `STATICCALL`, `DELEGATECALL`, and `CREATE` opcodes trigger them recursively for inter-contract communication.

### `Call`

`Call` is the most common entry point — it executes code at a target address. Here is the core logic with the tracer and Verkle code removed for clarity:

```go
func (evm *EVM) Call(caller common.Address, addr common.Address, input []byte, gas uint64, value *uint256.Int) (ret []byte, leftOverGas uint64, err error) {
    if evm.depth > int(params.CallCreateDepth) {
        return nil, gas, ErrDepth
    }
    if !value.IsZero() && !evm.Context.CanTransfer(evm.StateDB, caller, value) {
        return nil, gas, ErrInsufficientBalance
    }
    snapshot := evm.StateDB.Snapshot()
    p, isPrecompile := evm.precompile(addr)

    if !evm.StateDB.Exist(addr) {
        // ...EIP-158: calling a non-existent account with zero value is a no-op
        evm.StateDB.CreateAccount(addr)
    }
    evm.Context.Transfer(evm.StateDB, caller, addr, value)

    if isPrecompile {
        ret, gas, err = RunPrecompiledContract(p, input, gas, evm.Config.Tracer)
    } else {
        code := evm.resolveCode(addr)
        if len(code) == 0 {
            ret, err = nil, nil
        } else {
            contract := NewContract(caller, addr, value, gas, evm.jumpDests)
            contract.SetCallCode(evm.resolveCodeHash(addr), code)
            ret, err = evm.Run(contract, input, false)
            gas = contract.Gas
        }
    }
    if err != nil {
        evm.StateDB.RevertToSnapshot(snapshot)
        if err != ErrExecutionReverted {
            gas = 0
        }
    }
    return ret, gas, err
}
```

The sequence:

1. **Depth check**: if `evm.depth > 1024`, return `ErrDepth`. This prevents infinite recursion between contracts.
2. **Balance check**: if the call transfers value, verify the caller has enough.
3. **Snapshot**: take a state snapshot so we can revert on error.
4. **Precompile check**: if the target address is a precompiled contract, run it directly via `RunPrecompiledContract()` and skip the interpreter.
5. **Resolve code**: load the target's bytecode. Post-Prague, `resolveCode` follows EIP-7702 delegation designators one level deep.
6. **Create contract object**: `NewContract` packages the caller, address, value, gas allowance, and jump destination cache into a `Contract` struct.
7. **Run**: invoke `evm.Run()` — the interpreter loop.
8. **Error handling**: on error, revert state to the snapshot. If the error is `ErrExecutionReverted` (from the `REVERT` opcode), leftover gas is returned to the caller. For all other errors, gas is consumed entirely.

### `StaticCall`

`StaticCall` differs from `Call` in several ways beyond just read-only enforcement:

- **No value transfer**: the value is always zero — there is no `value` parameter in the signature.
- **No account creation**: unlike `Call`, it has no `Exist` / `CreateAccount` logic.
- **Zero-balance touch**: it unconditionally calls `AddBalance(addr, 0)` to "touch" the account, which matters for state accounting on some networks.
- **Read-only mode**: it passes `readOnly = true` to `Run()`. Any state-modifying opcode (SSTORE, LOG, CREATE, SELFDESTRUCT) checks `evm.readOnly` and returns `ErrWriteProtection`.

### `DelegateCall`

`DelegateCall` executes the target address's code but in the context of the *caller*:

```go
contract := NewContract(originCaller, caller, value, gas, evm.jumpDests)
```

Notice the first argument is `originCaller` (the caller's caller), not `caller`. Inside the delegated code, `CALLER` returns the original external caller and `ADDRESS` returns the calling contract's address — not the library's address. This is the mechanism behind Solidity's proxy/library pattern.

### `Create` and `Create2`

Contract creation follows a different path through the private `evm.create()` method:

1. Depth check and balance check (same as Call).
2. **Increment the creator's nonce** — this happens *before* execution, and it's the reason nonce is incremented even if creation fails.
3. **Compute the new address**: `CREATE` uses `keccak256(rlp([sender, nonce]))`. `CREATE2` uses `keccak256(0xff ++ sender ++ salt ++ keccak256(init_code))`.
4. **Collision check**: if the target address already has a non-zero nonce, non-empty code, or non-empty storage, return `ErrContractAddressCollision`.
5. **Create the account**, transfer value, set nonce to 1 (post-EIP-158).
6. **Run init code**: `evm.Run()` executes the constructor bytecode. The returned bytes become the deployed code.
7. **Post-checks**: code size must not exceed `params.MaxCodeSize` (24576 bytes). Post-London, code must not start with `0xEF` (EIP-3541, reserved for EOF).
8. **Charge code storage gas**: `len(deployedCode) * params.CreateDataGas` (200 gas per byte).

---

## The `Contract` Struct

Each call frame gets its own `Contract` object. It holds everything the interpreter needs to execute a single piece of code.

```go
type Contract struct {
    caller  common.Address
    address common.Address

    jumpDests JumpDestCache
    analysis  BitVec

    Code     []byte
    CodeHash common.Hash
    Input    []byte

    IsDeployment bool
    IsSystemCall bool

    Gas   uint64
    value *uint256.Int
}
```

- **`caller`** / **`address`**: who called this contract, and what address it's executing at. For `DelegateCall`, `caller` is the original external caller.
- **`Gas`**: the gas remaining for this call frame. Decremented directly by `UseGas()` as opcodes execute.
- **`Code`** / **`CodeHash`**: the bytecode and its hash. The hash is used to cache JUMPDEST analysis across calls to the same contract.
- **`Input`**: the calldata (set by `Run()` from the `input` parameter).
- **`jumpDests`** / **`analysis`**: JUMPDEST analysis results. The `jumpDests` cache is shared across the entire block's EVM instance, so if contract A calls contract B twice, the second call reuses B's JUMPDEST analysis.

`GetOp` fetches the opcode at a given program counter position, returning `STOP` if the PC exceeds the code length:

```go
func (c *Contract) GetOp(n uint64) OpCode {
    if n < uint64(len(c.Code)) {
        return OpCode(c.Code[n])
    }
    return STOP
}
```

---

## Stack and Memory

Each call frame allocates its own stack and memory from `sync.Pool`s, which are returned when the frame exits.

### Stack

```go
type Stack struct {
    data []uint256.Int
}
```

The EVM stack holds 256-bit integers. The maximum depth is 1024 items — enforced not in the stack itself but by the `maxStack` check in the interpreter loop. The pool pre-allocates a capacity of 16 elements:

```go
var stackPool = sync.Pool{
    New: func() interface{} {
        return &Stack{data: make([]uint256.Int, 0, 16)}
    },
}
```

Stack operations are minimal: `push`, `pop`, `peek` (return a pointer to the top without removing it), `swap1`–`swap16`, `dup`, and `Back(n)` (access the nth item from top). A common pattern in opcode implementations is `pop` + `peek`:

```go
func opAdd(pc *uint64, evm *EVM, scope *ScopeContext) ([]byte, error) {
    x, y := scope.Stack.pop(), scope.Stack.peek()
    y.Add(&x, y)
    return nil, nil
}
```

Here `opAdd` pops the first operand and peeks at the second (leaving it in place), then writes the result directly into `y`. This avoids an extra push — a micro-optimization that appears throughout `instructions.go`.

### Memory

```go
type Memory struct {
    store       []byte
    lastGasCost uint64
}
```

EVM memory is a byte-addressable, word-aligned buffer that grows on demand. The `lastGasCost` field tracks cumulative gas already paid for memory expansion, so that `memoryGasCost()` only charges the delta when memory grows further.

Memory expansion follows a quadratic cost model from the Yellow Paper:

```go
func memoryGasCost(mem *Memory, newMemSize uint64) (uint64, error) {
    // ...
    newMemSizeWords := toWordSize(newMemSize)
    newMemSize = newMemSizeWords * 32

    if newMemSize > uint64(mem.Len()) {
        square := newMemSizeWords * newMemSizeWords
        linCoef := newMemSizeWords * params.MemoryGas
        quadCoef := square / params.QuadCoeffDiv
        newTotalFee := linCoef + quadCoef

        fee := newTotalFee - mem.lastGasCost
        mem.lastGasCost = newTotalFee
        return fee, nil
    }
    return 0, nil
}
```

The formula is `memory_cost = (words * 3) + (words^2 / 512)`. The linear term keeps small reads cheap; the quadratic term makes large memory allocations progressively more expensive. The function calculates the *total* fee for the new size and subtracts the already-paid `lastGasCost` to get the incremental charge.

---

## The `ScopeContext`

The stack, memory, and contract are bundled into a `ScopeContext` — the per-call context that every opcode handler receives:

```go
type ScopeContext struct {
    Memory   *Memory
    Stack    *Stack
    Contract *Contract
}
```

This struct is created once per `Run()` invocation and passed to every `operation.execute()` call.

---

## The Interpreter Loop: `Run()`

`Run()` in `core/vm/interpreter.go` is the heart of the EVM. It's a method on `*EVM` (not a separate interpreter struct) that loops over bytecode and executes opcodes one at a time.

```go
func (evm *EVM) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
    evm.depth++
    defer func() { evm.depth-- }()

    if readOnly && !evm.readOnly {
        evm.readOnly = true
        defer func() { evm.readOnly = false }()
    }
    evm.returnData = nil

    if len(contract.Code) == 0 {
        return nil, nil
    }

    var (
        op        OpCode
        jumpTable *JumpTable = evm.table
        mem       = NewMemory()
        stack     = newstack()
        callContext = &ScopeContext{Memory: mem, Stack: stack, Contract: contract}
        pc        = uint64(0)
        cost      uint64
        // ...
    )
    defer func() {
        returnStack(stack)
        mem.Free()
    }()
    contract.Input = input

    for {
        op = contract.GetOp(pc)
        operation := jumpTable[op]
        cost = operation.constantGas

        // 1. Validate stack bounds
        if sLen := stack.len(); sLen < operation.minStack {
            return nil, &ErrStackUnderflow{stackLen: sLen, required: operation.minStack}
        } else if sLen > operation.maxStack {
            return nil, &ErrStackOverflow{stackLen: sLen, limit: operation.maxStack}
        }

        // 2. Charge constant gas
        if contract.Gas < cost {
            return nil, ErrOutOfGas
        }
        contract.Gas -= cost

        // 3. Compute and charge dynamic gas + memory expansion
        if operation.dynamicGas != nil {
            var memorySize uint64
            if operation.memorySize != nil {
                memSize, overflow := operation.memorySize(stack)
                // ...round up to 32-byte words
                memorySize, overflow = math.SafeMul(toWordSize(memSize), 32)
            }
            dynamicCost, err := operation.dynamicGas(evm, contract, stack, mem, memorySize)
            if contract.Gas < dynamicCost {
                return nil, ErrOutOfGas
            }
            contract.Gas -= dynamicCost
        }

        // 4. Expand memory if needed
        if memorySize > 0 {
            mem.Resize(memorySize)
        }

        // 5. Execute the opcode
        res, err = operation.execute(&pc, evm, callContext)
        if err != nil {
            break
        }
        pc++
    }

    if err == errStopToken {
        err = nil
    }
    return res, err
}
```

Walking through each iteration:

1. **Fetch**: `contract.GetOp(pc)` reads the byte at position `pc` in the bytecode and casts it to an `OpCode`.
2. **Lookup**: `jumpTable[op]` retrieves the `operation` struct — the handler, gas costs, stack bounds, and optional memory size function.
3. **Stack validation**: check that the stack has at least `minStack` items and no more than `maxStack`.
4. **Constant gas**: deduct the fixed gas cost (e.g., 3 for `ADD`, 8 for `JUMP`). If insufficient, return `ErrOutOfGas`.
5. **Dynamic gas**: some operations have variable costs. If `operation.dynamicGas` is set, call it to compute the additional charge. This is also where `memorySize` is calculated via `operation.memorySize`.
6. **Memory expansion**: if the operation requires memory beyond the current allocation, resize.
7. **Execute**: call `operation.execute(&pc, evm, callContext)`. The handler reads/writes the stack and memory, may call into `evm.StateDB`, and may modify `pc` directly (for `JUMP`/`JUMPI`).
8. **Advance**: `pc++`. For opcodes like `PUSH1`–`PUSH32`, the execute function advances `pc` past the embedded data bytes internally.

The loop exits when an opcode returns an error. `STOP` and `RETURN` return the sentinel `errStopToken`, which `Run()` clears to `nil` before returning.

---

## The Jump Table

The jump table is the data structure that defines *what each opcode does* for a given fork. It lives in `core/vm/jump_table.go`.

### The `operation` Struct

```go
type operation struct {
    execute     executionFunc
    constantGas uint64
    dynamicGas  gasFunc
    minStack    int
    maxStack    int
    memorySize  memorySizeFunc
    undefined   bool
}
```

- **`execute`**: the function that performs the operation (e.g., `opAdd`, `opSstore`).
- **`constantGas`**: the fixed gas cost charged on every execution.
- **`dynamicGas`**: an optional function that computes additional gas based on arguments (e.g., memory expansion, cold/warm storage access).
- **`minStack`** / **`maxStack`**: the minimum items required on the stack and the maximum allowed after execution.
- **`memorySize`**: an optional function that computes how much memory the operation needs. If set, `dynamicGas` must also be set.

### Fork Inheritance

The `JumpTable` is a 256-element array — one slot per possible opcode byte:

```go
type JumpTable [256]*operation
```

Each fork builds on the previous one by copying and patching:

```go
func newCancunInstructionSet() JumpTable {
    instructionSet := newShanghaiInstructionSet()
    enable4844(&instructionSet) // BLOBHASH opcode
    enable7516(&instructionSet) // BLOBBASEFEE opcode
    enable1153(&instructionSet) // Transient storage (TLOAD/TSTORE)
    enable5656(&instructionSet) // MCOPY opcode
    enable6780(&instructionSet) // Neutered SELFDESTRUCT
    return validate(instructionSet)
}
```

`newShanghaiInstructionSet()` calls `newMergeInstructionSet()`, which calls `newLondonInstructionSet()`, and so on back to `newFrontierInstructionSet()`. Each `enable*` function overwrites specific slots in the table. The `validate` function panics if any slot is nil or if `memorySize` is set without `dynamicGas`.

This chain of inheritance means you can trace every opcode back to the fork that introduced it. For example, `PUSH0` was added by `enable3855` (EIP-3855) in the Shanghai set.

### Example: How `ADD` Is Registered

In `newFrontierInstructionSet()`:

```go
ADD: {
    execute:     opAdd,
    constantGas: GasFastestStep,
    minStack:    minStack(2, 1),
    maxStack:    maxStack(2, 1),
},
```

`GasFastestStep` is 3 gas. `minStack(2, 1)` means "requires 2 items, will produce 1" — so the stack must have at least 2 items. `maxStack(2, 1)` computes the maximum stack size where adding 1 (net: -1) won't overflow.

---

## Representative Opcodes

With the interpreter loop and jump table understood, let's look at how specific opcodes implement their logic.

### Arithmetic: `opAdd`

```go
func opAdd(pc *uint64, evm *EVM, scope *ScopeContext) ([]byte, error) {
    x, y := scope.Stack.pop(), scope.Stack.peek()
    y.Add(&x, y)
    return nil, nil
}
```

Pop one operand, peek at the second, write the result in-place. All arithmetic uses `uint256.Int` from the `holiman/uint256` library — 256-bit math that maps directly to EVM word size. The `return nil, nil` pattern means "no return data, no error" — the interpreter increments `pc` by 1 and continues.

### Storage Read: `opSload`

```go
func opSload(pc *uint64, evm *EVM, scope *ScopeContext) ([]byte, error) {
    loc := scope.Stack.peek()
    hash := common.Hash(loc.Bytes32())
    val := evm.StateDB.GetState(scope.Contract.Address(), hash)
    loc.SetBytes(val.Bytes())
    return nil, nil
}
```

Peek at the top of stack (the storage key), call `StateDB.GetState()` to read the value, then overwrite the top of stack with the result. The gas cost for SLOAD is handled entirely in the dynamic gas function (not shown here) — either 2100 for a cold slot or 100 for a warm slot under EIP-2929.

### Storage Write: `opSstore`

```go
func opSstore(pc *uint64, evm *EVM, scope *ScopeContext) ([]byte, error) {
    if evm.readOnly {
        return nil, ErrWriteProtection
    }
    loc := scope.Stack.pop()
    val := scope.Stack.pop()
    evm.StateDB.SetState(scope.Contract.Address(), loc.Bytes32(), val.Bytes32())
    return nil, nil
}
```

The execute function is simple — pop key, pop value, write to state. The complexity of SSTORE is entirely in its dynamic gas function (covered below).

### Control Flow: `opJump`

```go
func opJump(pc *uint64, evm *EVM, scope *ScopeContext) ([]byte, error) {
    if evm.abort.Load() {
        return nil, errStopToken
    }
    pos := scope.Stack.pop()
    if !scope.Contract.validJumpdest(&pos) {
        return nil, ErrInvalidJump
    }
    *pc = pos.Uint64() - 1
    return nil, nil
}
```

Pop the destination from the stack. Validate that the destination byte is actually a `JUMPDEST` opcode (not data embedded in a `PUSH`). Set `*pc` to `dest - 1` because the interpreter loop will `pc++` after this returns.

The `validJumpdest` check prevents jumping into the middle of a PUSH instruction's data bytes. It uses a bitmap (`analysis BitVec`) that marks which byte positions are actual opcodes vs. push data. This bitmap is computed once and cached in the `JumpDestCache`.

### Inter-Contract Calls: `opCall`

```go
func opCall(pc *uint64, evm *EVM, scope *ScopeContext) ([]byte, error) {
    stack := scope.Stack
    temp := stack.pop()
    gas := evm.callGasTemp
    addr, value, inOffset, inSize, retOffset, retSize := stack.pop(), stack.pop(), stack.pop(), stack.pop(), stack.pop(), stack.pop()
    toAddr := common.Address(addr.Bytes20())
    args := scope.Memory.GetPtr(inOffset.Uint64(), inSize.Uint64())

    if evm.readOnly && !value.IsZero() {
        return nil, ErrWriteProtection
    }
    if !value.IsZero() {
        gas += params.CallStipend
    }
    ret, returnGas, err := evm.Call(scope.Contract.Address(), toAddr, args, gas, &value)

    if err != nil {
        temp.Clear()
    } else {
        temp.SetOne()
    }
    stack.push(&temp)
    if err == nil || err == ErrExecutionReverted {
        scope.Memory.Set(retOffset.Uint64(), retSize.Uint64(), ret)
    }

    scope.Contract.RefundGas(returnGas, evm.Config.Tracer, tracing.GasChangeCallLeftOverRefunded)
    evm.returnData = ret
    return ret, nil
}
```

Walking through:

- The first `pop` discards the gas argument from the stack — the *actual* gas to forward was already computed by `gasCall` (the dynamic gas function) and stored in `evm.callGasTemp`. This split exists because the 63/64 rule (EIP-150) must be applied during gas calculation, not during execution.
- Pop the remaining 6 arguments: target address, value, input offset/size, return offset/size.
- If the call transfers value, add `CallStipend` (2300 gas) — a free allowance so the receiver can emit a LOG.
- Recursively call `evm.Call()`. This increments `evm.depth`, creates a new `Contract` and `ScopeContext`, and enters `Run()` again.
- Push 1 (success) or 0 (failure) onto the stack.
- Copy return data into memory at the specified offset.
- Refund unused gas back to the current contract.

Note that `opCall` always returns `nil, nil` to the interpreter loop — sub-call errors are reported via the stack value, not by breaking the parent's execution. This is the distinction between *EVM-level errors* (which revert the sub-call) and *consensus errors* (which abort the entire transaction).

---

## Dynamic Gas Costs

Some opcodes have constant gas (e.g., ADD costs 3, always). Others have costs that depend on arguments, state, or memory access patterns. These are computed by `dynamicGas` functions in `core/vm/gas_table.go` and `core/vm/operations_acl.go`.

### Memory Expansion

Memory expansion gas is the most common dynamic cost — it applies to any opcode that accesses memory (MLOAD, MSTORE, CALLDATACOPY, CALL, etc.). We already saw the formula in the Memory section: `cost = 3*words + words^2/512`, charged incrementally.

### SSTORE: The Most Complex Gas Rule

SSTORE gas depends on three values: the **original** value (at the start of the transaction), the **current** value (possibly modified earlier in this transaction), and the **new** value being written. This three-way comparison implements the net gas metering rules from EIP-2200 and EIP-2929.

The post-Berlin SSTORE gas function in `operations_acl.go` handles all modern cases:

```go
func makeGasSStoreFunc(clearingRefund uint64) gasFunc {
    return func(evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
        if contract.Gas <= params.SstoreSentryGasEIP2200 {
            return 0, errors.New("not enough gas for reentrancy sentry")
        }
        var (
            y, x              = stack.Back(1), stack.peek()
            slot              = common.Hash(x.Bytes32())
            current, original = evm.StateDB.GetStateAndCommittedState(contract.Address(), slot)
            cost              = uint64(0)
        )
        if _, slotPresent := evm.StateDB.SlotInAccessList(contract.Address(), slot); !slotPresent {
            cost = params.ColdSloadCostEIP2929
            evm.StateDB.AddSlotToAccessList(contract.Address(), slot)
        }
        value := common.Hash(y.Bytes32())

        if current == value {  // no-op
            return cost + params.WarmStorageReadCostEIP2929, nil
        }
        if original == current {
            if original == (common.Hash{}) {  // 0 → non-zero
                return cost + params.SstoreSetGasEIP2200, nil
            }
            if value == (common.Hash{}) {  // non-zero → 0
                evm.StateDB.AddRefund(clearingRefund)
            }
            return cost + (params.SstoreResetGasEIP2200 - params.ColdSloadCostEIP2929), nil
        }
        // Slot is "dirty" — already modified in this transaction
        // ... (refund adjustments for various reset scenarios)
        return cost + params.WarmStorageReadCostEIP2929, nil
    }
}
```

The key cost tiers:

| Scenario | Gas Cost |
|----------|----------|
| No-op (write same value) | 100 (warm read) |
| Clean write: zero → non-zero | 20,000 |
| Clean write: non-zero → different non-zero | 2,900 (5000 - cold sload) |
| Clean write: non-zero → zero | 2,900 + refund |
| Dirty write (slot already changed this tx) | 100 (warm read) |
| Cold slot access (first touch) | + 2,100 |

The function also manages the refund counter: clearing a storage slot (setting to zero) adds to the refund, while recreating a cleared slot subtracts from it. The `clearingRefund` parameter differs between EIP-2200 and EIP-3529.

### EIP-2929: Warm vs. Cold Access

EIP-2929 (Berlin fork) introduced the warm/cold distinction for all state access opcodes. The first time a transaction touches an address or storage slot, it pays a "cold" cost. Subsequent accesses pay only a "warm" cost.

For SLOAD, this is implemented in `gasSLoadEIP2929`:

```go
func gasSLoadEIP2929(evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
    loc := stack.peek()
    slot := common.Hash(loc.Bytes32())
    if _, slotPresent := evm.StateDB.SlotInAccessList(contract.Address(), slot); !slotPresent {
        evm.StateDB.AddSlotToAccessList(contract.Address(), slot)
        return params.ColdSloadCostEIP2929, nil   // 2100
    }
    return params.WarmStorageReadCostEIP2929, nil  // 100
}
```

For CALL and its variants, `makeCallVariantGasCallEIP2929` wraps the pre-Berlin gas calculator to add cold access charges:

```go
gasCallEIP2929         = makeCallVariantGasCallEIP2929(gasCall, 1)
gasDelegateCallEIP2929 = makeCallVariantGasCallEIP2929(gasDelegateCall, 1)
gasStaticCallEIP2929   = makeCallVariantGasCallEIP2929(gasStaticCall, 1)
```

The wrapper checks if the target address is in the access list. If not, it charges `ColdAccountAccessCostEIP2929 - WarmStorageReadCostEIP2929` (2500) and adds the address.

---

## Precompiled Contracts

Precompiled contracts are native Go implementations of operations that would be prohibitively expensive to execute in EVM bytecode — primarily cryptographic functions.

```go
type PrecompiledContract interface {
    RequiredGas(input []byte) uint64
    Run(input []byte) ([]byte, error)
    Name() string
}
```

Each fork defines a map from address to implementation. Here's the Cancun set (addresses 0x01–0x0a):

| Address | Contract | Description |
|---------|----------|-------------|
| 0x01 | `ecrecover` | ECDSA public key recovery |
| 0x02 | `sha256hash` | SHA-256 hash |
| 0x03 | `ripemd160hash` | RIPEMD-160 hash |
| 0x04 | `dataCopy` | Identity (memory copy) |
| 0x05 | `bigModExp` | Modular exponentiation |
| 0x06 | `bn256AddIstanbul` | BN254 elliptic curve point addition |
| 0x07 | `bn256ScalarMulIstanbul` | BN254 scalar multiplication |
| 0x08 | `bn256PairingIstanbul` | BN254 pairing check |
| 0x09 | `blake2F` | BLAKE2b compression function |
| 0x0a | `kzgPointEvaluation` | KZG point evaluation (EIP-4844) |

Prague adds BLS12-381 operations at 0x0b–0x11, and Osaka adds P-256 signature verification at 0x0100.

When `Call()` detects that the target address is a precompile, it skips the interpreter entirely:

```go
ret, gas, err = RunPrecompiledContract(p, input, gas, evm.Config.Tracer)
```

`RunPrecompiledContract` calls `p.RequiredGas(input)` to compute the flat gas cost, deducts it, and calls `p.Run(input)` to get the result. There is no stack, memory, or opcode loop — just a direct Go function call.

---

## The `vm.StateDB` Interface

The EVM interacts with world state through the `vm.StateDB` interface defined in `core/vm/interface.go`. This is a deliberate abstraction boundary — the EVM package never imports `core/state`.

```go
type StateDB interface {
    CreateAccount(common.Address)
    CreateContract(common.Address)

    SubBalance(common.Address, *uint256.Int, tracing.BalanceChangeReason) uint256.Int
    AddBalance(common.Address, *uint256.Int, tracing.BalanceChangeReason) uint256.Int
    GetBalance(common.Address) *uint256.Int

    GetNonce(common.Address) uint64
    SetNonce(common.Address, uint64, tracing.NonceChangeReason)

    GetCodeHash(common.Address) common.Hash
    GetCode(common.Address) []byte
    SetCode(common.Address, []byte, tracing.CodeChangeReason) []byte
    GetCodeSize(common.Address) int

    AddRefund(uint64)
    SubRefund(uint64)
    GetRefund() uint64

    GetStateAndCommittedState(common.Address, common.Hash) (common.Hash, common.Hash)
    GetState(common.Address, common.Hash) common.Hash
    SetState(common.Address, common.Hash, common.Hash) common.Hash
    GetStorageRoot(addr common.Address) common.Hash

    // Transient storage (EIP-1153)
    GetTransientState(addr common.Address, key common.Hash) common.Hash
    SetTransientState(addr common.Address, key, value common.Hash)

    // Access list operations (EIP-2929)
    AddressInAccessList(addr common.Address) bool
    SlotInAccessList(addr common.Address, slot common.Hash) (addressOk bool, slotOk bool)
    AddAddressToAccessList(addr common.Address)
    AddSlotToAccessList(addr common.Address, slot common.Hash)

    // Snapshot/revert for atomic sub-calls
    Snapshot() int
    RevertToSnapshot(int)

    // Logs, preimages, and more
    AddLog(*types.Log)
    AddPreimage(common.Hash, []byte)
    Exist(common.Address) bool
    Empty(common.Address) bool
    SelfDestruct(common.Address) uint256.Int
    Finalise(bool)
    // ...
}
```

This interface defines every operation an opcode can perform on the world state. The concrete implementation (`core/state.StateDB` or `state.HookedState` for tracing) is injected at construction time. This separation makes the EVM testable in isolation — tests can provide a mock implementation without needing a full state trie.

Notable methods:

- **`GetStateAndCommittedState`**: returns both the current value and the value at the start of the transaction. The SSTORE gas calculation needs both to determine whether the slot is "dirty."
- **`Snapshot` / `RevertToSnapshot`**: used by every CALL variant to atomically revert state changes on failure.
- **`AddRefund` / `SubRefund`**: the refund counter that accumulates gas credits during execution (e.g., from clearing storage slots). Capped and applied in `stateTransition.execute()` after the EVM returns (see Chapter 06).

---

## What's Next

We've now seen both sides of transaction execution — the pipeline in Chapter 06 and the bytecode engine in this chapter. When a transaction arrives, it is validated, gas is bought, the EVM runs the bytecode (or dispatches to a precompile), unused gas is refunded, and a receipt is created.

But transactions don't arrive one at a time — they arrive in floods from the network and sit in a pool waiting to be included in blocks. Chapter 08 covers how geth receives, validates, prioritizes, and evicts pending transactions.
