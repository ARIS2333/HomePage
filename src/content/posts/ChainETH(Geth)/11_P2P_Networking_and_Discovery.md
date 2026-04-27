---
title: Geth(11) P2P Networking and Discovery
published: 2026-04-20
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

Every subsystem covered so far — block insertion, state management, the EVM — operates on a single node. But Ethereum is a distributed system: nodes must find each other, establish encrypted connections, and exchange protocol messages. This chapter covers geth's networking stack, from discovering peers on the internet to multiplexing sub-protocol messages over encrypted TCP connections.

---

## The Networking Stack at a Glance

Geth's P2P layer is built from three distinct systems, each operating at a different level:

```
 +-------------------------------------------------------+
 |  Sub-protocols (eth/68, snap/1, ...)                  |  Application
 |    Each runs in its own goroutine per peer            |
 +-------------------------------------------------------+
 |  devp2p base protocol                                 |  Session
 |    Handshake, capability negotiation, ping/pong       |
 |    Message multiplexing across sub-protocols          |
 +-------------------------------------------------------+
 |  RLPx encrypted transport                             |  Transport
 |    ECIES handshake -> AES-CTR + Keccak-256 MAC        |
 |    Snappy compression, framed messages                |
 +-------------------------------------------------------+
 |  TCP connection                                       |  Network
 +-------------------------------------------------------+

 +-------------------------------------------------------+
 |  Node discovery (discv4 / discv5)                     |  UDP
 |    Kademlia DHT, ping/pong, findnode/neighbors        |
 +-------------------------------------------------------+
```

The lifecycle of a peer connection follows these steps:

1. **Discovery** — find nodes via UDP-based Kademlia DHT (discv4/v5)
2. **TCP dial** — connect to a candidate node's TCP port
3. **RLPx encryption handshake** — establish shared secrets via ECIES
4. **devp2p protocol handshake** — exchange capabilities (supported protocols)
5. **Protocol dispatch** — launch a goroutine per matched sub-protocol
6. **Message loop** — read, decrypt, route messages until disconnect

---

## The Server: Managing All Connections

The `Server` struct in `p2p/server.go` is the top-level P2P manager. It owns the TCP listener, discovery protocols, dial scheduler, and all active peer connections:

```go
// p2p/server.go

type Server struct {
    Config  // Embedded configuration

    lock    sync.Mutex
    running bool

    listener     net.Listener
    ourHandshake *protoHandshake
    loopWG       sync.WaitGroup
    peerFeed     event.Feed
    log          log.Logger

    nodedb    *enode.DB
    localnode *enode.LocalNode
    discv4    *discover.UDPv4
    discv5    *discover.UDPv5
    discmix   *enode.FairMix
    dialsched *dialScheduler

    portMappingRegister chan *portMapping

    quit                    chan struct{}
    addtrusted              chan *enode.Node
    removetrusted           chan *enode.Node
    peerOp                  chan peerOpFunc
    peerOpDone              chan struct{}
    delpeer                 chan peerDrop
    checkpointPostHandshake chan *conn
    checkpointAddPeer       chan *conn

    inboundHistory expHeap
}
```

Key fields:

- **`Config`** — embedded configuration (max peers, protocols, bootnodes, etc.).
- **`discv4` / `discv5`** — UDP-based node discovery protocols.
- **`discmix`** — a `FairMix` that merges discovery results from multiple sources (v4, v5, DNS, static nodes) into a single iterator for the dial scheduler.
- **`dialsched`** — decides which discovered nodes to dial and when.
- **`checkpointPostHandshake` / `checkpointAddPeer`** — channels that gate peer acceptance through the main loop, ensuring all peer-count checks happen in a single goroutine.

### Configuration

The `Config` struct controls all P2P behavior:

```go
// p2p/config.go

type Config struct {
    PrivateKey *ecdsa.PrivateKey

    MaxPeers        int           // Maximum total connections
    MaxPendingPeers int           // Max concurrent handshakes (default 50)
    DialRatio       int           // Ratio of inbound to dialed (default 3)

    NoDiscovery bool
    DiscoveryV4 bool
    DiscoveryV5 bool

    Name             string
    BootstrapNodes   []*enode.Node   // Discv4 bootstrap
    BootstrapNodesV5 []*enode.Node   // Discv5 bootstrap
    StaticNodes      []*enode.Node   // Always-connected peers
    TrustedNodes     []*enode.Node   // Peers that bypass limits

    NetRestrict    *netutil.Netlist  // IP whitelist
    NodeDatabase   string            // Path to node DB
    Protocols      []Protocol        // Supported sub-protocols

    ListenAddr string
    DiscAddr   string
    NAT        nat.Interface
    Dialer     NodeDialer
    NoDial     bool

    EnableMsgEvents bool
    Logger          log.Logger
    clock           mclock.Clock
}
```

