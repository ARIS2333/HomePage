---
title: Geth(14) Node Lifecycle
published: 2026-04-23
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

Now that every subsystem has been covered — from primitives and state through execution, networking, and the RPC layer — this chapter shows how all the pieces wire together. We follow a single `geth` invocation from the command line through service initialization and shutdown, answering: *how does a geth process come to life, and how does it cleanly shut down?*

## The Startup Pipeline at a Glance

When a user runs `geth`, control flows through a five-stage pipeline:

```
  main()                              cmd/geth/main.go
    |
    v
  app.Run(os.Args)                    urfave/cli framework dispatches to geth()
    |
    v
  geth()                              cmd/geth/main.go
    |-- prepare()                     Log network, bump mainnet cache to 4096 MB
    |-- makeFullNode()                Build Node + Ethereum service (see below)
    |-- startNode()                   Signal handler, wallet events
    |-- stack.Wait()                  Block until shutdown
    v
  return
```

Three functions do the heavy lifting:

| Function | File | Responsibility |
|----------|------|----------------|
| `makeFullNode()` | `cmd/geth/config.go` | Build the `Node` container and create the `Ethereum` service inside it |
| `utils.StartNode()` | `cmd/utils/cmd.go` | Call `stack.Start()`, install signal handler |
| `stack.Close()` | `node/node.go` | Reverse-order teardown of all services and resources |

Each is covered in detail below.

---

## The CLI Entry Point

Geth uses the `urfave/cli/v2` framework. The `app` variable and its `init()` function in `cmd/geth/main.go` define the entire command tree:

```go
// cmd/geth/main.go

var app = flags.NewApp("the go-ethereum command line interface")

func init() {
    app.Action = geth
    app.Commands = []*cli.Command{
        initCommand,
        importCommand,
        exportCommand,
        // ...
        consoleCommand,
        attachCommand,
        // ...
    }
    app.Flags = slices.Concat(
        nodeFlags,
        rpcFlags,
        consoleFlags,
        debug.Flags,
        metricsFlags,
    )
    app.Before = func(ctx *cli.Context) error {
        maxprocs.Set()
        flags.MigrateGlobalFlags(ctx)
        if err := debug.Setup(ctx); err != nil {
            return err
        }
        flags.CheckEnvVars(ctx, app.Flags, "GETH")
        return nil
    }
    // ...
}
```

- `app.Action = geth` means that when no subcommand is given (e.g. just `geth --syncmode snap`), the `geth()` function runs.
- `app.Before` runs before any action: it sets `GOMAXPROCS`, migrates legacy flags, and initializes logging/tracing via `debug.Setup()`.
- Flag arrays like `nodeFlags` and `rpcFlags` are large slices of `cli.Flag` definitions — P2P ports, cache sizes, RPC hosts, and so on.

The `main()` function is trivial:

```go
func main() {
    if err := app.Run(os.Args); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

### The geth() Function

When the CLI framework calls `geth()`, the full startup sequence runs:

```go
// cmd/geth/main.go

func geth(ctx *cli.Context) error {
    if args := ctx.Args().Slice(); len(args) > 0 {
        return fmt.Errorf("invalid command: %q", args[0])
    }
    prepare(ctx)
    stack := makeFullNode(ctx)
    defer stack.Close()

    startNode(ctx, stack, false)
    stack.Wait()
    return nil
}
```

- `prepare()` logs the network being used and, for mainnet without an explicit `--cache` flag, bumps the cache default from its base value to 4096 MB.
- `makeFullNode()` builds the entire service graph — the Node container, the Ethereum service, the Engine API, and optional services like GraphQL and EthStats.
- `startNode()` calls `stack.Start()` and installs a signal handler for graceful shutdown.
- `stack.Wait()` blocks on a channel until `Close()` is called. When `Close()` returns, `geth()` returns, and the process exits.

---

## Building the Node: makeFullNode()

`makeFullNode()` in `cmd/geth/config.go` is the central wiring function. It builds the `Node` container, creates the `Ethereum` service inside it, and registers optional components.

### Step 1: Configuration and Node Creation

```go
// cmd/geth/config.go

