---
title: Geth(8) The Transaction Pool
published: 2026-04-17
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

Before a transaction can appear in a block, it must wait in the **transaction pool** (txpool). The pool is the holding area where geth receives transactions from the network and local RPC submissions, validates them, organises them by sender and nonce, and serves them to the block builder when it is time to produce a new block.

This chapter covers the pool's architecture: the coordinator that ties everything together, the shared validation pipeline, the `LegacyPool` for normal transactions, and the `BlobPool` for EIP-4844 blob transactions.

---

## How a Transaction Enters and Exits the Pool

```
                    Network / RPC
                         |
                         v
                   TxPool.Add()                 ── coordinator
                         |
                         | Split by tx type (Filter)
                         |
              +----------+----------+
              |                     |
              v                     v
       LegacyPool.Add()      BlobPool.Add()    ── subpools
              |                     |
              v                     v
       1. ValidateTxBasics    1. ValidateTxBasics
          (stateless)            (stateless)
       2. validateTx          2. validate
          (stateful)             (stateful)
       3. add() →             3. store to disk
          queue or pending
              |                     |
              +------ waiting ------+
                         |
                    ChainHeadEvent
                    triggers Reset()
                         |
              +----------+----------+
              |                     |
              v                     v
        LegacyPool:            BlobPool:
        promoteExecutables()   recheck accounts,
        demoteUnexecutables()  drop finalized txs
              |                     |
              v                     v
         Pending map           Pending map
              |                     |
              +----------+----------+
                         |
                         v
                    TxPool.Pending()
                         |
                    miner pulls txs
                    for block building
```

A transaction's life in the pool follows these stages:

1. **Arrival** — `TxPool.Add()` receives a batch, routes each transaction to the subpool whose `Filter()` matches its type.
2. **Stateless validation** — signature recovery, size limits, fork rules, intrinsic gas, fee sanity (shared `ValidateTransaction()`).
3. **Stateful validation** — nonce ordering, balance sufficiency, overdraft checks (shared `ValidateTransactionWithState()`).
4. **Insertion** — the transaction enters the subpool. In `LegacyPool` it typically goes into the **queue** (future transactions with nonce gaps) or directly into **pending** (executable transactions).
5. **Promotion** — when a new chain head arrives, `promoteExecutables()` moves queued transactions that are now executable into the pending set.
6. **Demotion** — `demoteUnexecutables()` moves pending transactions that are no longer valid (nonce already used, insufficient balance) back to the queue or drops them entirely.
7. **Consumption** — the miner calls `Pending()` to pull executable transactions for block building. Once included in a block, the next `Reset()` removes them.
8. **Eviction** — if the pool exceeds capacity, underpriced or expired transactions are dropped.

---

## The TxPool Coordinator

The top-level `TxPool` struct in `core/txpool/txpool.go` does not store transactions itself. It is a **coordinator** that delegates to specialised subpools while presenting a unified API to the rest of geth.

```go
// core/txpool/txpool.go

type TxPool struct {
    subpools []SubPool           // List of subpools for specialized transaction handling
    chain    BlockChain

    stateLock sync.RWMutex       // The lock for protecting state instance
    state     *state.StateDB     // Current state at the blockchain head

    subs event.SubscriptionScope // Subscription scope to unsubscribe all on shutdown
    quit chan chan error          // Quit channel to tear down the head updater
    term chan struct{}            // Termination channel to detect a closed pool

    sync chan chan error          // Testing / simulator channel to block until internal reset is done
}
```

- **`subpools`** — the registered subpool implementations. In production geth this is `[LegacyPool, BlobPool]`.
- **`state`** — a snapshot of the blockchain head state, used by `Nonce()` to return the on-chain nonce (without pending pool transactions applied).
- **`quit` / `term`** — shutdown coordination. The `term` channel is closed when `loop()` exits, letting callers detect a stopped pool.
- **`sync`** — used only in tests and simulator mode to force a deterministic reset cycle.

### Initialisation

`New()` creates the pool, initialises every subpool, and starts the event loop:

```go
// core/txpool/txpool.go

func New(gasTip uint64, chain BlockChain, subpools []SubPool) (*TxPool, error) {
    head := chain.CurrentBlock()

    statedb, err := chain.StateAt(head.Root)
    if err != nil {
        statedb, err = chain.StateAt(types.EmptyRootHash)
    }
    if err != nil {
        return nil, err
    }
    pool := &TxPool{
        subpools: subpools,
        chain:    chain,
        state:    statedb,
        quit:     make(chan chan error),
        term:     make(chan struct{}),
        sync:     make(chan chan error),
    }
    reserver := NewReservationTracker()
    for i, subpool := range subpools {
        if err := subpool.Init(gasTip, head, reserver.NewHandle(i)); err != nil {
            for j := i - 1; j >= 0; j-- {
                subpools[j].Close()
            }
            return nil, err
        }
    }
    go pool.loop(head)
    return pool, nil
}
```

Key points:

- The current block head is captured **once** so all subpools start from the same state, even if the chain advances during initialisation.
- A `ReservationTracker` is created and each subpool gets its own `Reserver` handle. This ensures that a given sender address is tracked by **exactly one** subpool at a time — if `LegacyPool` holds an account, `BlobPool` cannot accept transactions from that same sender.
- If any subpool fails to initialise, previously initialised subpools are closed in reverse order.

### The Event Loop

The coordinator's `loop()` goroutine listens for `ChainHeadEvent` notifications and triggers resets on all subpools when the chain head changes:

```go
// core/txpool/txpool.go  (simplified)

func (p *TxPool) loop(head *types.Header) {
    defer close(p.term)

    newHeadCh  := make(chan core.ChainHeadEvent)
    newHeadSub := p.chain.SubscribeChainHeadEvent(newHeadCh)
    defer newHeadSub.Unsubscribe()

    var (
        oldHead = head
        newHead = oldHead
    )
    resetBusy := make(chan struct{}, 1)
    resetDone := make(chan *types.Header)
    // ...

    for errc == nil {
        if newHead != oldHead || resetForced {
            select {
            case resetBusy <- struct{}{}:
                // Update the coordinator's state snapshot
                if statedb, err := p.chain.StateAt(newHead.Root); err == nil {
                    p.stateLock.Lock()
                    p.state = statedb
                    p.stateLock.Unlock()
                }
                // Reset all subpools in a background goroutine
                go func(oldHead, newHead *types.Header) {
                    for _, subpool := range p.subpools {
                        subpool.Reset(oldHead, newHead)
                    }
                    resetDone <- newHead
                }(oldHead, newHead)
            default:
                // A reset is already running; will retry on next iteration
            }
        }
        select {
        case event := <-newHeadCh:
            newHead = event.Header
        case head := <-resetDone:
            oldHead = head
            <-resetBusy
        case errc = <-p.quit:
            // break out on next iteration
        }
    }
}
```

The pattern here is worth noting:

- **At most one reset runs concurrently.** The `resetBusy` channel (capacity 1) acts as a semaphore. If a reset is already in progress, new chain head events are simply recorded — `newHead` is updated — and the next reset will process them.
- The coordinator updates its own `state` snapshot before kicking off subpool resets, so `Nonce()` calls reflect the latest head immediately.

### Routing Transactions to Subpools

When `Add()` receives a batch of transactions, it splits them across subpools using each subpool's `Filter()` method:

```go
// core/txpool/txpool.go

func (p *TxPool) Add(txs []*types.Transaction, sync bool) []error {
    txsets := make([][]*types.Transaction, len(p.subpools))
    splits := make([]int, len(txs))

    for i, tx := range txs {
        splits[i] = -1
        for j, subpool := range p.subpools {
            if subpool.Filter(tx) {
                txsets[j] = append(txsets[j], tx)
                splits[i] = j
                break
            }
        }
    }
    // Add split batches to each subpool
    errsets := make([][]error, len(p.subpools))
    for i := 0; i < len(p.subpools); i++ {
        errsets[i] = p.subpools[i].Add(txsets[i], sync)
    }
    // Reassemble errors in original order
    errs := make([]error, len(txs))
    for i, split := range splits {
        if split == -1 {
            errs[i] = fmt.Errorf("%w: received type %d", core.ErrTxTypeNotSupported, txs[i].Type())
            continue
        }
        errs[i] = errsets[split][0]
        errsets[split] = errsets[split][1:]
    }
    return errs
}
```

- Each transaction is routed to the **first** subpool whose `Filter()` returns true. `LegacyPool.Filter()` accepts types 0 (Legacy), 1 (AccessList), 2 (DynamicFee), and 4 (SetCode). `BlobPool.Filter()` accepts only type 3 (BlobTx).
- The `splits` array tracks which subpool each transaction went to, allowing the coordinator to stitch the per-subpool error slices back into the original batch order.
- If no subpool accepts a transaction, it gets `ErrTxTypeNotSupported`.

### Merging Pending Transactions

The miner calls `Pending()` to get all executable transactions. The coordinator merges results from every subpool:

```go
// core/txpool/txpool.go

func (p *TxPool) Pending(filter PendingFilter) map[common.Address][]*LazyTransaction {
    txs := make(map[common.Address][]*LazyTransaction)
    for _, subpool := range p.subpools {
        for addr, set := range subpool.Pending(filter) {
            txs[addr] = set
        }
    }
    return txs
}
```

Because the `Reserver` mechanism guarantees each address belongs to at most one subpool, the merge is a simple union — there are no conflicting entries for the same address.

The `LazyTransaction` wrapper (defined in `core/txpool/subpool.go`) carries just the metadata the miner needs for ordering — `GasFeeCap`, `GasTipCap`, `Gas`, `BlobGas` — without materialising the full transaction object. The full `*types.Transaction` is resolved on demand via `Resolve()`.

---

## The SubPool Interface

Every transaction subpool must implement the `SubPool` interface defined in `core/txpool/subpool.go`:

```go
// core/txpool/subpool.go

type SubPool interface {
    Filter(tx *types.Transaction) bool
    Init(gasTip uint64, head *types.Header, reserver Reserver) error
    Close() error
    Reset(oldHead, newHead *types.Header)
    SetGasTip(tip *big.Int)
    Has(hash common.Hash) bool
    Get(hash common.Hash) *types.Transaction
    GetRLP(hash common.Hash) []byte
    GetMetadata(hash common.Hash) *TxMetadata
    ValidateTxBasics(tx *types.Transaction) error
    Add(txs []*types.Transaction, sync bool) []error
    Pending(filter PendingFilter) map[common.Address][]*LazyTransaction
    SubscribeTransactions(ch chan<- core.NewTxsEvent, reorgs bool) event.Subscription
    Nonce(addr common.Address) uint64
    Stats() (int, int)
    Content() (map[common.Address][]*types.Transaction, map[common.Address][]*types.Transaction)
    ContentFrom(addr common.Address) ([]*types.Transaction, []*types.Transaction)
    Status(hash common.Hash) TxStatus
    Clear()
}
```

The interface has two distinct groups of methods:

- **Lifecycle methods** — `Filter`, `Init`, `Close`, `Reset`, `SetGasTip`. These are called by the coordinator to manage the subpool's lifecycle and keep it in sync with the chain.
- **Data methods** — `Has`, `Get`, `Add`, `Pending`, `Nonce`, `Stats`, `Content`, `Status`. These serve external queries and transaction submissions.

### Account Reservation

The `Reserver` interface prevents two subpools from tracking the same sender address simultaneously:

```go
// core/txpool/reserver.go

type Reserver interface {
    Hold(addr common.Address) error
    Release(addr common.Address) error
    Has(address common.Address) bool
}
```

When a subpool receives a transaction from a new sender, it calls `Hold(addr)`. If another subpool already holds that address, `Hold` returns an error and the transaction is rejected. When all transactions from an account are evicted, the subpool calls `Release(addr)`.

This is essential for correctness: if the same sender had transactions in both `LegacyPool` and `BlobPool`, nonce tracking and balance accounting would be inconsistent.

---

## Transaction Validation

Validation happens in two phases, both implemented as shared functions in `core/txpool/validation.go` so that all subpools apply identical rules.

### Phase 1: Stateless Validation

`ValidateTransaction()` checks everything that does not require reading the blockchain state:

```go
// core/txpool/validation.go

func ValidateTransaction(tx *types.Transaction, head *types.Header,
    signer types.Signer, opts *ValidationOptions) error {
    // 1. Transaction type accepted by this pool?
    if opts.Accept&(1<<tx.Type()) == 0 {
        return fmt.Errorf("%w: tx type %v not supported", core.ErrTxTypeNotSupported, tx.Type())
    }
    // 2. Blob count within limit?
    if blobCount := len(tx.BlobHashes()); blobCount > opts.MaxBlobCount { ... }

    // 3. Transaction size within limit?
    if tx.Size() > opts.MaxSize { ... }

    // 4. Fork-specific type checks
    if !rules.IsBerlin  && tx.Type() != types.LegacyTxType { ... }
    if !rules.IsLondon  && tx.Type() == types.DynamicFeeTxType { ... }
    if !rules.IsCancun  && tx.Type() == types.BlobTxType { ... }
    if !rules.IsPrague  && tx.Type() == types.SetCodeTxType { ... }

    // 5. Init code size limit (Shanghai)
    if rules.IsShanghai && tx.To() == nil && len(tx.Data()) > params.MaxInitCodeSize { ... }

    // 6. Max transaction gas limit (Osaka)
    if rules.IsOsaka && tx.Gas() > params.MaxTxGas { ... }

    // 7. No negative value
    if tx.Value().Sign() < 0 { ... }

    // 8. Gas within block limit
    if head.GasLimit < tx.Gas() { ... }

    // 9. Fee caps are sane (not astronomically large)
    if tx.GasFeeCap().BitLen() > 256 { ... }
    if tx.GasTipCap().BitLen() > 256 { ... }

    // 10. Fee cap >= tip cap
    if tx.GasFeeCapIntCmp(tx.GasTipCap()) < 0 { ... }

    // 11. Signature valid, sender recoverable
    if _, err := types.Sender(signer, tx); err != nil { ... }

    // 12. Nonce not at max (EIP-2681)
    if tx.Nonce()+1 < tx.Nonce() { ... }

    // 13. Enough gas to cover intrinsic cost
    intrGas, _ := core.IntrinsicGas(tx.Data(), tx.AccessList(),
        tx.SetCodeAuthorizations(), tx.To() == nil, true,
        rules.IsIstanbul, rules.IsShanghai)
    if tx.Gas() < intrGas { ... }

    // 14. Floor data gas (Prague)
    // 15. Minimum tip for this pool
    // 16. Blob-specific checks (if blob tx)
    // ...
}
```

The `ValidationOptions` struct controls per-pool differences:

| Field | Purpose |
|-------|---------|
| `Accept` | Bitmask of accepted transaction types (e.g., `LegacyPool` sets bits 0,1,2,4) |
| `MaxSize` | Maximum serialised transaction size (`LegacyPool`: 4 * 32KB = 128KB) |
| `MaxBlobCount` | Maximum blobs per transaction (0 for `LegacyPool`) |
| `MinTip` | Minimum gas tip to enter this pool |

For blob transactions, additional checks run in `validateBlobTx()`: the sidecar must be present, there must be at least one blob, the blob count must not exceed `BlobTxMaxBlobs`, the blob fee cap must meet the protocol minimum, blob/commitment/proof counts must match, and KZG proofs must verify. The proof verification is dispatched based on the sidecar's `Version` field: version 0 (legacy) uses per-blob KZG proofs, while version 1 (Osaka) uses cell proofs (EIP-7594).

### Phase 2: Stateful Validation

`ValidateTransactionWithState()` checks properties that require the current state:

```go
// core/txpool/validation.go

func ValidateTransactionWithState(tx *types.Transaction, signer types.Signer,
    opts *ValidationOptionsWithState) error {
    from, _ := types.Sender(signer, tx)

    // 1. Nonce must not be stale
    next := opts.State.GetNonce(from)
    if next > tx.Nonce() {
        return fmt.Errorf("%w: next nonce %v, tx nonce %v", core.ErrNonceTooLow, next, tx.Nonce())
    }
    // 2. No nonce gap (if pool enforces ordering)
    if opts.FirstNonceGap != nil {
        if gap := opts.FirstNonceGap(from); gap < tx.Nonce() {
            return fmt.Errorf("%w: tx nonce %v, gapped nonce %v", core.ErrNonceTooHigh, ...)
        }
    }
    // 3. Balance covers this transaction's cost
    balance := opts.State.GetBalance(from).ToBig()
    cost    := tx.Cost()
    if balance.Cmp(cost) < 0 { ... }

    // 4. Balance covers all queued transactions + this one (overdraft check)
    spent := opts.ExistingExpenditure(from)
    if prev := opts.ExistingCost(from, tx.Nonce()); prev != nil {
        // Replacement: check balance covers (total_spent + bump)
        bump := new(big.Int).Sub(cost, prev)
        need := new(big.Int).Add(spent, bump)
        if balance.Cmp(need) < 0 { ... }
    } else {
        // New nonce: check balance covers (total_spent + this_cost)
        need := new(big.Int).Add(spent, cost)
        if balance.Cmp(need) < 0 { ... }

        // Also check account slot limits
        if opts.UsedAndLeftSlots != nil {
            if used, left := opts.UsedAndLeftSlots(from); left <= 0 { ... }
        }
    }
    return nil
}
```

The `ValidationOptionsWithState` callbacks let each subpool plug in its own accounting. For example, `LegacyPool` provides `ExistingExpenditure` that sums the `totalcost` of the pending list, and `ExistingCost` that looks up a specific nonce's transaction cost. Notably, `LegacyPool` sets `FirstNonceGap` to `nil` — it deliberately allows nonce gaps in the queue, only enforcing continuity in the pending set.

---

## The LegacyPool

The `LegacyPool` in `core/txpool/legacypool/legacypool.go` handles all non-blob transaction types: Legacy (type 0), AccessList (type 1), DynamicFee (type 2), and SetCode (type 4). It is the workhorse of geth's transaction management.

### Configuration

The pool's behaviour is governed by `Config` with these defaults:

```go
// core/txpool/legacypool/legacypool.go

var DefaultConfig = Config{
    PriceLimit: 1,          // 1 Wei minimum gas tip
    PriceBump:  10,         // 10% price bump required for replacement
    AccountSlots: 16,       // Max pending txs per account
    GlobalSlots:  5120,     // Max pending txs across all accounts
    AccountQueue: 64,       // Max queued txs per account
    GlobalQueue:  1024,     // Max queued txs across all accounts
    Lifetime: 3 * time.Hour, // Max time a queued tx can survive
}
```

Two size constants control individual transactions:

- **`txSlotSize`** = 32 KB — the unit of measurement for pool capacity. A transaction's "slot count" is `ceil(size / 32KB)`.
- **`txMaxSize`** = 4 * `txSlotSize` = 128 KB — the absolute maximum transaction size.

### The Pending and Queue Maps

The pool maintains two core data structures:

```go
// core/txpool/legacypool/legacypool.go  (key fields)

type LegacyPool struct {
    pending      map[common.Address]*list  // Executable transactions (next nonce matches state)
    queue        *queue                    // Future transactions (nonce gaps exist)
    all          *lookup                   // Hash → transaction lookup for deduplication
    priced       *pricedList               // Price-sorted heap for eviction decisions
    pendingNonces *noncer                  // Virtual nonces tracking pending txs
    // ...
}
```

**Pending** (`map[common.Address]*list`) holds transactions that are immediately executable — their nonces form a contiguous sequence starting from the account's current on-chain nonce. Each `list` is a nonce-sorted structure backed by a `SortedMap` (a `map[uint64]*Transaction` with a heap-based nonce index). Pending lists use **strict mode** (`strict: true`): removing a transaction at nonce N also invalidates all transactions at nonces > N, since they are no longer contiguous.