The most important settings:

- **`MaxPeers`** — the hard cap on simultaneous peer connections (default 50 for mainnet). Trusted peers bypass this limit.
- **`DialRatio`** — controls the inbound/outbound split. With the default ratio of 3, at most `MaxPeers / 3` slots are used for outbound dials, and the rest accept inbound connections. This ensures the node is reachable by others.
- **`Protocols`** — the list of sub-protocols the node supports (e.g., `eth/68`, `snap/1`). Only peers that share at least one protocol are accepted.
- **`StaticNodes`** — peers that are always dialed and reconnected on disconnect. Used for operator-controlled peering.
- **`TrustedNodes`** — peers that are always accepted even when `MaxPeers` is reached.

### Connection Limits

The server splits its peer budget between dialed (outbound) and inbound connections:

```go
// p2p/server.go

func (srv *Server) MaxDialedConns() (limit int) {
    if srv.NoDial || srv.MaxPeers == 0 {
        return 0
    }
    if srv.DialRatio == 0 {
        limit = srv.MaxPeers / defaultDialRatio  // default: MaxPeers / 3
    } else {
        limit = srv.MaxPeers / srv.DialRatio
    }
    if limit == 0 {
        limit = 1
    }
    return limit
}

func (srv *Server) MaxInboundConns() int {
    return srv.MaxPeers - srv.MaxDialedConns()
}
```

With 50 max peers and ratio 3: up to 16 dialed, up to 34 inbound.

---

## Server Startup

`Start()` initializes every component in sequence:

```go
// p2p/server.go  (simplified)

func (srv *Server) Start() error {
    srv.lock.Lock()
    defer srv.lock.Unlock()
    if srv.running {
        return errors.New("server already running")
    }
    srv.running = true

    // Initialize channels
    srv.quit = make(chan struct{})
    srv.delpeer = make(chan peerDrop)
    srv.checkpointPostHandshake = make(chan *conn)
    srv.checkpointAddPeer = make(chan *conn)
    // ...

    // 1. Set up local node identity and devp2p handshake
    srv.setupLocalNode()

    // 2. Register NAT port mappings
    srv.setupPortMapping()

    // 3. Start TCP listener for inbound connections
    if srv.ListenAddr != "" {
        srv.setupListening()
    }

    // 4. Start UDP discovery (v4 and/or v5)
    srv.setupDiscovery()

    // 5. Start the dial scheduler
    srv.setupDialScheduler()

    // 6. Launch the main event loop
    go srv.run()
    return nil
}
```

The dial scheduler receives candidate nodes from `discmix` (which combines all discovery sources) and decides which to dial based on available slots:

```go
// p2p/server.go

func (srv *Server) setupDialScheduler() {
    // ...
    srv.dialsched = newDialScheduler(config, srv.discmix, srv.SetupConn)
    for _, n := range srv.StaticNodes {
        srv.dialsched.addStatic(n)
    }
}
```

Static nodes are added immediately. The scheduler will continuously attempt to connect to them.

---

## The Main Loop

The `run()` method is the server's single-threaded event loop. All peer-set mutations happen here, eliminating the need for complex locking:

```go
// p2p/server.go  (simplified)

func (srv *Server) run() {
    var (
        peers        = make(map[enode.ID]*Peer)
        inboundCount = 0
        trusted      = make(map[enode.ID]bool, len(srv.TrustedNodes))
    )
    for _, n := range srv.TrustedNodes {
        trusted[n.ID()] = true
    }

    for {
        select {
        case <-srv.quit:
            // Disconnect all peers, wait for them to shut down
            for _, p := range peers {
                p.Disconnect(DiscQuitting)
            }
            // ...
            return

        case n := <-srv.addtrusted:
            trusted[n.ID()] = true

        case n := <-srv.removetrusted:
            delete(trusted, n.ID())

        case op := <-srv.peerOp:
            op(peers) // Used by Peers(), PeerCount()
            srv.peerOpDone <- struct{}{}

        case c := <-srv.checkpointPostHandshake:
            if trusted[c.node.ID()] {
                c.flags |= trustedConn
            }
            c.cont <- srv.postHandshakeChecks(peers, inboundCount, c)

        case c := <-srv.checkpointAddPeer:
            err := srv.addPeerChecks(peers, inboundCount, c)
            if err == nil {
                p := srv.launchPeer(c)
                peers[c.node.ID()] = p
                srv.dialsched.peerAdded(c)
                if p.Inbound() {
                    inboundCount++
                }
            }
            c.cont <- err

        case pd := <-srv.delpeer:
            delete(peers, pd.ID())
            srv.dialsched.peerRemoved(pd.rw)
            if pd.Inbound() {
                inboundCount--
            }
        }
    }
}
```