func makeFullNode(ctx *cli.Context) *node.Node {
    stack, cfg := makeConfigNode(ctx)
    // ... fork overrides, metrics setup, service registration ...
    return stack
}
```

`makeConfigNode()` handles the first half — loading configuration and creating the blank Node:

```go
func makeConfigNode(ctx *cli.Context) (*node.Node, gethConfig) {
    cfg := loadBaseConfig(ctx)
    stack, err := node.New(&cfg.Node)
    if err != nil {
        utils.Fatalf("Failed to create the protocol stack: %v", err)
    }
    if err := setAccountManagerBackends(stack.Config(), stack.AccountManager(), stack.KeyStoreDir()); err != nil {
        utils.Fatalf("Failed to set account manager backends: %v", err)
    }
    utils.SetEthConfig(ctx, stack, &cfg.Eth)
    // ...
    return stack, cfg
}
```

The configuration loading pipeline:

1. **`loadBaseConfig()`** — starts from hardcoded defaults (`ethconfig.Defaults`, `defaultNodeConfig()`), loads a TOML config file if `--config` is set, then applies CLI flags via `utils.SetNodeConfig()`.
2. **`node.New()`** — creates the Node container (detailed in the next section).
3. **`setAccountManagerBackends()`** — adds KeyStore, Ledger, Trezor, or smart card backends to the account manager (see Chapter 13 for the account system).
4. **`utils.SetEthConfig()`** — maps remaining CLI flags to `ethconfig.Config` fields (sync mode, gas price, cache sizes, etc.).

### Step 2: Ethereum Service Registration

Back in `makeFullNode()`, the Ethereum service is created and registered on the node:

```go
backend, eth := utils.RegisterEthService(stack, &cfg.Eth)
```

`RegisterEthService()` in `cmd/utils/flags.go` is a thin wrapper:

```go
func RegisterEthService(stack *node.Node, cfg *ethconfig.Config) (*eth.EthAPIBackend, *eth.Ethereum) {
    backend, err := eth.New(stack, cfg)
    if err != nil {
        Fatalf("Failed to register the Ethereum service: %v", err)
    }
    stack.RegisterAPIs(tracers.APIs(backend.APIBackend))
    return backend.APIBackend, backend
}
```

`eth.New()` is the heavyweight constructor — covered in detail later in this chapter.

### Step 3: Optional Services

After the core Ethereum service, `makeFullNode()` registers optional components:

```go
filterSystem := utils.RegisterFilterAPI(stack, backend, &cfg.Eth)

if ctx.IsSet(utils.GraphQLEnabledFlag.Name) {
    utils.RegisterGraphQLService(stack, backend, filterSystem, &cfg.Node)
}
if cfg.Ethstats.URL != "" {
    utils.RegisterEthStatsService(stack, backend, cfg.Ethstats.URL)
}
```

### Step 4: Engine API / Dev Mode / BLSync

The final block selects the consensus integration mode:

```go
if ctx.IsSet(utils.DeveloperFlag.Name) {
    simBeacon, err := catalyst.NewSimulatedBeacon(...)
    // ...
    catalyst.RegisterSimulatedBeaconAPIs(stack, simBeacon)
    stack.RegisterLifecycle(simBeacon)
} else if ctx.IsSet(utils.BeaconApiFlag.Name) {
    blsyncer := blsync.NewClient(...)
    stack.RegisterLifecycle(blsyncer)
} else {
    err := catalyst.Register(stack, eth)
    // ...
}
```

- **Normal mode** — `catalyst.Register()` sets up the Engine API endpoints for communication with an external consensus client (see Chapter 09).
- **Developer mode** (`--dev`) — a `SimulatedBeacon` replaces the external consensus client, automatically sealing blocks when transactions are pending.
- **BLSync mode** — an experimental light sync mode using beacon chain APIs.

---

## The Node Container

The `Node` struct in `node/node.go` is the central container that holds all services together. It manages the P2P server, RPC endpoints, account manager, databases, and registered lifecycles.

```go
// node/node.go

