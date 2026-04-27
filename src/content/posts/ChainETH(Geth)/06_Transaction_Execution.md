---
title: Geth(6) Transaction Execution
published: 2026-04-15
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

This chapter traces a single transaction through geth's execution pipeline — from the moment it is picked from the block to the moment its receipt is produced. Previous chapters covered the data structures (types, state, tries, storage) that make execution possible. This chapter covers the *doing* — the code that actually changes Ethereum's world state.

The pipeline lives in three files in `core/`:

| File | Responsibility |
|------|---------------|
| `state_processor.go` | Block-level loop: iterates over transactions, produces receipts |
| `state_transition.go` | Single-transaction engine: preCheck → buyGas → EVM dispatch → refund → fee payment |
| `evm.go` | Context constructors: builds `BlockContext` and `TxContext` for the EVM |

---

## The Execution Pipeline

A transaction travels through these stages inside `stateTransition.execute()`:

```
execute()
│
├─ preCheck()
│    ├─ verify nonce == state nonce
│    ├─ verify sender is EOA (or EIP-7702 delegation)
│    ├─ verify gasFeeCap ≥ baseFee
│    └─ buyGas()
│         ├─ check sender balance ≥ gasLimit × gasFeeCap + value
│         ├─ gp.SubGas(gasLimit)         
│         └─ state.SubBalance(sender, gasLimit × gasPrice)
│
├─ IntrinsicGas()                      
│    └─ gasRemaining -= intrinsicGas
│
├─ state.Prepare()                      
│
├─ EVM dispatch
│    ├─ if msg.To == nil:  evm.Create()
│    └─ else:              evm.Call()
│
├─ gasRemaining += calcRefund()         
│
├─ returnGas()
│    ├─ state.AddBalance(sender, gasRemaining × gasPrice)
│    └─ gp.AddGas(gasRemaining)          
│
└─ state.AddBalance(coinbase, gasUsed × effectiveTip)
```

Each stage is covered in detail below.

---

## Block-Level Processing: `Process()`

Before individual transactions are executed, the block processor sets up the shared environment. The `StateProcessor.Process()` method orchestrates this:

```go
// core/state_processor.go

func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB, cfg vm.Config) (*ProcessResult, error) {
    var (
        receipts    types.Receipts
        usedGas     = new(uint64)
        header      = block.Header()
        allLogs     []*types.Log
        gp          = new(GasPool).AddGas(block.GasLimit())
    )
    // ...
    context = NewEVMBlockContext(header, p.chain, nil)
    evm := vm.NewEVM(context, tracingStateDB, config, cfg)

    // Pre-execution system calls
    if beaconRoot := block.BeaconRoot(); beaconRoot != nil {
        ProcessBeaconBlockRoot(*beaconRoot, evm)
    }

    // Iterate over and process the individual transactions
    for i, tx := range block.Transactions() {
        msg, err := TransactionToMessage(tx, signer, header.BaseFee)
        // ...
        receipt, err := ApplyTransactionWithEVM(msg, gp, statedb, blockNumber, blockHash, context.Time, tx, usedGas, evm)
        receipts = append(receipts, receipt)
        allLogs = append(allLogs, receipt.Logs...)
    }
    // ...
}
```

Walking through the key steps:

- **GasPool creation** — `new(GasPool).AddGas(block.GasLimit())` creates the block-level gas budget. Every transaction draws from this pool; if the pool runs dry, no more transactions can execute in this block.
- **EVM creation** — A single EVM instance is created for the entire block and reused across all transactions. The `BlockContext` (built from the header) provides block-level data: coinbase, block number, timestamp, base fee, blob base fee.
- **System calls** — Before user transactions, EIP-4788 stores the beacon block root and EIP-2935/7709 stores the parent block hash via system calls to predeployed contracts.
- **Transaction loop** — Each transaction is converted to a `Message`, then executed via `ApplyTransactionWithEVM`.

### TransactionToMessage

