---
title: Geth(13) JSON-RPC and Accounts
published: 2026-04-22
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

Every subsystem in previous chapters — block insertion, state management, the EVM, transaction pools, sync — runs inside geth. External clients (wallets, dApps, scripts, the beacon client) interact with geth through its **JSON-RPC** interface. This chapter covers how incoming RPC requests travel from the network transport through method dispatch and into the core, and how geth manages cryptographic keys for signing transactions.

---

## How a JSON-RPC Call Reaches Geth

When a client sends a request like `eth_getBalance`, it passes through these layers:

```
 Client sends JSON-RPC request
      |
      v
 1. Transport layer (HTTP / WebSocket / IPC)
      |
      v
 2. Server decodes JSON, creates handler
      |
      v
 3. handler dispatches to method:
      |
      +-- handleCallMsg()           -- classify: call, notification, subscribe
      |      |
      |      +-- handleCall()       -- look up callback in serviceRegistry
      |      |      |
      |      |      +-- handleSubscribe()   (if *_subscribe method)
      |      |      |
      |      |      +-- registry.callback("eth_getBalance")
      |      |             |
      |      |             +-- splits on "_" -> service "eth", method "getBalance"
      |      |             +-- returns *callback (reflected Go method)
      |      |
      |      +-- parsePositionalArguments()  -- decode JSON params to Go types
      |      |
      |      +-- callb.call(ctx, method, args)  -- invoke the Go method
      |
      v
 4. API struct method executes (e.g. BlockChainAPI.GetBalance)
      |
      v
 5. Backend interface delegates to core (blockchain, txpool, state)
      |
      v
 6. Response serialized as JSON and sent back
```

Each layer is covered in detail below.

---

## Transport Layer

Geth exposes JSON-RPC over three transports, each suited to different use cases:

| Transport | Protocol | Subscriptions | Typical Use |
|-----------|----------|---------------|-------------|
| **HTTP** | Request/response | No | Wallets, scripts, remote access |
| **WebSocket** | Full-duplex | Yes | dApps needing `eth_subscribe` |
| **IPC** | Unix socket | Yes | Local tooling (`geth attach`) |

### HTTP

The HTTP transport handles each request independently — no persistent connection state:

```go
// rpc/server.go

func (s *Server) serveSingleRequest(ctx context.Context, codec ServerCodec) {
    if !s.run.Load() {
        return
    }
    h := newHandler(ctx, codec, s.idgen, &s.services, s.batchItemLimit, s.batchResponseLimit)
    h.allowSubscribe = false
    defer h.close(io.EOF, nil)

    reqs, batch, err := codec.readBatch()
    // ...
    if batch {
        h.handleBatch(reqs)
    } else {
        h.handleMsg(reqs[0])
    }
}
```

A new `handler` is created per HTTP request. Subscriptions are explicitly disabled (`allowSubscribe = false`) because HTTP has no way to push notifications back to the client. The request is read, dispatched, and the handler is closed.

Default limits protect against abuse: HTTP body size is capped at 5MB (`defaultBodyLimit`), and configurable batch limits control the number of items and total response size.

### WebSocket

WebSocket connections are long-lived, making them suitable for subscriptions:

```go
// rpc/websocket.go

const (
    wsPingInterval     = 30 * time.Second
    wsPongTimeout      = 30 * time.Second
    wsDefaultReadLimit = 32 * 1024 * 1024  // 32 MB
)
```

The server upgrades an HTTP connection to WebSocket using the Gorilla library, then calls `ServeCodec()` which creates a persistent handler. Because the connection stays open, the handler can push subscription notifications to the client. A ping/pong heartbeat every 30 seconds detects dead connections.

### IPC

IPC uses a Unix domain socket (or named pipe on Windows), providing the fastest transport for local access:

```go
// rpc/ipc.go

func (s *Server) ServeListener(l net.Listener) {
    for {
        conn, err := l.Accept()
        if err != nil {
            // ...
            return
        }
        go s.ServeCodec(NewCodec(conn), 0)
    }
}
```

Each accepted connection gets its own goroutine running `ServeCodec`. Like WebSocket, IPC supports subscriptions since the connection is persistent.

---

## The Server and Service Registry

The `Server` struct manages all registered API methods and active connections:

```go
// rpc/server.go

type Server struct {
    services           serviceRegistry
    idgen              func() ID

    mutex              sync.Mutex
    codecs             map[ServerCodec]struct{}
    run                atomic.Bool
    batchItemLimit     int
    batchResponseLimit int
    httpBodyLimit      int
    wsReadLimit        int64
}
```

The key field is `services` — a `serviceRegistry` that maps namespace/method names to Go functions.

### How Methods Are Registered

When geth starts, each subsystem registers its API methods with the server:

```go
// rpc/server.go

func (s *Server) RegisterName(name string, receiver interface{}) error {
    return s.services.registerName(name, receiver)
}
```