type Node struct {
    eventmux      *event.TypeMux
    config        *Config
    accman        *accounts.Manager
    log           log.Logger
    keyDir        string
    keyDirTemp    bool
    dirLock       *flock.Flock  // prevents concurrent use of instance directory
    stop          chan struct{} // Channel to wait for termination notifications
    server        *p2p.Server
    startStopLock sync.Mutex
    state         int           // Tracks state of node lifecycle

    lock          sync.Mutex
    lifecycles    []Lifecycle
    rpcAPIs       []rpc.API
    http          *httpServer
    ws            *httpServer
    httpAuth      *httpServer
    wsAuth        *httpServer
    ipc           *ipcServer
    inprocHandler *rpc.Server

    databases map[*closeTrackingDB]struct{}
}
```

Key fields:

- **`state`** — a simple state machine with three values: `initializingState` (0), `runningState` (1), `closedState` (2). Transitions are: initializing → running (via `Start()`) → closed (via `Close()`). The `startStopLock` mutex prevents concurrent Start/Close calls.
- **`lifecycles`** — a slice of everything that needs `Start()` and `Stop()` calls. The Ethereum service, local tx tracker, simulated beacon, and BLSync client all register here.
- **`rpcAPIs`** — accumulated API definitions from all services, registered via `RegisterAPIs()`.
- **`server`** — the P2P server (see Chapter 11).
- **`databases`** — all open databases are tracked so they can be auto-closed on shutdown.
- **`stop`** — a channel that `Wait()` blocks on; `doClose()` closes it to unblock.

### The Lifecycle Interface

Every service that the Node manages must implement the `Lifecycle` interface, defined in `node/lifecycle.go`:

```go
// node/lifecycle.go