**Queue** (`*queue`, wrapping `map[common.Address]*list`) holds future transactions — those with nonce gaps. Queue lists use **non-strict mode** (`strict: false`): removing a transaction does not cascade. The queue also tracks a `beats` map of last-activity timestamps per account, used for eviction of stale entries.

**`all`** (`*lookup`) is a flat `map[common.Hash]*Transaction` for O(1) deduplication. It also tracks the total slot count used across both pending and queue.

**`priced`** (`*pricedList`) maintains two price-sorted heaps — an **urgent** heap (sorted by effective tip at current base fee) and a **floating** heap (sorted by fee cap). When the pool overflows, transactions are evicted from the cheaper heap. The two-heap design handles both congested periods (where effective tip matters most) and base fee peaks (where fee cap is the binding constraint).

### The list and SortedMap Internals

Each per-account `list` wraps a `SortedMap`:

```go
// core/txpool/legacypool/list.go

type SortedMap struct {
    items   map[uint64]*types.Transaction // nonce → transaction
    index   *nonceHeap                    // min-heap of nonces
    cache   types.Transactions            // cached nonce-sorted slice
    cacheMu sync.Mutex
}

type list struct {
    strict    bool         // true for pending (contiguous nonces), false for queue
    txs       *SortedMap
    costcap   *uint256.Int // highest single-tx cost seen
    gascap    uint64       // highest single-tx gas limit seen
    totalcost *uint256.Int // sum of Cost() for all txs in the list
}
```

The `SortedMap` provides the key operations:

- **`Put(tx)`** — insert or replace a transaction by nonce. Invalidates the sorted cache.
- **`Forward(threshold)`** — remove all transactions with nonce < threshold. Used during `demoteUnexecutables()` to strip confirmed transactions.
- **`Ready(start)`** — extract a contiguous run of transactions starting at nonce `start`. Used during `promoteExecutables()` to pull queue entries into pending.
- **`Filter(fn)`** — remove all transactions matching a predicate. Used to drop transactions that exceed the account's balance.
- **`Cap(threshold)`** — keep only `threshold` transactions, dropping the highest-nonce ones. Used during `truncatePending()`.

### Transaction Replacement

When a transaction arrives with a nonce that already exists in the pool, it is a **replacement** attempt. `list.Add()` enforces a minimum price bump:

```go
// core/txpool/legacypool/list.go

func (l *list) Add(tx *types.Transaction, priceBump uint64) (bool, *types.Transaction) {
    old := l.txs.Get(tx.Nonce())
    if old != nil {
        if old.GasFeeCapCmp(tx) >= 0 || old.GasTipCapCmp(tx) >= 0 {
            return false, nil
        }
        // Both fee cap and tip must exceed old * (100 + priceBump) / 100
        a := big.NewInt(100 + int64(priceBump))
        aFeeCap := new(big.Int).Mul(a, old.GasFeeCap())
        aTip := a.Mul(a, old.GasTipCap())

        b := big.NewInt(100)
        thresholdFeeCap := aFeeCap.Div(aFeeCap, b)
        thresholdTip := aTip.Div(aTip, b)

        if tx.GasFeeCapIntCmp(thresholdFeeCap) < 0 ||
           tx.GasTipCapIntCmp(thresholdTip) < 0 {
            return false, nil
        }
    }
    // Accept: update totalcost, insert into SortedMap
    // ...
}
```

With the default `PriceBump` of 10, both the fee cap and tip cap must be at least 10% higher than the existing transaction. This prevents spam replacements while allowing legitimate fee bumps.

### The add() Pipeline

When a new transaction passes validation, `add()` handles insertion:

```go
// core/txpool/legacypool/legacypool.go  (simplified)

func (pool *LegacyPool) add(tx *types.Transaction) (replaced bool, err error) {
    hash := tx.Hash()

    // 1. Reject if already known
    if pool.all.Get(hash) != nil {
        return false, txpool.ErrAlreadyKnown
    }
    // 2. Stateful validation (nonce, balance, overdraft)
    if err := pool.validateTx(tx); err != nil {
        return false, err
    }
    from, _ := types.Sender(pool.signer, tx)

    // 3. Reserve the account if new to this subpool
    if !hasPending && !hasQueued {
        if err := pool.reserver.Hold(from); err != nil {
            return false, err
        }
    }
    // 4. If pool is full, evict underpriced transactions
    if pool.all.Slots()+numSlots(tx) > GlobalSlots+GlobalQueue {
        if pool.priced.Underpriced(tx) {
            return false, txpool.ErrUnderpriced
        }
        drop, success := pool.priced.Discard(overflow)
        if !success {
            return false, ErrTxPoolOverflow
        }
        for _, tx := range drop {
            pool.removeTx(tx.Hash(), false, ...)
        }
    }
    // 5. If replacing an existing pending tx, do it directly
    if list := pool.pending[from]; list != nil && list.Contains(tx.Nonce()) {
        inserted, old := list.Add(tx, pool.config.PriceBump)
        if !inserted {
            return false, txpool.ErrReplaceUnderpriced
        }
        // ...
        return old != nil, nil
    }
    // 6. Otherwise, enqueue for later promotion
    pool.enqueueTx(hash, tx, true)
    return false, nil
}
```