The loop handles six types of events:

1. **Shutdown** — disconnect all peers, close discovery, wait for cleanup.
2. **Trust management** — `addtrusted` / `removetrusted` modify the trusted set. A peer already connected can be upgraded to trusted status.
3. **Peer queries** — `peerOp` allows `Peers()` and `PeerCount()` to safely read the peer map.
4. **Post-handshake checkpoint** — after the RLPx encryption handshake, the connection is checked against `MaxPeers`, `MaxInboundConns`, duplicate connections, and self-connections.
5. **Add-peer checkpoint** — after the protocol handshake, checks that at least one sub-protocol matches, then launches the peer.
6. **Peer removal** — updates peer count and notifies the dial scheduler so it can fill the vacant slot.

### Peer Validation

Two validation steps gate every new connection:

```go
// p2p/server.go

func (srv *Server) postHandshakeChecks(peers map[enode.ID]*Peer, inboundCount int, c *conn) error {
    switch {
    case !c.is(trustedConn) && len(peers) >= srv.MaxPeers:
        return DiscTooManyPeers
    case !c.is(trustedConn) && c.is(inboundConn) && inboundCount >= srv.MaxInboundConns():
        return DiscTooManyPeers
    case peers[c.node.ID()] != nil:
        return DiscAlreadyConnected
    case c.node.ID() == srv.localnode.ID():
        return DiscSelf
    default:
        return nil
    }
}

func (srv *Server) addPeerChecks(peers map[enode.ID]*Peer, inboundCount int, c *conn) error {
    if len(srv.Protocols) > 0 && countMatchingProtocols(srv.Protocols, c.caps) == 0 {
        return DiscUselessPeer
    }
    return srv.postHandshakeChecks(peers, inboundCount, c)
}
```

Post-handshake checks run after the encryption handshake reveals the remote identity. Add-peer checks run after the protocol handshake reveals capabilities. The post-handshake checks are repeated in `addPeerChecks` because the peer set may have changed between the two checkpoints.

---

## Connection Establishment

When a TCP connection is established (either outbound dial or inbound accept), `SetupConn()` runs the two-phase handshake:

```go
// p2p/server.go

func (srv *Server) SetupConn(fd net.Conn, flags connFlag, dialDest *enode.Node) error {
    c := &conn{fd: fd, flags: flags, cont: make(chan error)}
    if dialDest == nil {
        c.transport = srv.newTransport(fd, nil)           // inbound: no remote key yet
    } else {
        c.transport = srv.newTransport(fd, dialDest.Pubkey()) // outbound: know remote key
    }
    err := srv.setupConn(c, dialDest)
    if err != nil {
        c.close(err)
    }
    return err
}
```

The internal `setupConn()` runs both handshakes:

```go
// p2p/server.go  (simplified)

func (srv *Server) setupConn(c *conn, dialDest *enode.Node) error {
    // Phase 1: RLPx encryption handshake
    remotePubkey, err := c.doEncHandshake(srv.PrivateKey)
    if err != nil {
        return fmt.Errorf("%w: %v", errEncHandshakeError, err)
    }
    if dialDest != nil {
        c.node = dialDest
    } else {
        c.node = nodeFromConn(remotePubkey, c.fd)
    }
    // Checkpoint: validate against peer limits
    err = srv.checkpoint(c, srv.checkpointPostHandshake)
    if err != nil {
        return err
    }

    // Phase 2: devp2p protocol handshake
    phs, err := c.doProtoHandshake(srv.ourHandshake)
    if err != nil {
        return &protoHandshakeError{err: err}
    }
    // Verify identity: Keccak256(pubkey) must match node ID
    if id := c.node.ID(); !bytes.Equal(crypto.Keccak256(phs.ID), id[:]) {
        return DiscUnexpectedIdentity
    }
    c.caps, c.name = phs.Caps, phs.Name

    // Checkpoint: validate protocol match
    err = srv.checkpoint(c, srv.checkpointAddPeer)
    return err
}
```

The `checkpoint()` calls send the `conn` to the main loop via a channel and block until the main loop responds on `c.cont`. This ensures all peer-count checks are serialized in the single-threaded `run()` loop.

### Connection Types

Every connection is tagged with flags indicating how it was established:

```go
// p2p/server.go

const (
    dynDialedConn    connFlag = 1 << iota // Discovered via DHT, dialed dynamically
    staticDialedConn                       // Static node, always reconnected
    inboundConn                            // Remote peer initiated the connection
    trustedConn                            // Bypasses MaxPeers limit
)
```

A connection can have multiple flags — for example, a static node that is also trusted.

---

## RLPx: The Encrypted Transport

Every TCP connection is wrapped in the **RLPx** transport (`p2p/rlpx/rlpx.go`), which provides authenticated encryption. The protocol has two phases: an ECIES handshake that establishes shared secrets, and a framed message protocol that encrypts all subsequent traffic.

### The Handshake

The RLPx handshake (defined in EIP-8) uses Elliptic Curve Integrated Encryption Scheme (ECIES) to establish a shared secret:

```
 Initiator (dialer)                     Responder (listener)
       |                                        |
       |  auth message (ECIES-encrypted)        |
       |  { signature, initiator-pubkey,        |
       |    nonce, version }                    |
       | -------------------------------------> |
       |                                        |  Decrypt, verify signature
       |  auth-ack message (ECIES-encrypted)    |
       |  { responder-ephemeral-pubkey,         |
       |    nonce, version }                    |
       | <------------------------------------- |
       |                                        |
       |  Both sides derive shared secrets      |
       |  from ECDHE key agreement              |
```

The handshake state tracks the ephemeral keys and nonces:

```go
// p2p/rlpx/rlpx.go

type handshakeState struct {
    initiator            bool
    remote               *ecies.PublicKey
    initNonce, respNonce []byte
    randomPrivKey        *ecies.PrivateKey   // ephemeral ECDHE key
    remoteRandomPub      *ecies.PublicKey     // remote ephemeral key
    // ...
}
```

Each side generates a random ephemeral key pair for ECDHE (Elliptic Curve Diffie-Hellman Ephemeral). The auth message contains:

```go
// p2p/rlpx/rlpx.go

type authMsgV4 struct {
    Signature       [65]byte   // ECDSA signature
    InitiatorPubkey [64]byte   // Static public key
    Nonce           [32]byte   // Random nonce
    Version         uint
    Rest []rlp.RawValue `rlp:"tail"` // Forward-compatibility
}
```

After both messages are exchanged, shared secrets are derived:

```go
// p2p/rlpx/rlpx.go

func (h *handshakeState) secrets(auth, authResp []byte) (Secrets, error) {
    ecdheSecret, err := h.randomPrivKey.GenerateShared(h.remoteRandomPub, sskLen, sskLen)

    sharedSecret := crypto.Keccak256(ecdheSecret, crypto.Keccak256(h.respNonce, h.initNonce))
    aesSecret := crypto.Keccak256(ecdheSecret, sharedSecret)
    s := Secrets{
        remote: h.remote.ExportECDSA(),
        AES:    aesSecret,
        MAC:    crypto.Keccak256(ecdheSecret, aesSecret),
    }

    // Setup MAC states (direction depends on initiator/responder role)
    mac1 := sha3.NewLegacyKeccak256()
    mac1.Write(xor(s.MAC, h.respNonce))
    mac1.Write(auth)
    mac2 := sha3.NewLegacyKeccak256()
    mac2.Write(xor(s.MAC, h.initNonce))
    mac2.Write(authResp)

    if h.initiator {
        s.EgressMAC, s.IngressMAC = mac1, mac2
    } else {
        s.EgressMAC, s.IngressMAC = mac2, mac1
    }
    return s, nil
}
```

The derivation chain is: `ecdheSecret` → `sharedSecret` → `aesSecret` → `MAC key`. Two separate Keccak-256 MAC states are initialized for each direction, seeded with the nonces and the raw handshake packets. This means each direction has its own MAC chain, and the handshake messages themselves are mixed into the MAC state for authentication.

### Frame Encryption

After the handshake, all messages are sent as encrypted frames:

```go
// p2p/rlpx/rlpx.go

type Conn struct {
    dialDest *ecdsa.PublicKey
    conn     net.Conn
    session  *sessionState

    snappyReadBuffer  []byte
    snappyWriteBuffer []byte
}

type sessionState struct {
    enc        cipher.Stream    // AES-256-CTR for encryption
    dec        cipher.Stream    // AES-256-CTR for decryption
    egressMAC  hashMAC          // Outgoing MAC state (Keccak-256 + AES-128)
    ingressMAC hashMAC          // Incoming MAC state
    rbuf       readBuffer
    wbuf       writeBuffer
}
```

Each frame has this wire format:

```
 Header (32 bytes):
   [frame-size: 3 bytes] [header-data: 13 bytes]  ← AES-CTR encrypted
   [header-MAC: 16 bytes]                          ← Keccak-256 MAC

 Body (variable):
   [frame-data: padded to 16-byte boundary]        ← AES-CTR encrypted
   [frame-MAC: 16 bytes]                           ← Keccak-256 MAC
```

The body contains the RLP-encoded message code followed by the payload. If Snappy compression is enabled (which it is after the protocol handshake), the payload is compressed before encryption.

---

## The devp2p Base Protocol

On top of the encrypted RLPx transport, the **devp2p base protocol** provides session management. It reserves message codes 0–15 for its own use:

```go
// p2p/message.go

const (
    baseProtocolVersion    = 5
    baseProtocolLength     = 16    // Codes 0-15 reserved

    handshakeMsg = 0x00
    discMsg      = 0x01
    pingMsg      = 0x02
    pongMsg      = 0x03
)
```

The **protocol handshake** (`handshakeMsg`) is the first message exchanged after encryption. It carries the node's identity and supported capabilities:

```go
// p2p/peer.go (protoHandshake type defined in message.go)

type protoHandshake struct {
    Version    uint64
    Name       string    // e.g. "Geth/v1.16.7-stable/linux-amd64/go1.23.0"
    Caps       []Cap     // e.g. [{eth 68}, {snap 1}]
    ListenPort uint64
    ID         []byte    // secp256k1 public key (64 bytes)
}
```

After exchanging handshakes, both sides know each other's capabilities. Only protocols that both sides support are activated.

---

## The Peer: Message Multiplexing

Once the handshakes complete and validation passes, the server creates a `Peer` and launches its run loop:

```go
// p2p/peer.go

type Peer struct {
    rw      *conn
    running map[string]*protoRW  // Active sub-protocols by name
    log     log.Logger
    created mclock.AbsTime

    wg       sync.WaitGroup
    protoErr chan error
    closed   chan struct{}
    pingRecv chan struct{}
    disc     chan DiscReason

    events   *event.Feed
    // ...
}
```

### Protocol Matching

Before launching the peer, `matchProtocols()` aligns local and remote capabilities:

```go
// p2p/peer.go

func matchProtocols(protocols []Protocol, caps []Cap, rw MsgReadWriter) map[string]*protoRW {
    slices.SortFunc(caps, Cap.Cmp)
    offset := baseProtocolLength  // Start at 16 (after base protocol codes)
    result := make(map[string]*protoRW)

    for _, cap := range caps {
        for _, proto := range protocols {
            if proto.Name == cap.Name && proto.Version == cap.Version {
                if old := result[cap.Name]; old != nil {
                    offset -= old.Length  // Replace older version
                }
                result[cap.Name] = &protoRW{
                    Protocol: proto,
                    offset:   offset,
                    in:       make(chan Msg),
                    w:        rw,
                }
                offset += proto.Length
                continue
            }
        }
    }
    return result
}
```

Each matched protocol gets a contiguous range of message codes starting from offset 16. For example, if `eth/68` uses 17 message codes and `snap/1` uses 8, then `eth` gets codes 16–32 and `snap` gets 33–40. If both sides support multiple versions of the same protocol, only the highest matching version is used.

### The Peer Run Loop

`Peer.run()` coordinates all goroutines for a single peer connection:

```go
// p2p/peer.go  (simplified)

func (p *Peer) run() (remoteRequested bool, err error) {
    var (
        writeStart = make(chan struct{}, 1)
        writeErr   = make(chan error, 1)
        readErr    = make(chan error, 1)
    )
    p.wg.Add(2)
    go p.readLoop(readErr)
    go p.pingLoop()

    // Allow the first write
    writeStart <- struct{}{}
    p.startProtocols(writeStart, writeErr)

    // Wait for any error
    for {
        select {
        case err = <-writeErr:
            if err != nil {
                break // network error
            }
            writeStart <- struct{}{} // allow next write
        case err = <-readErr:
            break // read error or remote disconnect
        case err = <-p.protoErr:
            break // protocol handler error
        case err = <-p.disc:
            break // local disconnect request
        }
    }

    close(p.closed)
    p.rw.close(reason)
    p.wg.Wait()
    return remoteRequested, err
}
```

Three types of goroutines run concurrently:

1. **`readLoop()`** — reads messages from the encrypted connection and dispatches them.
2. **`pingLoop()`** — sends a PING every 15 seconds and responds to incoming PINGs with PONGs.
3. **One goroutine per protocol** — each sub-protocol's `Run()` function executes in its own goroutine.

The **write serialization** mechanism is key: `writeStart` is a channel with capacity 1 that acts as a token. Only one goroutine can write at a time. After a write completes, the token is returned to `writeStart` so the next goroutine can write. This prevents message interleaving on the wire without requiring a mutex.