type Lifecycle interface {
    Start() error
    Stop() error
}
```

Services register via `RegisterLifecycle()` during initialization (before `Start()` is called):

```go
func (n *Node) RegisterLifecycle(lifecycle Lifecycle) {
    n.lock.Lock()
    defer n.lock.Unlock()

    if n.state != initializingState {
        panic("can't register lifecycle on running/stopped node")
    }
    if slices.Contains(n.lifecycles, lifecycle) {
        panic(fmt.Sprintf("attempt to register lifecycle %T more than once", lifecycle))
    }
    n.lifecycles = append(n.lifecycles, lifecycle)
}
```

The panic on non-initializing state enforces a strict rule: all registration happens before `Start()`. The same guard exists in `RegisterAPIs()` and `RegisterProtocols()`.

### node.New()

The `New()` constructor creates a Node but does not start it:

```go
func New(conf *Config) (*Node, error) {
    // ... copy config, resolve absolute DataDir, validate Name ...

    server := rpc.NewServer()
    server.SetBatchLimits(conf.BatchRequestLimit, conf.BatchResponseMaxSize)
    node := &Node{
        config:        conf,
        inprocHandler: server,
        eventmux:      new(event.TypeMux),
        log:           conf.Logger,
        stop:          make(chan struct{}),
        server:        &p2p.Server{Config: conf.P2P},
        databases:     make(map[*closeTrackingDB]struct{}),
    }

    node.rpcAPIs = append(node.rpcAPIs, node.apis()...)
    if err := node.openDataDir(); err != nil {
        return nil, err
    }
    // ... set up keyDir, accman, p2p server config, RPC servers ...
    return node, nil
}
```

Walking through the initialization:

1. **Config copy** — the config is copied to prevent mutations after construction. `DataDir` is resolved to an absolute path.
2. **In-process RPC server** — created immediately, used by `Attach()` to create internal RPC clients.
3. **P2P server** — a `p2p.Server` is created with the P2P config, but not started yet.
4. **Built-in APIs** — `node.apis()` adds admin, debug, and web3 APIs provided by the node itself.
5. **Data directory lock** — `openDataDir()` creates the instance directory and acquires a file lock (`flock`) to prevent two geth instances from using the same datadir.
6. **Account manager** — created empty; backends (keystore, hardware wallets) are added later by `setAccountManagerBackends()`.
7. **RPC servers** — HTTP, WebSocket, authenticated HTTP/WS, and IPC server objects are created but not started.

### Node.Start()

`Start()` transitions the node from initializing to running:

```go
func (n *Node) Start() error {
    n.startStopLock.Lock()
    defer n.startStopLock.Unlock()

    n.lock.Lock()
    switch n.state {
    case runningState:
        n.lock.Unlock()
        return ErrNodeRunning
    case closedState:
        n.lock.Unlock()
        return ErrNodeStopped
    }
    n.state = runningState
    err := n.openEndpoints()
    lifecycles := make([]Lifecycle, len(n.lifecycles))
    copy(lifecycles, n.lifecycles)
    n.lock.Unlock()

    if err != nil {
        n.doClose(nil)
        return err
    }
    var started []Lifecycle
    for _, lifecycle := range lifecycles {
        if err = lifecycle.Start(); err != nil {
            break
        }
        started = append(started, lifecycle)
    }
    if err != nil {
        n.stopServices(started)
        n.doClose(nil)
    }
    return err
}
```

The startup sequence:

1. **State check** — only `initializingState` proceeds; running or closed nodes return an error.
2. **`openEndpoints()`** — starts the P2P server and all RPC endpoints (HTTP, WS, IPC, authenticated).
3. **Lifecycle startup** — each registered lifecycle is started in registration order. If any fails, all already-started lifecycles are stopped in reverse order, and the node is closed.

`openEndpoints()` is the bridge between the Node and the network:

```go
func (n *Node) openEndpoints() error {
    n.log.Info("Starting peer-to-peer node", "instance", n.server.Name)
    if err := n.server.Start(); err != nil {
        return convertFileLockError(err)
    }
    err := n.startRPC()
    if err != nil {
        n.stopRPC()
        n.server.Stop()
    }
    return err
}
```

`startRPC()` configures and launches all RPC transports. It separates APIs into two sets — unauthenticated (exposed on public HTTP/WS) and authenticated (exposed on the Engine API port with JWT authentication). The authenticated endpoint is only started when Engine API methods are present (i.e., when the set of all APIs is larger than the unauthenticated set).

---

## The Node Config

The `Config` struct in `node/config.go` controls the Node's behavior:

```go
// node/config.go

type Config struct {
    Name     string `toml:"-"`
    DataDir  string
    P2P      p2p.Config

    KeyStoreDir           string
    ExternalSigner        string
    UseLightweightKDF     bool
    InsecureUnlockAllowed bool
    USB                   bool

    IPCPath  string
    HTTPHost string
    HTTPPort int
    HTTPCors []string
    // ... HTTPVirtualHosts, HTTPModules, HTTPTimeouts, HTTPPathPrefix ...

    AuthAddr string
    AuthPort int

    WSHost string
    WSPort int
    // ... WSOrigins, WSModules, WSPathPrefix ...

    JWTSecret string
    DBEngine  string

    BatchRequestLimit    int
    BatchResponseMaxSize int
    // ...
}
```

Notable fields:

- **`DataDir`** — root data directory (e.g., `~/.ethereum`). An empty `DataDir` creates an ephemeral in-memory node.
- **`P2P`** — the full P2P configuration (max peers, listen address, NAT, bootnodes, etc.).
- **`HTTPHost`/`WSHost`** — if empty, the corresponding RPC transport is disabled. Set to `"localhost"` or `"0.0.0.0"` to enable.
- **`AuthAddr`/`AuthPort`** — the Engine API endpoint (default: `localhost:8551`), protected by JWT.
- **`DBEngine`** — selects the key-value store backend (`"leveldb"` or `"pebble"`), see Chapter 05.

---

## The Ethereum Service: eth.New() Revisited

In earlier chapters, individual subsystems were introduced in isolation. Now that the reader understands each component, here is the complete initialization order inside `eth.New()` in `eth/backend.go`:

```go
// eth/backend.go