For example, `RegisterName("eth", blockChainAPI)` registers all exported methods of `blockChainAPI` under the `eth` namespace. The registry uses reflection to discover methods:

```go
// rpc/service.go

type serviceRegistry struct {
    mu       sync.Mutex
    services map[string]service
}

type service struct {
    name          string
    callbacks     map[string]*callback
    subscriptions map[string]*callback
}

type callback struct {
    fn          reflect.Value
    rcvr        reflect.Value
    argTypes    []reflect.Type
    hasCtx      bool
    errPos      int
    isSubscribe bool
}
```

The `registerName()` function calls `suitableCallbacks()`, which iterates over the receiver's exported methods and validates each one:

```go
// rpc/service.go

func suitableCallbacks(receiver reflect.Value) map[string]*callback {
    typ := receiver.Type()
    callbacks := make(map[string]*callback)
    for m := 0; m < typ.NumMethod(); m++ {
        method := typ.Method(m)
        if method.PkgPath != "" {
            continue // method not exported
        }
        cb := newCallback(receiver, method.Func)
        if cb == nil {
            continue // function invalid
        }
        name := formatName(method.Name)
        callbacks[name] = cb
    }
    return callbacks
}
```

A Go method qualifies as an RPC callback if it:

1. Is exported (uppercase first letter).
2. Optionally takes `context.Context` as its first argument (after the receiver).
3. Returns at most two values: an optional result and an optional `error` (error must be last).

The method name is transformed by `formatName()`, which lowercases the first character: `GetBalance` becomes `getBalance`. Combined with the namespace, this produces the JSON-RPC method name `eth_getBalance`.

Methods that return `*rpc.Subscription` are registered as subscriptions rather than regular callbacks. These handle `eth_subscribe` requests.

---

## Request Dispatch

When a message arrives, the `handler` routes it through several stages:

```go
// rpc/handler.go

type handler struct {
    reg                  *serviceRegistry
    unsubscribeCb        *callback
    idgen                func() ID
    respWait             map[string]*requestOp
    clientSubs           map[string]*ClientSubscription
    callWG               sync.WaitGroup
    rootCtx              context.Context
    cancelRoot           func()
    conn                 jsonWriter
    log                  log.Logger
    allowSubscribe       bool
    batchRequestLimit    int
    batchResponseMaxSize int

    subLock    sync.Mutex
    serverSubs map[ID]*Subscription
}
```

### Single Request Flow

For a non-batch request, `handleMsg()` starts a call proc and delegates:

```go
// rpc/handler.go

func (h *handler) handleCallMsg(ctx *callProc, msg *jsonrpcMessage) *jsonrpcMessage {
    start := time.Now()
    switch {
    case msg.isNotification():
        h.handleCall(ctx, msg)
        h.log.Debug("Served "+msg.Method, "duration", time.Since(start))
        return nil

    case msg.isCall():
        resp := h.handleCall(ctx, msg)
        // ... log duration and errors ...
        return resp

    case msg.hasValidID():
        return msg.errorResponse(&invalidRequestError{"invalid request"})

    default:
        return errorMessage(&invalidRequestError{"invalid request"})
    }
}
```

The `handleCall()` function does the actual method lookup and invocation:

```go
// rpc/handler.go

func (h *handler) handleCall(cp *callProc, msg *jsonrpcMessage) *jsonrpcMessage {
    if msg.isSubscribe() {
        return h.handleSubscribe(cp, msg)
    }
    var callb *callback
    if msg.isUnsubscribe() {
        callb = h.unsubscribeCb
    } else {
        callb = h.reg.callback(msg.Method)
    }
    if callb == nil {
        return msg.errorResponse(&methodNotFoundError{method: msg.Method})
    }

    args, err := parsePositionalArguments(msg.Params, callb.argTypes)
    if err != nil {
        return msg.errorResponse(&invalidParamsError{err.Error()})
    }
    start := time.Now()
    answer := h.runMethod(cp.ctx, msg, callb, args)
    // ... metrics ...
    return answer
}
```

The dispatch chain:

1. **Subscription check** — if the method ends with `_subscribe`, route to `handleSubscribe()`.
2. **Unsubscribe check** — `*_unsubscribe` calls use the built-in unsubscribe callback.
3. **Method lookup** — `h.reg.callback(msg.Method)` splits the method string on `_` (e.g., `eth_getBalance` → service `eth`, method `getBalance`) and looks up the corresponding `callback`.
4. **Argument parsing** — `parsePositionalArguments()` decodes the JSON params array into the Go types expected by the callback.
5. **Invocation** — `callb.call()` builds the full argument list (prepending the receiver and context), calls the Go method via reflection, and catches panics.

### Subscriptions

Subscription requests (e.g., `eth_subscribe("newHeads")`) are handled specially:

```go
// rpc/handler.go

func (h *handler) handleSubscribe(cp *callProc, msg *jsonrpcMessage) *jsonrpcMessage {
    if !h.allowSubscribe {
        return msg.errorResponse(ErrNotificationsUnsupported)
    }
    name, err := parseSubscriptionName(msg.Params)
    // ...
    namespace := msg.namespace()
    callb := h.reg.subscription(namespace, name)
    // ...

    n := &Notifier{h: h, namespace: namespace}
    cp.notifiers = append(cp.notifiers, n)
    ctx := context.WithValue(cp.ctx, notifierKey{}, n)

    return h.runMethod(ctx, msg, callb, args)
}
```

A `Notifier` is injected into the context. The subscription handler retrieves it via `NotifierFromContext(ctx)`, creates a `Subscription`, and starts pushing notifications:

```go
// rpc/subscription.go

type Notifier struct {
    h         *handler
    namespace string

    mu           sync.Mutex
    sub          *Subscription
    buffer       []any
    callReturned bool
    activated    bool
}

type Subscription struct {
    ID        ID
    namespace string
    err       chan error
}
```

Notifications sent before the subscription ID is delivered to the client are buffered. Once the handler sends the subscription ID response, `activate()` flushes the buffer and enables direct delivery.

### Timeout Handling

HTTP requests support server-side timeouts. The `handleNonBatchCall()` function creates a timer that cancels the context and sends an error response if the method doesn't return in time:

```go
// rpc/handler.go  (simplified)

func (h *handler) handleNonBatchCall(cp *callProc, msg *jsonrpcMessage) {
    var responded sync.Once
    cp.ctx, cancel = context.WithCancel(cp.ctx)
    defer cancel()

    if timeout, ok := ContextRequestTimeout(cp.ctx); ok {
        timer = time.AfterFunc(timeout, func() {
            cancel()
            responded.Do(func() {
                resp := msg.errorResponse(&internalServerError{errcodeTimeout, errMsgTimeout})
                h.conn.writeJSON(cp.ctx, resp, true)
            })
        })
    }

    answer := h.handleCallMsg(cp, msg)
    // ...
    if answer != nil {
        responded.Do(func() {
            h.conn.writeJSON(cp.ctx, answer, false)
        })
    }
}
```

The `sync.Once` ensures exactly one response is sent, whether it's the normal result or the timeout error.

---

## Block Number Resolution

Many RPC methods accept a block identifier — either a number or a hash. The `BlockNumber` type handles the special string values:

```go
// rpc/types.go

type BlockNumber int64

const (
    EarliestBlockNumber  = BlockNumber(-5)
    SafeBlockNumber      = BlockNumber(-4)
    FinalizedBlockNumber = BlockNumber(-3)
    LatestBlockNumber    = BlockNumber(-2)
    PendingBlockNumber   = BlockNumber(-1)
)
```

JSON-RPC clients can pass `"latest"`, `"pending"`, `"finalized"`, `"safe"`, `"earliest"`, or a hex block number. The `UnmarshalJSON` method converts these strings to the corresponding negative sentinel values.

For methods that accept either a block number or a block hash, `BlockNumberOrHash` provides a union type:

```go
// rpc/types.go

type BlockNumberOrHash struct {
    BlockNumber      *BlockNumber `json:"blockNumber,omitempty"`
    BlockHash        *common.Hash `json:"blockHash,omitempty"`
    RequireCanonical bool         `json:"requireCanonical,omitempty"`
}
```

When `RequireCanonical` is true and a block hash is provided, the backend verifies that the block is on the canonical chain, not a fork.

---

## The API Structs

The actual RPC method implementations live in `internal/ethapi/api.go`, split across several structs by domain. Each struct wraps the `Backend` interface:

```go
// internal/ethapi/backend.go

func GetAPIs(apiBackend Backend) []rpc.API {
    nonceLock := new(AddrLocker)
    return []rpc.API{
        {Namespace: "eth", Service: NewEthereumAPI(apiBackend)},
        {Namespace: "eth", Service: NewBlockChainAPI(apiBackend)},
        {Namespace: "eth", Service: NewTransactionAPI(apiBackend, nonceLock)},
        {Namespace: "txpool", Service: NewTxPoolAPI(apiBackend)},
        {Namespace: "debug", Service: NewDebugAPI(apiBackend)},
        {Namespace: "eth", Service: NewEthereumAccountAPI(apiBackend.AccountManager())},
    }
}
```

Multiple structs can share the same namespace (`eth`). The registry merges their methods: `EthereumAPI.GasPrice` → `eth_gasPrice`, `BlockChainAPI.GetBalance` → `eth_getBalance`, `TransactionAPI.SendRawTransaction` → `eth_sendRawTransaction`.

The `Ethereum` service in `eth/backend.go` adds more namespaces when registering with the node:

```go
// eth/backend.go

func (s *Ethereum) APIs() []rpc.API {
    apis := ethapi.GetAPIs(s.APIBackend)
    return append(apis, []rpc.API{
        {Namespace: "miner", Service: NewMinerAPI(s)},
        {Namespace: "eth", Service: downloader.NewDownloaderAPI(...)},
        {Namespace: "admin", Service: NewAdminAPI(s)},
        {Namespace: "debug", Service: NewDebugAPI(s)},
        {Namespace: "net", Service: s.netRPCService},
    }...)
}
```

| Struct | Namespace | Key Methods |
|--------|-----------|-------------|
| `EthereumAPI` | `eth` | `gasPrice`, `maxPriorityFeePerGas`, `feeHistory`, `syncing` |
| `BlockChainAPI` | `eth` | `blockNumber`, `getBalance`, `getBlockByNumber`, `call`, `estimateGas`, `getProof` |
| `TransactionAPI` | `eth` | `sendRawTransaction`, `sendTransaction`, `getTransactionByHash`, `getTransactionReceipt` |
| `TxPoolAPI` | `txpool` | `content`, `status`, `inspect` |
| `DebugAPI` | `debug` | `getRawHeader`, `getRawBlock`, `getRawReceipts` |
| `EthereumAccountAPI` | `eth` | `accounts` |
| `NetAPI` | `net` | `version`, `listening`, `peerCount` |

---

## The Backend Interface

All API structs delegate to the `Backend` interface, which abstracts access to geth's core subsystems:

```go
// internal/ethapi/backend.go

type Backend interface {
    // General
    SyncProgress(ctx context.Context) ethereum.SyncProgress
    SuggestGasTipCap(ctx context.Context) (*big.Int, error)
    FeeHistory(ctx context.Context, blockCount uint64, lastBlock rpc.BlockNumber,
        rewardPercentiles []float64) (*big.Int, [][]*big.Int, []*big.Int, []float64,
        []*big.Int, []float64, error)
    BlobBaseFee(ctx context.Context) *big.Int
    ChainDb() ethdb.Database
    AccountManager() *accounts.Manager
    ExtRPCEnabled() bool
    RPCGasCap() uint64
    RPCEVMTimeout() time.Duration
    RPCTxFeeCap() float64
    UnprotectedAllowed() bool

    // Blockchain
    HeaderByNumber(ctx context.Context, number rpc.BlockNumber) (*types.Header, error)
    HeaderByHash(ctx context.Context, hash common.Hash) (*types.Header, error)
    HeaderByNumberOrHash(ctx context.Context, blockNrOrHash rpc.BlockNumberOrHash) (*types.Header, error)
    CurrentHeader() *types.Header
    CurrentBlock() *types.Header
    BlockByNumber(ctx context.Context, number rpc.BlockNumber) (*types.Block, error)
    BlockByHash(ctx context.Context, hash common.Hash) (*types.Block, error)
    BlockByNumberOrHash(ctx context.Context, blockNrOrHash rpc.BlockNumberOrHash) (*types.Block, error)
    StateAndHeaderByNumber(ctx context.Context, number rpc.BlockNumber) (*state.StateDB, *types.Header, error)
    StateAndHeaderByNumberOrHash(ctx context.Context, blockNrOrHash rpc.BlockNumberOrHash) (*state.StateDB, *types.Header, error)
    Pending() (*types.Block, types.Receipts, *state.StateDB)
    GetReceipts(ctx context.Context, hash common.Hash) (types.Receipts, error)
    GetEVM(ctx context.Context, state *state.StateDB, header *types.Header,
        vmConfig *vm.Config, blockCtx *vm.BlockContext) *vm.EVM
    SubscribeChainEvent(ch chan<- core.ChainEvent) event.Subscription
    SubscribeChainHeadEvent(ch chan<- core.ChainHeadEvent) event.Subscription

    // Transaction pool
    SendTx(ctx context.Context, signedTx *types.Transaction) error
    GetCanonicalTransaction(txHash common.Hash) (bool, *types.Transaction, common.Hash, uint64, uint64)
    GetPoolTransactions() (types.Transactions, error)
    GetPoolTransaction(txHash common.Hash) *types.Transaction
    GetPoolNonce(ctx context.Context, addr common.Address) (uint64, error)
    Stats() (pending int, queued int)
    TxPoolContent() (map[common.Address][]*types.Transaction, map[common.Address][]*types.Transaction)
    SubscribeNewTxsEvent(chan<- core.NewTxsEvent) event.Subscription

    ChainConfig() *params.ChainConfig
    Engine() consensus.Engine
    // ... filter-related methods ...
}
```

The interface groups into four areas: general queries (gas prices, sync progress), blockchain reads (headers, blocks, state), transaction pool operations, and event subscriptions. This separation allows the same API code to work with different backends — the full `Ethereum` backend and the lighter `LESAPIBackend` for light clients.

### EthAPIBackend: The Implementation