### Message Dispatch

The `readLoop()` reads raw messages and routes them:

```go
// p2p/peer.go

func (p *Peer) readLoop(errc chan<- error) {
    defer p.wg.Done()
    for {
        msg, err := p.rw.ReadMsg()
        if err != nil {
            errc <- err
            return
        }
        msg.ReceivedAt = time.Now()
        if err = p.handle(msg); err != nil {
            errc <- err
            return
        }
    }
}

func (p *Peer) handle(msg Msg) error {
    switch {
    case msg.Code == pingMsg:
        msg.Discard()
        p.pingRecv <- struct{}{}
    case msg.Code == discMsg:
        return decodeDisconnectMessage(msg.Payload)
    case msg.Code < baseProtocolLength:
        return msg.Discard()  // ignore other base protocol msgs
    default:
        // Sub-protocol message: find the owning protocol by code range
        proto, err := p.getProto(msg.Code)
        if err != nil {
            return fmt.Errorf("msg code out of range: %v", msg.Code)
        }
        proto.in <- msg  // deliver to protocol handler
    }
    return nil
}
```

Base protocol messages (codes 0–15) are handled directly. Sub-protocol messages are routed to the appropriate `protoRW.in` channel based on which protocol's code range the message code falls in.

### The protoRW Wrapper

Each sub-protocol handler reads and writes through a `protoRW` that transparently translates between protocol-local message codes and wire codes:

```go
// p2p/peer.go

type protoRW struct {
    Protocol
    in     chan Msg        // receives read messages
    closed <-chan struct{} // peer shutdown signal
    wstart <-chan struct{} // write serialization token
    werr   chan<- error    // write result
    offset uint64          // base message code offset
    w      MsgWriter
}

func (rw *protoRW) WriteMsg(msg Msg) error {
    if msg.Code >= rw.Length {
        return newPeerError(errInvalidMsgCode, "not handled")
    }
    msg.Code += rw.offset        // translate to wire code

    select {
    case <-rw.wstart:            // wait for write token
        err := rw.w.WriteMsg(msg)
        rw.werr <- err           // report result
        return err
    case <-rw.closed:
        return ErrShuttingDown
    }
}

func (rw *protoRW) ReadMsg() (Msg, error) {
    select {
    case msg := <-rw.in:
        msg.Code -= rw.offset    // translate from wire code
        return msg, nil
    case <-rw.closed:
        return Msg{}, io.EOF
    }
}
```

The protocol handler sees clean message codes starting from 0, without knowing about the wire-level offset. This allows protocols to be composed without code conflicts.

---

## The Protocol Struct

Sub-protocols are registered with the server via the `Protocol` struct:

```go
// p2p/protocol.go

type Protocol struct {
    Name    string    // Protocol name (e.g. "eth")
    Version uint      // Protocol version (e.g. 68)
    Length  uint64    // Number of message codes used

    Run func(peer *Peer, rw MsgReadWriter) error  // Handler function

    NodeInfo func() interface{}                    // Optional: local node info
    PeerInfo func(id enode.ID) interface{}          // Optional: per-peer info

    DialCandidates enode.Iterator                  // Optional: protocol-specific discovery
    Attributes     []enr.Entry                     // Optional: ENR attributes
}
```

The `Run` function is called in a new goroutine when the protocol is negotiated with a peer. It should read and write messages via `rw` until the connection closes. The Ethereum wire protocol (`eth/68`) and the snap sync protocol (`snap/1`) are both registered as `Protocol` instances — their implementation is covered in Chapter 12.

Capabilities are advertised during the protocol handshake as name/version pairs:

```go
// p2p/protocol.go

type Cap struct {
    Name    string
    Version uint
}
```

---

## Node Discovery

Before geth can connect to peers, it must find them. Discovery runs over UDP, separate from the TCP-based RLPx connections, using a Kademlia-like distributed hash table.

### Node Identity

Every node is identified by a `Node` struct:

```go
// p2p/enode/node.go

type Node struct {
    r        enr.Record     // Signed Ethereum Node Record
    id       ID             // 32-byte Keccak-256 of secp256k1 public key
    hostname string         // Optional DNS name

    ip  netip.Addr          // Chosen IP address
    udp uint16              // UDP port (discovery)
    tcp uint16              // TCP port (RLPx)
}
```

The `ID` type is a 32-byte hash (`[32]byte`), computed as `Keccak256(secp256k1_pubkey)`. Nodes are addressable via **enode URLs**:

```
enode://<hex-pubkey>@<ip>:<tcp-port>
```

For example:
```
enode://d860a01f...db1f666@18.138.108.67:30303
```