func New(stack *node.Node, config *ethconfig.Config) (*Ethereum, error) {
    // 1. Validate config (sync mode, gas price, cache allocation)
    // 2. Open chaindata database
    chainDb, err := stack.OpenDatabaseWithOptions("chaindata", dbOptions)

    // 3. Determine state scheme (hash-based or path-based)
    scheme, err := rawdb.ParseStateScheme(config.StateScheme, chainDb)

    // 4. Load chain config and create consensus engine
    chainConfig, _, err := core.LoadChainConfig(chainDb, config.Genesis)
    engine, err := ethconfig.CreateConsensusEngine(chainConfig, chainDb)

    // 5. Assemble the Ethereum struct
    eth := &Ethereum{
        config:         config,
        chainDb:        chainDb,
        eventMux:       stack.EventMux(),
        accountManager: stack.AccountManager(),
        engine:         engine,
        // ...
    }

    // 6. Create the BlockChain
    eth.blockchain, err = core.NewBlockChain(chainDb, config.Genesis, eth.engine, options)

    // 7. Initialize log index (FilterMaps)
    eth.filterMaps, err = filtermaps.NewFilterMaps(...)

    // 8. Create transaction pools
    legacyPool := legacypool.New(config.TxPool, eth.blockchain)
    eth.blobTxPool = blobpool.New(config.BlobPool, eth.blockchain, ...)
    eth.txPool, err = txpool.New(config.TxPool.PriceLimit, eth.blockchain,
        []txpool.SubPool{legacyPool, eth.blobTxPool})

    // 9. Create the protocol handler
    eth.handler, err = newHandler(&handlerConfig{...})

    // 10. Create the miner
    eth.miner = miner.New(eth, config.Miner, eth.engine)

    // 11. Create the API backend
    eth.APIBackend = &EthAPIBackend{...}
    eth.APIBackend.gpo = gasprice.NewOracle(eth.APIBackend, config.GPO, ...)

    // 12. Register on the node
    stack.RegisterAPIs(eth.APIs())
    stack.RegisterProtocols(eth.Protocols())
    stack.RegisterLifecycle(eth)

    return eth, nil
}
```

The order matters — each step depends on the previous ones:

| Step | Component | Depends On |
|------|-----------|------------|
| 2 | chainDb | Node (for data directory) |
| 4 | engine | chainDb (for chain config) |
| 6 | BlockChain | chainDb, engine |
| 8 | TxPool | BlockChain (for chain head, validation) |
| 9 | handler | BlockChain, TxPool (for sync and broadcast) |
| 10 | miner | Ethereum, engine (for block building) |
| 11 | APIBackend | all of the above (for RPC methods) |

Step 12 is the critical wiring step: `RegisterLifecycle(eth)` means the Node will call `eth.Start()` and `eth.Stop()` during its own lifecycle. `RegisterProtocols()` adds the `eth/68` (and optionally `snap/1`) sub-protocols to the P2P server. `RegisterAPIs()` adds all JSON-RPC methods.

### Ethereum.Start()

When the Node starts the Ethereum lifecycle, `Start()` brings up the networking layer:

```go
// eth/backend.go

func (s *Ethereum) Start() error {
    if err := s.setupDiscovery(); err != nil {
        return err
    }
    s.shutdownTracker.Start()
    s.handler.Start(s.p2pServer.MaxPeers)
    s.dropper.Start(s.p2pServer, func() bool { return !s.Synced() })
    s.filterMaps.Start()
    go s.updateFilterMapsHeads()
    return nil
}
```

- **`setupDiscovery()`** — configures the discovery mix with DNS-based node sources and DHT iterators from discv4/v5 (see Chapter 11).
- **`handler.Start()`** — begins the sync process and starts transaction/block broadcast loops (see Chapter 12).
- **`dropper.Start()`** — manages connection quality, dropping poorly-performing peers.
- **`filterMaps.Start()`** — starts the log index for `eth_getLogs` queries.

### Ethereum.Stop()

Shutdown is the reverse of startup:

```go
func (s *Ethereum) Stop() error {
    // 1. Stop peer-related components
    s.discmix.Close()
    s.dropper.Stop()
    s.handler.Stop()

    // 2. Stop internal services
    ch := make(chan struct{})
    s.closeFilterMaps <- ch
    <-ch
    s.filterMaps.Stop()
    s.txPool.Close()
    s.blockchain.Stop()
    s.engine.Close()

    // 3. Mark clean shutdown, close database
    s.shutdownTracker.Stop()
    s.chainDb.Close()
    s.eventMux.Stop()
    return nil
}
```

The ordering ensures no component reads from a resource that has already been closed:
1. Stop networking first (handler, discovery) so no new data arrives.
2. Stop internal processing (filter maps, tx pool, blockchain, engine).
3. Close the database last, after all readers and writers have stopped.

---

## Signal Handling and Graceful Shutdown

The signal handler is installed by `utils.StartNode()` in `cmd/utils/cmd.go`:

```go
// cmd/utils/cmd.go