The `Message` struct is geth's internal representation of a transaction, stripped of signature data and with the effective gas price already computed:

```go
// core/state_transition.go

type Message struct {
    To                    *common.Address
    From                  common.Address
    Nonce                 uint64
    Value                 *big.Int
    GasLimit              uint64
    GasPrice              *big.Int
    GasFeeCap             *big.Int
    GasTipCap             *big.Int
    Data                  []byte
    AccessList            types.AccessList
    BlobGasFeeCap         *big.Int
    BlobHashes            []common.Hash
    SetCodeAuthorizations []types.SetCodeAuthorization
    SkipNonceChecks       bool
    SkipTransactionChecks bool
}
```

`TransactionToMessage` recovers the sender address from the signature (via `types.Sender(signer, tx)`), and computes the effective gas price as `min(gasTipCap + baseFee, gasFeeCap)`.

### GasPool

The `GasPool` type is remarkably simple — just a `uint64` wrapper:

```go
// core/gaspool.go

type GasPool uint64

func (gp *GasPool) AddGas(amount uint64) *GasPool {
    // ... overflow check ...
    *(*uint64)(gp) += amount
    return gp
}

func (gp *GasPool) SubGas(amount uint64) error {
    if uint64(*gp) < amount {
        return ErrGasLimitReached
    }
    *(*uint64)(gp) -= amount
    return nil
}
```

Each transaction calls `gp.SubGas(msg.GasLimit)` at the start (reserving its full gas limit from the block pool) and `gp.AddGas(remaining)` at the end (returning unused gas). This ensures the block never exceeds its gas limit, even when multiple transactions are in-flight.

---

## The stateTransition Engine

The core of transaction execution is the `stateTransition` struct. `ApplyMessage` creates one and calls `execute()`:

```go
// core/state_transition.go

func ApplyMessage(evm *vm.EVM, msg *Message, gp *GasPool) (*ExecutionResult, error) {
    evm.SetTxContext(NewEVMTxContext(msg))
    return newStateTransition(evm, msg, gp).execute()
}
```

The struct itself holds the transaction context:

```go
// core/state_transition.go

type stateTransition struct {
    gp           *GasPool
    msg          *Message
    gasRemaining uint64
    initialGas   uint64
    state        vm.StateDB
    evm          *vm.EVM
}
```

`gasRemaining` tracks how much gas is left for this transaction. It starts at `msg.GasLimit` (set during `buyGas`), decreases as computation proceeds, and partially recovers during refund.

---

## preCheck()

`preCheck()` validates the transaction against the current state before any gas is spent:

```go
// core/state_transition.go

func (st *stateTransition) preCheck() error {
    msg := st.msg
    if !msg.SkipNonceChecks {
        stNonce := st.state.GetNonce(msg.From)
        if msgNonce := msg.Nonce; stNonce < msgNonce {
            return fmt.Errorf("%w: address %v, tx: %d state: %d", ErrNonceTooHigh, ...)
        } else if stNonce > msgNonce {
            return fmt.Errorf("%w: address %v, tx: %d state: %d", ErrNonceTooLow, ...)
        }
    }
    if !msg.SkipTransactionChecks {
        // Verify the sender is an EOA (or has a delegation)
        code := st.state.GetCode(msg.From)
        _, delegated := types.ParseDelegation(code)
        if len(code) > 0 && !delegated {
            return fmt.Errorf("%w: address %v, len(code): %d", ErrSenderNoEOA, ...)
        }
    }
    // Verify gasFeeCap >= baseFee (post-London)
    // Verify blobGasFeeCap >= blobBaseFee (post-Cancun)
    // Verify blob version hashes, EIP-7702 authorization list
    // ...
    return st.buyGas()
}
```

The checks in order:

1. **Nonce match** — The transaction's nonce must exactly equal the sender's state nonce. Too high means a gap; too low means a replay.
2. **Sender is EOA** — The sender must not have contract code, unless it has an EIP-7702 delegation (which allows EOAs to temporarily delegate to contract code).
3. **Gas fee cap** — Post-London, `gasFeeCap >= baseFee` is required; post-Cancun, blob transactions also need `blobGasFeeCap >= blobBaseFee`.
4. **Blob validation** — Blob transactions must have a `To` address, at least one blob hash, and valid KZG version hashes.

If all checks pass, `preCheck` calls `buyGas()` at the end.

---

### buyGas()

This is the financial commitment step. The sender pays upfront for the maximum gas the transaction could consume:

```go
// core/state_transition.go

func (st *stateTransition) buyGas() error {
    mgval := new(big.Int).SetUint64(st.msg.GasLimit)
    mgval.Mul(mgval, st.msg.GasPrice)
    balanceCheck := new(big.Int).Set(mgval)
    if st.msg.GasFeeCap != nil {
        balanceCheck.SetUint64(st.msg.GasLimit)
        balanceCheck = balanceCheck.Mul(balanceCheck, st.msg.GasFeeCap)
    }
    balanceCheck.Add(balanceCheck, st.msg.Value)
    // ... add blob fee to balanceCheck for Cancun+ ...

    if have, want := st.state.GetBalance(st.msg.From), balanceCheckU256; have.Cmp(want) < 0 {
        return fmt.Errorf("%w: address %v have %v want %v", ErrInsufficientFunds, ...)
    }
    if err := st.gp.SubGas(st.msg.GasLimit); err != nil {
        return err
    }
    st.gasRemaining = st.msg.GasLimit
    st.initialGas = st.msg.GasLimit
    mgvalU256, _ := uint256.FromBig(mgval)
    st.state.SubBalance(st.msg.From, mgvalU256, tracing.BalanceDecreaseGasBuy)
    return nil
}
```

Two different values are computed:

- **`balanceCheck`** is the *worst-case* cost: `gasLimit × gasFeeCap + value + blobFee`. This checks that the sender *could* afford the transaction even at the maximum fee cap. It's a pure validation — nothing is deducted at this amount.
- **`mgval`** is the *actual* deduction: `gasLimit × effectiveGasPrice + blobFee`. This is what's subtracted from the sender's balance. The effective gas price is always `≤ gasFeeCap`, so the actual deduction is always `≤ balanceCheck`.

Then `gp.SubGas(msg.GasLimit)` reserves gas from the block pool, `gasRemaining` is set to the full gas limit, and the ETH is subtracted from the sender.

---

## IntrinsicGas()

After `preCheck` succeeds, `execute()` computes the intrinsic gas — the minimum gas any transaction must pay before the EVM even starts:

```go
// core/state_transition.go

gas, err := IntrinsicGas(msg.Data, msg.AccessList, msg.SetCodeAuthorizations,
    contractCreation, rules.IsHomestead, rules.IsIstanbul, rules.IsShanghai)
```

The `IntrinsicGas` function computes the sum of:

| Component | Cost |
|-----------|------|
| Base cost | 21,000 (simple transfer) or 53,000 (contract creation) |
| Zero bytes in calldata | 4 gas each |
| Non-zero bytes in calldata | 16 gas each (post-Istanbul; was 68 pre-Istanbul) |
| Initcode word charge | 2 gas per 32-byte word (EIP-3860, Shanghai+, creation only) |
| Access list addresses | 2,400 gas each (EIP-2930) |
| Access list storage keys | 1,900 gas each (EIP-2930) |
| EIP-7702 authorizations | 25,000 gas each (`CallNewAccountGas`) |