The 128-character hex string is the uncompressed secp256k1 public key (64 bytes). The Ethereum Node Record (ENR), defined in EIP-778, provides a more extensible format: a signed, versioned key-value record that can carry IP addresses, ports, and protocol-specific attributes.

### Discovery v4: Kademlia DHT

The primary discovery protocol (`p2p/discover/v4_udp.go`) implements a simplified Kademlia DHT over UDP:

```go
// p2p/discover/v4_udp.go

type UDPv4 struct {
    conn        UDPConn
    priv        *ecdsa.PrivateKey
    localNode   *enode.LocalNode
    db          *enode.DB
    tab         *Table                  // Kademlia routing table
    // ...
    addReplyMatcher chan *replyMatcher
    gotreply        chan reply
}
```

Four packet types form the core protocol:

```go
// p2p/discover/v4wire/v4wire.go

const (
    PingPacket      = iota + 1  // 1
    PongPacket                   // 2
    FindnodePacket               // 3
    NeighborsPacket              // 4
    ENRRequestPacket             // 5 (EIP-868)
    ENRResponsePacket            // 6 (EIP-868)
)
```

| Packet | Purpose |
|--------|---------|
| **PING** | Verify a node is alive. Contains sender/recipient endpoints and an ENR sequence number. |
| **PONG** | Reply to PING. Echoes the sender's observed IP (used for NAT discovery). |
| **FINDNODE** | Request nodes closest to a target public key. |
| **NEIGHBORS** | Reply to FINDNODE. Contains up to 12 node records. |
| **ENRRequest/ENRResponse** | Request/return a node's full ENR record (EIP-868 extension). |

Each discovery packet has this wire format:

```
 [32 bytes]  MAC        = Keccak-256(signature || packet-type || packet-data)
 [65 bytes]  Signature  = ECDSA signature over (packet-type || packet-data)
 [1 byte]    Packet type
 [variable]  RLP-encoded packet data
```

The MAC provides integrity checking. The ECDSA signature authenticates the sender and allows the recipient to recover the sender's public key (and thus node ID) without a prior key exchange.

### The Kademlia Routing Table

The routing table (`p2p/discover/table.go`) organizes known nodes by their distance from the local node:

```go
// p2p/discover/table.go

const (
    alpha           = 3     // Concurrency factor
    bucketSize      = 16    // Max nodes per bucket
    maxReplacements = 10    // Replacement candidates per bucket
    hashBits        = 256   // Bits in node ID
    nBuckets        = 17    // hashBits / 15
)

type Table struct {
    mutex   sync.Mutex
    buckets [nBuckets]*bucket    // Indexed by log-distance
    nursery []*enode.Node        // Bootstrap nodes
    rand    reseedingRandom
    ips     netutil.DistinctNetSet
    db      *enode.DB
    net     transport
    // ...
}

type bucket struct {
    entries      []*tableNode      // Live entries, MRU first
    replacements []*tableNode      // Replacement candidates
    ips          netutil.DistinctNetSet
    index        int
}
```

Distance is measured as the XOR of two node IDs, expressed as `log2(XOR(a, b))`. Nodes that are "closer" (in XOR space) to the local node fill lower-numbered buckets. Each bucket holds up to 16 nodes plus 10 replacement candidates.

IP address limits prevent Sybil attacks: at most 2 nodes from the same /24 subnet per bucket, and at most 10 per the entire table.

### The Lookup Algorithm

A **lookup** finds the nodes closest to a target key. It is the core operation of the Kademlia DHT:

1. Pick the `alpha` (3) closest known nodes to the target from the routing table.
2. Send FINDNODE to those nodes concurrently.
3. Each FINDNODE returns up to 12 neighbors.
4. Add the new neighbors to a result set, sorted by distance to target.
5. Pick the next `alpha` closest nodes that haven't been queried yet.
6. Repeat until no closer nodes are found.

The lookup converges because each round discovers nodes that are progressively closer to the target in XOR space.

### Bootstrapping

On first startup, the routing table is empty. Geth bootstraps by contacting **bootnodes** — hardcoded, well-known nodes run by the Ethereum Foundation and other organizations:

```go
// params/bootnodes.go

var MainnetBootnodes = []string{
    "enode://d860a01f...db1f666@18.138.108.67:30303",
    "enode://22a8232c...a68d4de@3.209.45.79:30303",
    "enode://2b252ab6...e6ffc@65.108.70.101:30303",
    "enode://4aeb4ab6...82052@157.90.35.166:30303",
}
```

The bootnodes are added to the table's `nursery`. When the table needs to be populated (on startup, or when too many nodes go offline), a `refresh()` operation performs a lookup for the local node's own ID — this fills the routing table with nodes from all distance ranges.