func StartNode(ctx *cli.Context, stack *node.Node, isConsole bool) {
    if err := stack.Start(); err != nil {
        Fatalf("Error starting protocol stack: %v", err)
    }
    go func() {
        sigc := make(chan os.Signal, 1)
        signal.Notify(sigc, syscall.SIGINT, syscall.SIGTERM)
        defer signal.Stop(sigc)

        // ... disk space monitoring setup ...

        shutdown := func() {
            log.Info("Got interrupt, shutting down...")
            go stack.Close()
            for i := 10; i > 0; i-- {
                <-sigc
                if i > 1 {
                    log.Warn("Already shutting down, interrupt more to panic.", "times", i-1)
                }
            }
            debug.Exit()
            debug.LoudPanic("boom")
        }

        if isConsole {
            for {
                sig := <-sigc
                if sig == syscall.SIGTERM {
                    shutdown()
                    return
                }
            }
        } else {
            <-sigc
            shutdown()
        }
    }()
}
```

The shutdown flow:

1. First `SIGINT` (Ctrl-C) or `SIGTERM` triggers `shutdown()`.
2. `stack.Close()` runs in a separate goroutine — it can take time to flush databases and stop services.
3. The handler then waits for 10 more signals. Each additional interrupt prints a warning with a countdown.
4. After 10 interrupts, `debug.LoudPanic("boom")` force-kills the process — a last resort for a stuck shutdown.
5. In console mode, `SIGINT` is ignored (it's handled by the JavaScript console), and only `SIGTERM` triggers shutdown.

`StartNode()` also sets up disk space monitoring. A background goroutine checks free disk space every 30 seconds; if it drops below the critical threshold (default: 2 × `TrieDirtyCache`, i.e., 512 MB), it sends a `SIGTERM` to trigger graceful shutdown before the database is corrupted.

### Node.Close() — The Full Teardown

When `stack.Close()` is called, the Node performs a complete teardown:

```go
func (n *Node) Close() error {
    n.startStopLock.Lock()
    defer n.startStopLock.Unlock()

    n.lock.Lock()
    state := n.state
    n.lock.Unlock()
    switch state {
    case initializingState:
        return n.doClose(nil)
    case runningState:
        var errs []error
        if err := n.stopServices(n.lifecycles); err != nil {
            errs = append(errs, err)
        }
        return n.doClose(errs)
    case closedState:
        return ErrNodeStopped
    }
    // ...
}
```

For a running node, `Close()` calls `stopServices()` followed by `doClose()`.

`stopServices()` tears down networking and lifecycles in reverse registration order:

```go
func (n *Node) stopServices(running []Lifecycle) error {
    n.stopRPC()

    failure := &StopError{Services: make(map[reflect.Type]error)}
    for i := len(running) - 1; i >= 0; i-- {
        if err := running[i].Stop(); err != nil {
            failure.Services[reflect.TypeOf(running[i])] = err
        }
    }
    n.server.Stop()
    // ...
}
```

- RPC endpoints are stopped first so no new requests arrive.
- Lifecycles are stopped in reverse order — services registered later (which may depend on earlier ones) are stopped first.
- The P2P server is stopped last, after all protocol handlers have shut down.

`doClose()` releases the remaining resources:

```go
func (n *Node) doClose(errs []error) error {
    n.lock.Lock()
    n.state = closedState
    errs = append(errs, n.closeDatabases()...)
    n.lock.Unlock()

    if err := n.accman.Close(); err != nil {
        errs = append(errs, err)
    }
    if n.keyDirTemp {
        if err := os.RemoveAll(n.keyDir); err != nil {
            errs = append(errs, err)
        }
    }
    n.closeDataDir()
    close(n.stop) // unblock Wait()
    // ...
}
```

- All tracked databases are closed.
- The account manager is closed (stops hardware wallet USB monitoring).
- Ephemeral key directories are removed.
- The data directory lock is released.
- `close(n.stop)` unblocks `stack.Wait()` in `geth()`, which causes the function to return and the process to exit.

---

## The Event System

Throughout previous chapters, we've seen events connecting subsystems — `ChainHeadEvent` triggers the miner, `TxPreEvent` triggers broadcast, `WalletEvent` triggers wallet management. The `event.Feed` in `event/feed.go` is the mechanism behind all of these.

### Feed

A `Feed` provides one-to-many event distribution. Subscribers register a channel; when `Send()` is called, the value is delivered to all subscriber channels simultaneously.

```go
// event/feed.go