Step 4 is the eviction mechanism. When the combined size of pending + queue exceeds `GlobalSlots + GlobalQueue` (6144 transactions by default), the pool must make room. It checks whether the new transaction is cheaper than the cheapest existing one (`Underpriced`). If not, it calls `Discard()` on the `pricedList` to evict the cheapest remote transactions.

Step 5 is a fast path: if the sender already has a pending transaction at this nonce, we attempt a direct replacement without going through the queue. This is the common "speed up" or "cancel" flow where a user resubmits with a higher gas price.

Step 6 is the normal path: the transaction goes into the queue and will be promoted during the next reorg cycle.

### The Reorg Cycle: promote and demote

The `LegacyPool` runs a background `scheduleReorgLoop()` goroutine that batches and processes two types of requests:

1. **Reset requests** — triggered by `ChainHeadEvent` via the coordinator. These re-sync the pool's state to a new chain head.
2. **Promote requests** — triggered after `Add()` to check if newly added transactions are immediately executable.

Both are processed in `runReorg()`, which runs with the pool lock held:

```go
// core/txpool/legacypool/legacypool.go  (simplified)

func (pool *LegacyPool) runReorg(done chan struct{}, reset *txpoolResetRequest,
    dirtyAccounts *accountSet, events map[common.Address]*SortedMap) {
    // ...
    pool.mu.Lock()

    if reset != nil {
        pool.reset(reset.oldHead, reset.newHead)
        promoteAddrs = pool.queue.addresses() // promote all accounts after reset
    }
    // Move newly-executable txs from queue → pending
    promoted := pool.promoteExecutables(promoteAddrs)

    if reset != nil {
        // Remove txs that are no longer valid at the new head
        pool.demoteUnexecutables()

        // Update base fee for price sorting
        pendingBaseFee := eip1559.CalcBaseFee(pool.chainconfig, reset.newHead)
        pool.priced.SetBaseFee(pendingBaseFee)
    }
    // Enforce capacity limits
    pool.truncatePending()
    pool.truncateQueue()

    pool.mu.Unlock()

    // Broadcast new transaction events
    pool.txFeed.Send(core.NewTxsEvent{Txs: txs})
}
```

#### reset()

When the chain head changes, `reset()` handles reorg recovery:

```go
// core/txpool/legacypool/legacypool.go  (simplified)

func (pool *LegacyPool) reset(oldHead, newHead *types.Header) {
    // If a reorg occurred (oldHead is not parent of newHead):
    //   Walk back both chains to the common ancestor
    //   Collect transactions from discarded blocks
    //   Subtract transactions from newly-included blocks
    //   Reinject the difference back into the pool

    // Update state to new head
    statedb, _ := pool.chain.StateAt(newHead.Root)
    pool.currentHead.Store(newHead)
    pool.currentState = statedb
    pool.pendingNonces = newNoncer(statedb)
}
```

The reorg detection walks both chains backwards: the old chain's transactions are collected as "discarded", the new chain's as "included". The set difference (`discarded - included`) represents transactions that were in blocks on the old fork but not the new one — these are reinjected into the pool so they can be re-mined.

Reorgs deeper than 64 blocks are skipped to avoid excessive memory usage during fast sync.

#### promoteExecutables()

This method moves transactions from the queue to the pending set:

For each account in the promotion set, it calls `queue.promoteExecutables()` which:

1. **Drops stale transactions** — those with nonces below the current state nonce (already included in the chain).
2. **Drops expensive transactions** — those whose cost exceeds the account's balance or whose gas exceeds the block gas limit.
3. **Extracts a contiguous nonce run** — using `Ready(pendingNonce)`, which pulls all queue entries starting from the current pending nonce into a contiguous batch.
4. **Caps per-account queue size** — drops the highest-nonce transactions if the account exceeds `AccountQueue` (64).

Each extracted transaction is then passed to `promoteTx()`, which inserts it into the pending list using the same price-bump replacement logic as `list.Add()`.

#### demoteUnexecutables()

After a reset, some pending transactions may no longer be valid:

```go
// core/txpool/legacypool/legacypool.go  (simplified)

func (pool *LegacyPool) demoteUnexecutables() {
    gasLimit := pool.currentHead.Load().GasLimit
    for addr, list := range pool.pending {
        nonce := pool.currentState.GetNonce(addr)

        // Drop confirmed transactions (nonce < state nonce)
        olds := list.Forward(nonce)

        // Drop transactions that exceed balance or gas limit
        drops, invalids := list.Filter(pool.currentState.GetBalance(addr), gasLimit)

        // Move invalidated txs back to queue
        for _, tx := range invalids {
            pool.enqueueTx(tx.Hash(), tx, false)
        }
        // If a nonce gap appeared, demote everything
        if list.Len() > 0 && list.txs.Get(nonce) == nil {
            gapped := list.Cap(0)
            for _, tx := range gapped {
                pool.enqueueTx(tx.Hash(), tx, false)
            }
        }
        if list.Empty() {
            delete(pool.pending, addr)
            if _, ok := pool.queue.get(addr); !ok {
                pool.reserver.Release(addr)
            }
        }
    }
}
```