The `EthAPIBackend` struct bridges the API layer to the actual `Ethereum` service:

```go
// eth/api_backend.go

type EthAPIBackend struct {
    extRPCEnabled       bool
    allowUnprotectedTxs bool
    eth                 *Ethereum
    gpo                 *gasprice.Oracle
}
```

Most methods delegate directly to the underlying subsystem. For example, `StateAndHeaderByNumberOrHash` resolves the block identifier and opens a state database at that block's root:

```go
// eth/api_backend.go  (simplified)

func (b *EthAPIBackend) HeaderByNumber(ctx context.Context, number rpc.BlockNumber) (*types.Header, error) {
    if number == rpc.PendingBlockNumber {
        block, _, _ := b.eth.miner.Pending()
        return block.Header(), nil
    }
    if number == rpc.LatestBlockNumber {
        return b.eth.blockchain.CurrentBlock(), nil
    }
    if number == rpc.FinalizedBlockNumber {
        return b.eth.blockchain.CurrentFinalBlock(), nil
    }
    if number == rpc.SafeBlockNumber {
        return b.eth.blockchain.CurrentSafeBlock(), nil
    }
    if number == rpc.EarliestBlockNumber {
        return b.eth.blockchain.GetHeaderByNumber(b.HistoryPruningCutoff()), nil
    }
    return b.eth.blockchain.GetHeaderByNumber(uint64(number)), nil
}
```

This is where the `BlockNumber` sentinel values are resolved to actual headers from the blockchain (see Chapter 10 for the head pointers).

---

## Key API Methods

### eth_getBalance

The simplest read operation — look up an account's balance at a given block:

```go
// internal/ethapi/api.go

func (api *BlockChainAPI) GetBalance(ctx context.Context, address common.Address,
    blockNrOrHash rpc.BlockNumberOrHash) (*hexutil.Big, error) {

    state, _, err := api.b.StateAndHeaderByNumberOrHash(ctx, blockNrOrHash)
    if state == nil || err != nil {
        return nil, err
    }
    b := state.GetBalance(address).ToBig()
    return (*hexutil.Big)(b), state.Error()
}
```

The pattern is: resolve the block → open a `StateDB` at that block's state root → read from state (see Chapter 04 for `StateDB.GetBalance`).

### eth_call

`eth_call` executes a transaction against a given state **without** modifying the chain. It is the workhorse behind dApp read operations like token balances, DEX quotes, and contract simulations:

```go
// internal/ethapi/api.go

func (api *BlockChainAPI) Call(ctx context.Context, args TransactionArgs,
    blockNrOrHash *rpc.BlockNumberOrHash, overrides *override.StateOverride,
    blockOverrides *override.BlockOverrides) (hexutil.Bytes, error) {

    if blockNrOrHash == nil {
        latest := rpc.BlockNumberOrHashWithNumber(rpc.LatestBlockNumber)
        blockNrOrHash = &latest
    }
    result, err := DoCall(ctx, api.b, args, *blockNrOrHash, overrides, blockOverrides,
        api.b.RPCEVMTimeout(), api.b.RPCGasCap())
    if err != nil {
        return nil, err
    }
    if errors.Is(result.Err, vm.ErrExecutionReverted) {
        return nil, newRevertError(result.Revert())
    }
    return result.Return(), result.Err
}
```

The execution pipeline in `doCall()` sets up the EVM and runs the transaction:

```go
// internal/ethapi/api.go  (simplified)

func doCall(ctx context.Context, b Backend, args TransactionArgs, state *state.StateDB,
    header *types.Header, overrides *override.StateOverride,
    blockOverrides *override.BlockOverrides, timeout time.Duration,
    globalGasCap uint64) (*core.ExecutionResult, error) {

    // 1. Create EVM block context from the target header
    blockCtx := core.NewEVMBlockContext(header, NewChainContext(ctx, b), nil)

    // 2. Apply block overrides (custom block number, timestamp, etc.)
    if blockOverrides != nil {
        blockOverrides.Apply(&blockCtx)
    }

    // 3. Apply state overrides (custom balances, code, storage)
    rules := b.ChainConfig().Rules(blockCtx.BlockNumber, blockCtx.Random != nil, blockCtx.Time)
    precompiles := vm.ActivePrecompiledContracts(rules)
    overrides.Apply(state, precompiles)

    // 4. Set up gas pool and timeout
    var cancel context.CancelFunc
    if timeout > 0 {
        ctx, cancel = context.WithTimeout(ctx, timeout)
    } else {
        ctx, cancel = context.WithCancel(ctx)
    }
    defer cancel()
    gp := new(core.GasPool)
    if globalGasCap == 0 {
        gp.AddGas(gomath.MaxUint64)
    } else {
        gp.AddGas(globalGasCap)
    }

    // 5. Execute the message
    return applyMessage(ctx, b, args, state, header, timeout, gp, &blockCtx, ...)
}
```