type Feed struct {
    once      sync.Once
    sendLock  chan struct{}    // one-element buffer, empty when held
    removeSub chan interface{} // interrupts Send
    sendCases caseList         // active select cases

    mu    sync.Mutex
    inbox caseList
    etype reflect.Type
}
```

- **Type safety** — the `etype` field is set on first `Send()` or `Subscribe()`. All subsequent operations must use the same type or panic.
- **`sendCases`** — a slice of `reflect.SelectCase` entries, one per subscriber. `sendCases[0]` is always a receive case for `removeSub` (to handle unsubscriptions during a Send).
- **`inbox`** — new subscriptions are buffered here and merged into `sendCases` at the start of the next `Send()`.

### Subscribe

```go
func (f *Feed) Subscribe(channel interface{}) Subscription {
    chanval := reflect.ValueOf(channel)
    chantyp := chanval.Type()
    if chantyp.Kind() != reflect.Chan || chantyp.ChanDir()&reflect.SendDir == 0 {
        panic(errBadChannel)
    }
    sub := &feedSub{feed: f, channel: chanval, err: make(chan error, 1)}

    f.once.Do(func() { f.init(chantyp.Elem()) })
    // ...
    f.mu.Lock()
    defer f.mu.Unlock()
    cas := reflect.SelectCase{Dir: reflect.SelectSend, Chan: chanval}
    f.inbox = append(f.inbox, cas)
    return sub
}
```

The caller provides a channel (`chan SomeEvent`). Feed wraps it in a `reflect.SelectCase` and adds it to the inbox. The returned `Subscription` has an `Unsubscribe()` method and an `Err()` channel.

### Send

`Send()` delivers a value to all subscribers using a two-phase approach:

```go
func (f *Feed) Send(value interface{}) (nsent int) {
    rvalue := reflect.ValueOf(value)
    // ...
    <-f.sendLock

    // Merge inbox into sendCases
    f.mu.Lock()
    f.sendCases = append(f.sendCases, f.inbox...)
    f.inbox = nil
    f.mu.Unlock()

    // Set value on all channels
    for i := firstSubSendCase; i < len(f.sendCases); i++ {
        f.sendCases[i].Send = rvalue
    }

    cases := f.sendCases
    for {
        // Fast path: TrySend (non-blocking)
        for i := firstSubSendCase; i < len(cases); i++ {
            if cases[i].Chan.TrySend(rvalue) {
                nsent++
                cases = cases.deactivate(i)
                i--
            }
        }
        if len(cases) == firstSubSendCase {
            break
        }
        // Slow path: reflect.Select (blocking)
        chosen, recv, _ := reflect.Select(cases)
        if chosen == 0 { // removeSub channel
            // ... handle unsubscription ...
        } else {
            cases = cases.deactivate(chosen)
            nsent++
        }
    }
    // ...
}
```

The two phases:

1. **Fast path** — `TrySend` attempts a non-blocking send on each subscriber's channel. If the channel has buffer space, this succeeds immediately. Subscribers that receive are deactivated (moved to the end of the slice).
2. **Slow path** — for any remaining blocked subscribers, `reflect.Select` waits until at least one channel is ready. The `removeSub` channel (case 0) handles unsubscriptions that arrive while `Send` is blocked.

This design means slow subscribers block the sender (Feed does not drop events), which is why the documentation recommends ample buffer space on subscriber channels.

### Usage Pattern

A typical usage across geth subsystems looks like this:

```go
// Producer side (e.g., in blockchain.go)
type BlockChain struct {
    chainHeadFeed event.Feed
    // ...
}