```go
// core/state_transition.go

func IntrinsicGas(data []byte, accessList types.AccessList, authList []types.SetCodeAuthorization,
    isContractCreation, isHomestead, isEIP2028, isEIP3860 bool) (uint64, error) {
    var gas uint64
    if isContractCreation && isHomestead {
        gas = params.TxGasContractCreation   // 53,000
    } else {
        gas = params.TxGas                   // 21,000
    }
    // Zero and non-zero bytes are priced differently
    z := uint64(bytes.Count(data, []byte{0}))
    nz := uint64(len(data)) - z

    nonZeroGas := params.TxDataNonZeroGasFrontier   // 68
    if isEIP2028 {
        nonZeroGas = params.TxDataNonZeroGasEIP2028 // 16
    }
    gas += nz * nonZeroGas
    gas += z * params.TxDataZeroGas                 // 4

    if accessList != nil {
        gas += uint64(len(accessList)) * params.TxAccessListAddressGas      // 2,400
        gas += uint64(accessList.StorageKeys()) * params.TxAccessListStorageKeyGas // 1,900
    }
    if authList != nil {
        gas += uint64(len(authList)) * params.CallNewAccountGas             // 25,000
    }
    return gas, nil
}
```

If `gasRemaining < intrinsicGas`, the transaction fails immediately with `ErrIntrinsicGas`. Otherwise, intrinsic gas is subtracted from `gasRemaining` and the rest is available for EVM execution.

### Floor Data Gas (EIP-7623)

Post-Prague, there is an additional constraint: the **floor data gas**. This ensures data-heavy transactions (like calldata-based rollup batches) pay a minimum cost proportional to their data size:

```go
// core/state_transition.go

func FloorDataGas(data []byte) (uint64, error) {
    var (
        z      = uint64(bytes.Count(data, []byte{0}))
        nz     = uint64(len(data)) - z
        tokens = nz*params.TxTokenPerNonZeroByte + z
    )
    return params.TxGas + tokens*params.TxCostFloorPerToken, nil
}
```

If the gas actually used by execution is less than the floor data gas, `gasRemaining` is reduced so that the total gas consumed equals the floor. This prevents data-heavy transactions from getting away with paying less than their fair share.

---

## EVM Dispatch

After intrinsic gas is deducted, the access list is prepared and the EVM is dispatched:

```go
// core/state_transition.go (inside execute)

st.state.Prepare(rules, msg.From, st.evm.Context.Coinbase, msg.To, vm.ActivePrecompiles(rules), msg.AccessList)

var (
    ret   []byte
    vmerr error
)
if contractCreation {
    ret, _, st.gasRemaining, vmerr = st.evm.Create(msg.From, msg.Data, st.gasRemaining, value)
} else {
    st.state.SetNonce(msg.From, st.state.GetNonce(msg.From)+1, tracing.NonceChangeEoACall)
    // ... apply EIP-7702 authorizations ...
    ret, st.gasRemaining, vmerr = st.evm.Call(msg.From, st.to(), msg.Data, st.gasRemaining, value)
}
```

Key observations:

- **`Prepare()`** warms the access list — the sender, recipient, coinbase, and all precompile addresses are added to the EIP-2929 access list. Any addresses and storage keys from the transaction's EIP-2930 access list are also added. This means subsequent SLOAD/CALL to these addresses pay the "warm" gas cost instead of the "cold" cost.
- **Contract creation** — If `msg.To == nil`, `evm.Create()` is called. The `Data` field is treated as init code, executed to produce the deployed bytecode.
- **Regular call** — Otherwise, the nonce is incremented first (before the call, not after), then `evm.Call()` executes the contract at the destination address. EIP-7702 authorizations are applied between the nonce increment and the call.
- **`vmerr` vs `err`** — EVM errors (`vmerr`) like `ErrOutOfGas` or `ErrExecutionReverted` are *not* consensus errors. They don't prevent the transaction from being included in the block — the transaction is included, gas is consumed, but the state changes from the reverted call are rolled back. Consensus errors (returned as `err` from `execute()`) mean the transaction is invalid and must not be included.

The EVM internals are covered in Chapter 07 — The EVM Deep Dive.

---

## calcRefund()

After EVM execution, unused gas and refund counters are reconciled:

```go
// core/state_transition.go

func (st *stateTransition) calcRefund() uint64 {
    var refund uint64
    if !st.evm.ChainConfig().IsLondon(st.evm.Context.BlockNumber) {
        refund = st.gasUsed() / params.RefundQuotient        // gasUsed / 2
    } else {
        refund = st.gasUsed() / params.RefundQuotientEIP3529  // gasUsed / 5
    }
    if refund > st.state.GetRefund() {
        refund = st.state.GetRefund()
    }
    return refund
}
```

The refund counter is incremented during EVM execution by operations like SSTORE (clearing a storage slot to zero). But the refund is capped:

- **Pre-London**: capped at `gasUsed / 2`
- **Post-London (EIP-3529)**: capped at `gasUsed / 5`

The cap was tightened in EIP-3529 to prevent "gas token" exploits where users would create and destroy storage slots specifically to harvest refunds.

The computed refund is added back to `gasRemaining`:

```go
// inside execute()
st.gasRemaining += st.calcRefund()
```

---

## returnGas()

After refunds are applied, the remaining gas is converted back to ETH and returned to the sender:

```go
// core/state_transition.go

func (st *stateTransition) returnGas() {
    remaining := uint256.NewInt(st.gasRemaining)
    remaining.Mul(remaining, uint256.MustFromBig(st.msg.GasPrice))
    st.state.AddBalance(st.msg.From, remaining, tracing.BalanceIncreaseGasReturn)

    // Return remaining gas to the block gas counter
    st.gp.AddGas(st.gasRemaining)
}
```

This reverses part of what `buyGas()` did. The sender originally paid `gasLimit × gasPrice`. Now they get back `gasRemaining × gasPrice`. The gas pool is also replenished, making this gas available for subsequent transactions in the block.

---

## Fee Distribution

After returning gas to the sender, the tip is paid to the coinbase (the block producer):

```go
// core/state_transition.go (inside execute)

effectiveTip := msg.GasPrice
if rules.IsLondon {
    effectiveTip = new(big.Int).Sub(msg.GasPrice, st.evm.Context.BaseFee)
}
effectiveTipU256, _ := uint256.FromBig(effectiveTip)

fee := new(uint256.Int).SetUint64(st.gasUsed())
fee.Mul(fee, effectiveTipU256)
st.state.AddBalance(st.evm.Context.Coinbase, fee, tracing.BalanceIncreaseRewardTransactionFee)
```

Post-London (EIP-1559), the gas price splits into two parts:

- **Base fee** (`msg.GasPrice - effectiveTip`): This ETH was already deducted from the sender during `buyGas()`, but it is **not** added to the coinbase. It is effectively burned — removed from circulation.
- **Tip** (`effectiveTip = msg.GasPrice - baseFee`): This goes to the coinbase as the block producer's reward for including the transaction.

The `gasUsed()` helper computes `initialGas - gasRemaining` — the total gas actually consumed (including intrinsic gas but after refunds).

---

## Receipt Creation

After execution completes, `ApplyTransactionWithEVM` calls `MakeReceipt` to build the receipt:

```go
// core/state_processor.go

func MakeReceipt(evm *vm.EVM, result *ExecutionResult, statedb *state.StateDB,
    blockNumber *big.Int, blockHash common.Hash, blockTime uint64,
    tx *types.Transaction, usedGas uint64, root []byte) *types.Receipt {

    receipt := &types.Receipt{Type: tx.Type(), PostState: root, CumulativeGasUsed: usedGas}
    if result.Failed() {
        receipt.Status = types.ReceiptStatusFailed
    } else {
        receipt.Status = types.ReceiptStatusSuccessful
    }
    receipt.TxHash = tx.Hash()
    receipt.GasUsed = result.UsedGas

    if tx.Type() == types.BlobTxType {
        receipt.BlobGasUsed = uint64(len(tx.BlobHashes()) * params.BlobTxBlobGasPerBlob)
        receipt.BlobGasPrice = evm.Context.BlobBaseFee
    }

    if tx.To() == nil {
        receipt.ContractAddress = crypto.CreateAddress(evm.TxContext.Origin, tx.Nonce())
    }

    receipt.Logs = statedb.GetLogs(tx.Hash(), blockNumber.Uint64(), blockHash, blockTime)
    receipt.Bloom = types.CreateBloom(receipt)
    return receipt
}
```