Walking through the key steps:

- **Step 1** constructs the EVM block context from the target block's header.
- **Steps 2–3** are the override mechanism: callers can modify block fields (number, timestamp, base fee) and state (account balances, contract code, storage slots) before execution. This powers tools like Tenderly simulations.
- **Step 4** creates a gas pool. If `RPCGasCap` is configured (default: 50M gas), it limits the gas available for the call. If zero, the call gets unlimited gas.
- **Step 5** converts `TransactionArgs` to a `core.Message`, creates an EVM instance, and calls `core.ApplyMessage()` — the same execution path used for real transactions (see Chapter 06). A goroutine monitors the context and cancels the EVM if the timeout expires.

### TransactionArgs

The `TransactionArgs` struct represents the input to `eth_call`, `eth_estimateGas`, and `eth_sendTransaction`:

```go
// internal/ethapi/transaction_args.go

type TransactionArgs struct {
    From                 *common.Address
    To                   *common.Address
    Gas                  *hexutil.Uint64
    GasPrice             *hexutil.Big
    MaxFeePerGas         *hexutil.Big
    MaxPriorityFeePerGas *hexutil.Big
    Value                *hexutil.Big
    Nonce                *hexutil.Uint64

    Data  *hexutil.Bytes
    Input *hexutil.Bytes

    AccessList *types.AccessList
    ChainID    *hexutil.Big

    BlobFeeCap *hexutil.Big
    BlobHashes []common.Hash

    Blobs       []kzg4844.Blob
    Commitments []kzg4844.Commitment
    Proofs      []kzg4844.Proof

    AuthorizationList []types.SetCodeAuthorization
}
```

All fields are pointers so callers can omit any of them. The `CallDefaults()` method fills in sensible defaults for omitted fields (zero value, gas cap, etc.), and `ToMessage()` converts the args into a `core.Message` for EVM execution.

### State and Block Overrides

`eth_call` supports two override mechanisms that let callers customize the execution environment:

```go
// internal/ethapi/override/override.go

type StateOverride map[common.Address]OverrideAccount

type OverrideAccount struct {
    Nonce            *hexutil.Uint64
    Code             *hexutil.Bytes
    Balance          *hexutil.Big
    State            map[common.Hash]common.Hash // complete state replacement
    StateDiff        map[common.Hash]common.Hash // incremental state diff
    MovePrecompileTo *common.Address
}

type BlockOverrides struct {
    Number        *hexutil.Big
    Time          *hexutil.Uint64
    GasLimit      *hexutil.Uint64
    FeeRecipient  *common.Address
    PrevRandao    *common.Hash
    BaseFeePerGas *hexutil.Big
    BlobBaseFee   *hexutil.Big
    // ...
}
```

`StateOverride` lets callers replace an account's balance, nonce, code, or storage before the call executes. `BlockOverrides` lets callers modify block context fields like the block number and timestamp. These are applied to the in-memory state and block context only — nothing is written to disk.

### eth_sendRawTransaction

This is how signed transactions enter geth from external clients:

```go
// internal/ethapi/api.go

func (api *TransactionAPI) SendRawTransaction(ctx context.Context,
    input hexutil.Bytes) (common.Hash, error) {

    tx := new(types.Transaction)
    if err := tx.UnmarshalBinary(input); err != nil {
        return common.Hash{}, err
    }
    // Handle blob sidecar version conversion if needed
    if sc := tx.BlobTxSidecar(); sc != nil {
        // ...convert v0 -> v1 if needed...
    }
    return SubmitTransaction(ctx, api.b, tx)
}
```

The input is a raw RLP-encoded transaction. After decoding, `SubmitTransaction()` performs final checks and submits to the pool:

```go
// internal/ethapi/api.go  (simplified)

func SubmitTransaction(ctx context.Context, b Backend, tx *types.Transaction) (common.Hash, error) {
    // Check fee cap against RPCTxFeeCap
    if err := checkTxFee(tx.GasPrice(), tx.Gas(), b.RPCTxFeeCap()); err != nil {
        return common.Hash{}, err
    }
    // Reject unprotected (non-EIP-155) transactions unless explicitly allowed
    if !b.UnprotectedAllowed() && !tx.Protected() {
        return common.Hash{}, errors.New("only replay-protected transactions allowed")
    }
    // Submit to the transaction pool
    if err := b.SendTx(ctx, tx); err != nil {
        return common.Hash{}, err
    }
    return tx.Hash(), nil
}
```

`SendTx()` delegates to the transaction pool's `Add()` method (see Chapter 08). From there, the transaction enters the pool and is broadcast to peers (see Chapter 12).

### eth_getTransactionByHash

Transaction lookup searches two places — the canonical chain and the pending pool:

```go
// internal/ethapi/api.go  (simplified)

func (api *TransactionAPI) GetTransactionByHash(ctx context.Context,
    hash common.Hash) (*RPCTransaction, error) {

    // 1. Try the canonical chain index
    found, tx, blockHash, blockNumber, index := api.b.GetCanonicalTransaction(hash)
    if found {
        header, _ := api.b.HeaderByHash(ctx, blockHash)
        return newRPCTransaction(tx, blockHash, blockNumber, header.Time, index, header.BaseFee, ...)
    }

    // 2. Try the pending pool
    if tx := api.b.GetPoolTransaction(hash); tx != nil {
        return NewRPCPendingTransaction(tx, ...)
    }

    // 3. If the transaction indexer is still building the index, return a specific error
    if !api.b.TxIndexDone() {
        return nil, errors.New("transaction indexing is in progress")
    }
    return nil, nil  // not found
}
```

Pending pool transactions are returned with null block fields, signaling to the client that the transaction hasn't been mined yet.

---

## Account Management

Geth can manage cryptographic keys for signing transactions. The account system is built around three interfaces:

```go
// accounts/accounts.go

type Account struct {
    Address common.Address
    URL     URL
}

type Wallet interface {
    URL() URL
    Status() (string, error)
    Open(passphrase string) error
    Close() error
    Accounts() []Account
    Contains(account Account) bool
    Derive(path DerivationPath, pin bool) (Account, error)
    SelfDerive(bases []DerivationPath, chain ethereum.ChainStateReader)
    SignData(account Account, mimeType string, data []byte) ([]byte, error)
    SignTx(account Account, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)
    SignTxWithPassphrase(account Account, passphrase string, tx *types.Transaction,
        chainID *big.Int) (*types.Transaction, error)
    // ...
}

type Backend interface {
    Wallets() []Wallet
    Subscribe(sink chan<- WalletEvent) event.Subscription
}
```

A `Wallet` is anything that holds keys and can sign. An `Account` is an address plus a locator URL. A `Backend` is a source of wallets (keystore directory, hardware wallet USB, etc.).

### The Manager

The `Manager` aggregates all wallet backends and provides a unified API:

```go
// accounts/manager.go

type Manager struct {
    backends    map[reflect.Type][]Backend
    updaters    []event.Subscription
    updates     chan WalletEvent
    newBackends chan newBackendEvent
    wallets     []Wallet

    feed event.Feed
    quit chan chan error
    term chan struct{}
    lock sync.RWMutex
}
```

The manager subscribes to each backend's wallet events (arrival, departure) and maintains a sorted cache of all wallets. When an API method needs to sign a transaction, it calls `Manager.Find(account)` to locate the wallet that contains the requested account.

Wallet events are broadcast to subscribers:

```go
// accounts/accounts.go

const (
    WalletArrived WalletEventType = iota  // New wallet detected
    WalletOpened                           // Wallet successfully opened
    WalletDropped                          // Wallet removed or disconnected
)

type WalletEvent struct {
    Wallet Wallet
    Kind   WalletEventType
}
```

### The KeyStore

The `KeyStore` is the most common wallet backend. It stores encrypted private keys as JSON files on disk:

```go
// accounts/keystore/keystore.go

type KeyStore struct {
    storage  keyStore
    cache    *accountCache
    changes  chan struct{}
    unlocked map[common.Address]*unlocked

    wallets     []accounts.Wallet
    updateFeed  event.Feed
    updateScope event.SubscriptionScope
    updating    bool

    mu       sync.RWMutex
    importMu sync.Mutex
}
```

Key operations:

- **`NewKeyStore(keydir, scryptN, scryptP)`** — creates a keystore backed by the given directory with specified scrypt parameters.
- **`NewAccount(passphrase)`** — generates a new ECDSA key pair, encrypts the private key with the passphrase, and writes it to disk.
- **`Unlock(account, passphrase)`** — decrypts the key and holds it in memory indefinitely.
- **`TimedUnlock(account, passphrase, timeout)`** — decrypts and holds the key for a limited duration.
- **`Lock(address)`** — removes the decrypted key from memory immediately.
- **`SignTx(account, tx, chainID)`** — signs a transaction using an unlocked key.

### Key Encryption

Private keys are stored in the **Web3 Secret Storage** format (version 3). The encryption uses scrypt for key derivation and AES-128-CTR for encryption:

```go
// accounts/keystore/passphrase.go

const (
    StandardScryptN = 1 << 18  // 262144 — ~256MB memory, ~1s CPU
    StandardScryptP = 1
    LightScryptN    = 1 << 12  // 4096   — ~4MB memory, ~100ms CPU
    LightScryptP    = 6

    scryptR     = 8
    scryptDKLen = 32
)

func EncryptDataV3(data, auth []byte, scryptN, scryptP int) (CryptoJSON, error) {
    salt := make([]byte, 32)
    io.ReadFull(rand.Reader, salt)

    derivedKey, err := scrypt.Key(auth, salt, scryptN, scryptR, scryptP, scryptDKLen)
    encryptKey := derivedKey[:16]

    iv := make([]byte, aes.BlockSize)
    io.ReadFull(rand.Reader, iv)

    cipherText, _ := aesCTRXOR(encryptKey, data, iv)
    mac := crypto.Keccak256(derivedKey[16:32], cipherText)
    // ...
}
```