func (bc *BlockChain) SubscribeChainHeadEvent(ch chan<- ChainHeadEvent) event.Subscription {
    return bc.chainHeadFeed.Subscribe(ch)
}

// Inside block insertion:
// bc.chainHeadFeed.Send(ChainHeadEvent{Block: block})
```

```go
// Consumer side (e.g., in miner or handler)
headCh := make(chan core.ChainHeadEvent, chainHeadChanSize)
sub := bc.SubscribeChainHeadEvent(headCh)
defer sub.Unsubscribe()

for {
    select {
    case head := <-headCh:
        // react to new chain head
    case err := <-sub.Err():
        return
    }
}
```

This pattern appears throughout geth — the blockchain publishes `ChainHeadEvent` and `ChainSideEvent`, the transaction pool publishes `TxPreEvent`, and the account manager publishes `WalletEvent`. The Feed provides the decoupling that lets these subsystems communicate without direct imports.

---

## The Complete Lifecycle — Start to Finish

Putting it all together, here is the full sequence from process start to exit:

```
  Process Start
  =============
  main()
    app.Before()       -- GOMAXPROCS, logging, debug setup
    geth()
      prepare()        -- log network, bump mainnet cache
      makeFullNode()
        loadBaseConfig()        -- defaults + TOML + CLI flags
        node.New()              -- Node container, data dir lock, P2P/RPC objects
        setAccountManagerBackends() -- keystore, hardware wallets
        eth.New()               -- chainDb, engine, blockchain, txpool,
                                   handler, miner, APIBackend
                                   Registers APIs + Protocols + Lifecycle
        catalyst.Register()     -- Engine API
      startNode()
        utils.StartNode()
          stack.Start()
            openEndpoints()     -- P2P server start, RPC start
            lifecycle.Start()   -- Ethereum.Start() -> discovery,
                                   handler, dropper, filterMaps
          signal handler goroutine installed
        wallet event listener goroutine started
      stack.Wait()             -- blocks on n.stop channel

  Signal Received (SIGINT / SIGTERM)
  ==================================
  signal handler:
    stack.Close()
      stopServices()
        stopRPC()              -- stop all RPC endpoints
        lifecycle.Stop()       -- Ethereum.Stop() in reverse order:
          discmix.Close()        peer discovery
          dropper.Stop()         connection manager
          handler.Stop()         sync, broadcast loops
          filterMaps.Stop()      log indexer
          txPool.Close()         transaction pool
          blockchain.Stop()      blockchain, trie flushing
          engine.Close()         consensus engine
          shutdownTracker.Stop() clean shutdown marker
          chainDb.Close()        close LevelDB/Pebble
          eventMux.Stop()        event multiplexer
        server.Stop()          -- P2P server
      doClose()
        closeDatabases()       -- any remaining open databases
        accman.Close()         -- account manager
        closeDataDir()         -- release file lock
        close(n.stop)          -- unblocks Wait()

  geth() returns -> main() returns -> process exits
```

The key invariant is **reverse-order teardown**: services registered last are stopped first, networking is stopped before internal processing, and the database is closed last. This ensures no component tries to read or write after its dependencies are gone.