Key fields in the receipt:

- **Status**: 1 (success) or 0 (failure). Post-Byzantium, this replaced the post-state root.
- **CumulativeGasUsed**: Running total of gas used by all transactions in the block up to and including this one.
- **GasUsed**: Gas consumed by this individual transaction.
- **ContractAddress**: If the transaction created a contract, the new contract's address (derived from sender + nonce).
- **Logs**: Events emitted during EVM execution.
- **Bloom**: A 2048-bit bloom filter over the logs, used for efficient log filtering.

Before receipt creation, `ApplyTransactionWithEVM` also calls `statedb.Finalise(true)` (post-Byzantium) to promote dirty state into the pending trie, preparing the state for the next transaction.

---

## The ExecutionResult

The `execute()` method returns an `ExecutionResult` that captures the outcome:

```go
// core/state_transition.go

type ExecutionResult struct {
    UsedGas    uint64 // Total gas used (after refunds)
    MaxUsedGas uint64 // Peak gas consumed during execution (before refunds)
    Err        error  // EVM error (nil = success, non-nil = reverted/OOG/etc.)
    ReturnData []byte // Data returned by the EVM
}

func (result *ExecutionResult) Failed() bool { return result.Err != nil }
```

`Failed()` returns true for any EVM error. Even when `Failed()` is true, the transaction is included in the block — gas is consumed, the nonce is incremented, but the EVM's state changes are reverted. The receipt's status will be 0 (failed).

---

## The EVM Context

The EVM needs two kinds of context — block-level and transaction-level. Both are constructed in `core/evm.go`:

```go
// core/evm.go

func NewEVMBlockContext(header *types.Header, chain ChainContext, author *common.Address) vm.BlockContext {
    // ...
    return vm.BlockContext{
        CanTransfer: CanTransfer,
        Transfer:    Transfer,
        GetHash:     GetHashFn(header, chain),
        Coinbase:    beneficiary,
        BlockNumber: new(big.Int).Set(header.Number),
        Time:        header.Time,
        Difficulty:  new(big.Int).Set(header.Difficulty),
        BaseFee:     baseFee,
        BlobBaseFee: blobBaseFee,
        GasLimit:    header.GasLimit,
        Random:      random,
    }
}
```

- **`CanTransfer`** and **`Transfer`** are function values that check balances and move ETH. By injecting them as functions, the EVM doesn't need to know about `StateDB` specifics.
- **`GetHash`** returns a closure that looks up ancestor block hashes by number (needed for the `BLOCKHASH` opcode). It uses a cache that lazily fills from the chain.
- **`Random`** is the post-Merge RANDAO mix (the `MixDigest` header field when `Difficulty == 0`), exposed to the EVM via the `PREVRANDAO` opcode.

The transaction context is simpler:

```go
// core/evm.go

func NewEVMTxContext(msg *Message) vm.TxContext {
    ctx := vm.TxContext{
        Origin:     msg.From,
        GasPrice:   new(big.Int).Set(msg.GasPrice),
        BlobHashes: msg.BlobHashes,
    }
    if msg.BlobGasFeeCap != nil {
        ctx.BlobFeeCap = new(big.Int).Set(msg.BlobGasFeeCap)
    }
    return ctx
}
```

`Origin` is the transaction sender (used by the `ORIGIN` opcode) and `GasPrice` is the effective gas price (used by `GASPRICE`).

---

## What's Next

This chapter traced the execution pipeline from `Process()` down to the EVM dispatch and back. The EVM itself — the bytecode interpreter, opcode dispatch, gas metering, memory/stack management, and nested calls — is covered in Chapter 07 — The EVM Deep Dive.