The encryption flow:

1. **Key derivation** — scrypt derives a 32-byte key from the passphrase and a random salt. The `StandardScrypt` parameters use ~256MB of memory and ~1 second of CPU time, making brute-force attacks expensive.
2. **Encryption** — the first 16 bytes of the derived key encrypt the private key data using AES-128-CTR with a random IV.
3. **MAC** — `Keccak256(derivedKey[16:32] || cipherText)` produces a MAC that verifies both the passphrase and the ciphertext integrity during decryption.

The encrypted key is stored as a JSON file named `UTC--<timestamp>--<address>` in the keystore directory.

### Signing a Transaction via eth_sendTransaction

When a client calls `eth_sendTransaction` (as opposed to `eth_sendRawTransaction`), geth signs the transaction using a managed key:

```go
// internal/ethapi/api.go  (simplified)

func (api *TransactionAPI) SendTransaction(ctx context.Context,
    args TransactionArgs) (common.Hash, error) {

    account := accounts.Account{Address: args.from()}
    wallet, err := api.b.AccountManager().Find(account)
    if err != nil {
        return common.Hash{}, err
    }

    // Lock the nonce to prevent concurrent transactions from using the same nonce
    if args.Nonce == nil {
        api.nonceLock.LockAddr(args.from())
        defer api.nonceLock.UnlockAddr(args.from())
    }

    // Set defaults (nonce, gas, gas price) and build the transaction
    args.setDefaults(ctx, api.b, sidecarConfig{})
    tx := args.ToTransaction(types.LegacyTxType)

    // Sign with the wallet
    signed, err := wallet.SignTx(account, tx, api.b.ChainConfig().ChainID)
    if err != nil {
        return common.Hash{}, err
    }
    return SubmitTransaction(ctx, api.b, signed)
}
```

The `AddrLocker` (`nonceLock`) ensures that concurrent `eth_sendTransaction` calls from the same address don't race on nonce assignment.

---

## ABI Encoding

The `accounts/abi` package handles encoding and decoding of function calls and return values according to the Solidity ABI specification:

```go
// accounts/abi/abi.go

type ABI struct {
    Constructor Method
    Methods     map[string]Method
    Events      map[string]Event
    Errors      map[string]Error
    Fallback    Method
    Receive     Method
}
```

The two core operations:

```go
// accounts/abi/abi.go

func (abi ABI) Pack(name string, args ...interface{}) ([]byte, error) {
    if name == "" {
        arguments, err := abi.Constructor.Inputs.Pack(args...)
        return arguments, err
    }
    method, exist := abi.Methods[name]
    if !exist {
        return nil, fmt.Errorf("method '%s' not found", name)
    }
    arguments, err := method.Inputs.Pack(args...)
    return append(method.ID, arguments...), nil
}

func (abi ABI) Unpack(name string, data []byte) ([]interface{}, error) {
    args, err := abi.getArguments(name, data)
    return args.Unpack(data)
}
```

`Pack` encodes a function call: the 4-byte method selector (`Keccak256(signature)[:4]`) followed by ABI-encoded arguments. `Unpack` decodes return data back into Go values. These are used internally by geth's contract interaction code and are available to external tooling.

---

## Putting It All Together

Here is the complete path of an `eth_call` request from client to EVM execution and back:

1. **Transport** — the client sends a JSON-RPC POST to geth's HTTP endpoint.
2. **Server** — `ServeHTTP` validates the request, reads the JSON body, and calls `serveSingleRequest()`.
3. **Handler** — `handleCallMsg()` classifies the message as a call, `handleCall()` looks up `eth_call` → splits to service `eth`, method `call` → finds `BlockChainAPI.Call` via the `serviceRegistry`.
4. **Argument parsing** — `parsePositionalArguments()` decodes the JSON params into `TransactionArgs`, `*BlockNumberOrHash`, `*StateOverride`, and `*BlockOverrides`.
5. **Callback invocation** — `callb.call()` invokes `BlockChainAPI.Call()` via reflection.
6. **Backend** — `StateAndHeaderByNumberOrHash()` resolves the block and opens a `StateDB` at that block's state root.
7. **EVM execution** — `doCall()` creates an EVM instance and calls `core.ApplyMessage()`, executing the transaction against the state.
8. **Response** — the return data (or revert reason) is serialized as hex bytes and sent back as the JSON-RPC response.

The entire round trip happens synchronously within the HTTP handler's lifetime, with an optional timeout that cancels the EVM execution if it runs too long.