The node database (`enode.DB`, stored on disk) persists known nodes across restarts, so a node that has been running before does not need to re-bootstrap from scratch.

### Discovery v5

Discovery v5 (`p2p/discover/v5_udp.go`) is a newer protocol that adds:

- **Session-based encryption** — after an initial WHOAREYOU challenge, messages are encrypted with session keys, unlike v4 where every packet carries a full ECDSA signature.
- **ENR as first-class records** — nodes advertise their capabilities via ENR attributes, enabling protocol-specific peer filtering.
- **Topic advertisement** — nodes can register interest in topics, allowing targeted peer discovery (e.g., finding light client servers).

Both v4 and v5 can run simultaneously. The `FairMix` in the server merges results from both into a single stream of candidate nodes for the dial scheduler.

---

## NAT Traversal

Nodes behind Network Address Translation (NAT) need help making their listening port reachable. The `nat.Interface` in `p2p/nat/nat.go` provides an abstraction:

```go
// p2p/nat/nat.go

type Interface interface {
    AddMapping(protocol string, extport, intport int,
               name string, lifetime time.Duration) (uint16, error)
    DeleteMapping(protocol string, extport, intport int) error
    ExternalIP() (net.IP, error)
    String() string
}
```

Supported mechanisms:

| Mechanism | Description |
|-----------|-------------|
| `"none"` | No NAT traversal |
| `"extip:IP"` | Assume reachable on the given external IP |
| `"upnp"` | Universal Plug and Play — queries the router for port mappings |
| `"pmp"` | NAT-PMP — a simpler alternative to UPnP, common on Apple routers |
| `"stun"` | STUN — discovers external IP via a STUN server |
| `"any"` | Auto-detect the first available mechanism |

During startup, the server calls `setupPortMapping()` which registers both TCP (RLPx) and UDP (discovery) ports with the NAT gateway. The mapping is periodically refreshed to keep it alive.

Additionally, the PONG response in discovery v4 echoes the sender's observed IP address, providing a second mechanism for a node to discover its external address.

---

## Inbound Connection Handling

The `listenLoop()` accepts incoming TCP connections:

```go
// p2p/server.go  (simplified)

func (srv *Server) listenLoop() {
    // Limit concurrent handshakes
    tokens := defaultMaxPendingPeers  // 50
    if srv.MaxPendingPeers > 0 {
        tokens = srv.MaxPendingPeers
    }
    slots := make(chan struct{}, tokens)
    for i := 0; i < tokens; i++ {
        slots <- struct{}{}
    }

    for {
        <-slots  // wait for a free handshake slot
        fd, err := srv.listener.Accept()
        // ...
        go func() {
            defer func() { slots <- struct{}{} }()
            srv.SetupConn(fd, inboundConn, nil)
        }()
    }
}
```

Each accepted connection consumes a handshake slot. The slot is returned after the handshake completes (or fails), ensuring at most `MaxPendingPeers` concurrent handshakes. Inbound connections are also rate-limited by source IP — the `inboundHistory` expiration heap ensures at most one connection per IP per 30 seconds, defending against connection-flood attacks.

---

## Putting It All Together

Here is the complete lifecycle of a peer connection:

1. **Discovery** fills the routing table with candidate nodes via UDP PING/PONG and FINDNODE/NEIGHBORS exchanges.
2. **The dial scheduler** picks candidates from the `FairMix` iterator and dials their TCP port. Static nodes are always dialed; dynamic nodes are dialed to fill remaining slots.
3. **RLPx encryption handshake** establishes shared AES and MAC keys via ECIES. Both sides now know each other's static public key.
4. **The server's main loop** validates the connection at the post-handshake checkpoint: checks `MaxPeers`, duplicate connections, and self-connections.
5. **devp2p protocol handshake** exchanges capabilities. The server's main loop validates at the add-peer checkpoint: ensures at least one matching sub-protocol.
6. **`launchPeer()`** creates a `Peer`, matches protocols, and starts goroutines: one for reading, one for pings, and one per matched sub-protocol.
7. **Message flow**: the `readLoop` decrypts and decompresses messages, then routes them — base protocol messages (ping/pong/disconnect) are handled directly, sub-protocol messages are dispatched to the appropriate `protoRW.in` channel.
8. **Disconnection** can be triggered by: a network error, the remote peer sending a `discMsg`, a protocol handler returning an error, or a local shutdown. The `Peer.run()` loop closes the connection, waits for all goroutines, and sends a `peerDrop` to the server's main loop, which removes the peer and notifies the dial scheduler.