The `Filter()` call on a strict-mode list is important: if a transaction at nonce N is dropped because it exceeds the balance, all transactions at nonces > N are also invalidated (since they depend on N executing first). These invalidated transactions are moved back to the queue rather than dropped, giving them a chance to become executable again if the account's balance increases.

### Capacity Management

After every reorg, two truncation functions enforce global limits:

**`truncatePending()`** enforces `GlobalSlots` (5120). It identifies "spammer" accounts — those with more than `AccountSlots` (16) pending transactions — and iteratively removes the highest-nonce transaction from the account with the most pending transactions, equalising counts across all spammers. This fair-share approach prevents a single account from monopolising the pending set.

**`truncateQueue()`** enforces `GlobalQueue` (1024). The queue's internal eviction is based on the `beats` timestamp: accounts are sorted by last-activity time and the oldest (least recently active) accounts' transactions are dropped first until the global limit is satisfied. Per-account queue limits (`AccountQueue` = 64) are enforced separately, during `promoteExecutables()`.

Additionally, the `loop()` goroutine runs a periodic eviction ticker (every minute) that removes queued transactions that have been sitting for longer than `Lifetime` (3 hours).

### EIP-7702 SetCode Transaction Restrictions

The `LegacyPool` enforces special rules for accounts involved in EIP-7702 delegation (see Chapter 02):

- **Delegated accounts** (those whose code hash indicates a delegation designator, or those with a pending authorization) are limited to **at most one in-flight executable transaction**. This prevents stacking multiple transactions on a delegated account, since the delegation could be revoked at any time.
- **Authority accounts** named in a SetCode transaction's authorization list are checked against the `Reserver` — if the authority is already tracked by another subpool, the transaction is rejected.

---

## The BlobPool

The `BlobPool` in `core/txpool/blobpool/blobpool.go` is a dedicated subpool for EIP-4844 blob transactions. Blob transactions are fundamentally different from normal transactions: they carry large data blobs (each ~128 KB) intended for rollup data availability, and have a separate fee market (blob gas).

### Why a Separate Pool?

The `BlobPool` exists because blob transactions have properties that conflict with `LegacyPool`'s assumptions:

1. **Size** — a single blob transaction with 6 blobs can be ~768 KB. Keeping thousands of these in memory is impractical. The `BlobPool` stores transaction data **on disk** using a persistent key-value store (`billy.Database`), keeping only lightweight `blobTxMeta` structs in memory.

2. **Low churn** — block blob-space is limited (a few blob transactions per block). The `BlobPool` exploits this by persisting transactions to disk immediately, solving the "lost transactions on restart" problem that plagues `LegacyPool`.

3. **No nonce gaps** — blob transactions are meant for rollups that submit sequentially. The `BlobPool` disallows nonce gaps entirely, unlike `LegacyPool`'s queue.

4. **Aggressive replacement pricing** — since propagating replacement blobs is expensive, the `BlobPool` requires much higher fee bumps than `LegacyPool`'s 10%.

5. **Per-account limits** — at most `maxTxsPerAccount` (16) blob transactions per sender, versus `LegacyPool`'s 64 in queue + 16 in pending.

### BlobPool Structure

```go
// core/txpool/blobpool/blobpool.go  (key fields)

type BlobPool struct {
    store  billy.Database               // On-disk persistent storage
    stored uint64                       // Total bytes on disk
    limbo  *limbo                       // Included-but-not-finalised blob storage

    lookup *lookup                          // hash → storage mapping
    index  map[common.Address][]*blobTxMeta // Per-account tx metadata, sorted by nonce
    spent  map[common.Address]*uint256.Int  // Cumulative cost per account
    evict  *evictHeap                       // Priority queue for eviction
    // ...
}
```

- **`store`** — the `billy.Database` is a size-class-based persistent store. Transactions are written to disk on `Add()` and deleted when they are finalised. The `Datacap` config (currently ~2.5 GB, with a TODO to raise to 10 GB) limits total on-disk usage.
- **`limbo`** — a secondary persistent store for blob transactions that have been included in a block but not yet **finalised**. This is necessary because blob data is not part of the execution chain — a reorg could require re-pooling "lost" blobs. Once a block is finalised, its blobs are purged from limbo.
- **`index`** — per-account metadata slices, sorted by nonce. Each `blobTxMeta` is a few hundred bytes (hash, versioned hashes, nonce, fee caps, gas, eviction priorities), versus hundreds of KB for the full transaction with blobs. This keeps the in-memory footprint manageable.
- **`evict`** — a priority heap that determines which account's transactions to drop when the pool reaches capacity.

### The blobTxMeta and Eviction Priority

Each blob transaction in memory is represented by a compact metadata struct:

```go
// core/txpool/blobpool/blobpool.go

type blobTxMeta struct {
    hash    common.Hash
    vhashes []common.Hash // Blob versioned hashes
    version byte          // Sidecar version (0 = legacy, 1 = Osaka cell proofs)

    id      uint64       // Storage ID in billy.Database
    size    uint64       // RLP-encoded size including blobs
    nonce   uint64
    costCap *uint256.Int // tx.Cost()
    execTipCap  *uint256.Int
    execFeeCap  *uint256.Int
    blobFeeCap  *uint256.Int
    execGas     uint64
    blobGas     uint64

    basefeeJumps float64 // log1.125(execFeeCap) - log1.125(currentBaseFee)
    blobfeeJumps float64 // log1.125(blobFeeCap) - log1.125(currentBlobFee)

    evictionExecTip      *uint256.Int // worst tip across all prior nonces
    evictionExecFeeJumps float64      // worst base fee jumps across all prior nonces
    evictionBlobFeeJumps float64      // worst blob fee jumps across all prior nonces
    // ...
}
```

The eviction algorithm is notably sophisticated. Blob transactions have **three** independent price dimensions (execution tip, execution fee cap, blob fee cap), making a simple total ordering impossible. The `BlobPool` reduces this dimensionality through several steps:

1. **Fee jumps** — convert absolute fees to "jumps", the number of 1.125x fee adjustments between the current network fee and the transaction's cap. This normalises across different fee magnitudes.
2. **Priority per dimension** — for each fee dimension (base fee, blob fee), compute the difference between the transaction's jumps and the current network fee's jumps, then compress with `sign(diff) * log2(abs(diff))`. This reduces noise at high values.
3. **Combine dimensions** — take `min(basePriority, blobPriority)`. The binding constraint (whichever fee is closest to exceeding the cap) determines the priority. Positive values (fees well below caps) are clamped to 0 to prevent pool wars.
4. **Tip as tiebreaker** — within the same priority bucket, the execution tip breaks ties.
5. **Worst-across-nonces** — track the worst priority values across all of an account's nonce sequence. A cheap transaction at nonce 5 degrades the priority of the expensive transaction at nonce 10, since nonce 5 must execute first.

When the pool exceeds `Datacap`, the account with the worst eviction priority has its **highest-nonce** transaction dropped. The highest nonce is chosen because it is the furthest from execution, and dropping the lowest nonce would create a gap (which is forbidden).

### Filter

The `BlobPool`'s `Filter()` is simple — it accepts only type 3 (blob) transactions:

```go
// core/txpool/blobpool/blobpool.go

func (p *BlobPool) Filter(tx *types.Transaction) bool {
    return tx.Type() == types.BlobTxType
}
```

---

## Event Subscription

Both subpools emit `core.NewTxsEvent` when new transactions are accepted. The coordinator joins these subscriptions:

```go
// core/txpool/txpool.go

func (p *TxPool) SubscribeTransactions(ch chan<- core.NewTxsEvent, reorgs bool) event.Subscription {
    subs := make([]event.Subscription, len(p.subpools))
    for i, subpool := range p.subpools {
        subs[i] = subpool.SubscribeTransactions(ch, reorgs)
    }
    return p.subs.Track(event.JoinSubscriptions(subs...))
}
```

The `eth/handler.go` module subscribes to this feed and broadcasts new transactions to peers — either by sending the full transaction or by announcing the hash for the peer to request later. This is how transactions propagate across the network.

In `LegacyPool`, event emission is batched through the reorg cycle: newly promoted transactions are collected during `runReorg()` and sent as a single `NewTxsEvent` at the end. This avoids firing events for transactions that might be immediately invalidated by a concurrent reset.

---

## Putting It All Together

Here is how the full transaction lifecycle connects to the rest of the system:

1. **Arrival** — a peer sends a transaction via the Ethereum wire protocol (see Chapter 12), or a user submits via `eth_sendRawTransaction` (see Chapter 13). Both end up calling `TxPool.Add()`.
2. **Validation and pooling** — the transaction is validated, inserted into the appropriate subpool, and potentially promoted to the pending set.
3. **Broadcasting** — the pool emits a `NewTxsEvent`, which the handler picks up and relays to peers.
4. **Block building** — the miner (see Chapter 09) calls `TxPool.Pending()` to get executable transactions, orders them by effective tip, and executes them via `StateProcessor.Process()` (see Chapter 06).
5. **Inclusion** — a `ChainHeadEvent` fires. The coordinator triggers `Reset()` on all subpools. `demoteUnexecutables()` removes included transactions from pending. `promoteExecutables()` promotes newly-valid queued transactions.
6. **Reorg** — if the new head is not a direct descendant of the old head, `reset()` walks back to the common ancestor, reinjects "lost" transactions, and repeats steps 5's cleanup.

The pool is the bridge between geth's networking layer and its execution engine — it ensures that only valid, properly-priced transactions reach the miner, while handling the complexity of nonce ordering, balance accounting, fee markets, and chain reorganisations.
