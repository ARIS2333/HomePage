---
title: Geth(2) Core Data Types
published: 2026-04-11
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

This chapter introduces the "nouns" of the system — the structs that flow through every subsystem from the transaction pool to the blockchain database. Before you can follow a transaction through execution, state, or storage, you need to know what a transaction *is* at the struct level, and how it relates to blocks, receipts, and logs.

Here is a high-level map of the types and how they relate:

```
┌─────────────────────────────────────────────────────────────────┐
│  Block                                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Header                                                  │   │
│  │  ParentHash, StateRoot, TxHash, ReceiptHash, Number,     │   │
│  │  GasLimit, GasUsed, Time, BaseFee, BlobGasUsed, ...      │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Body                                                    │   │
│  │  ┌────────────────────────────────────┐                  │   │
│  │  │  Transactions (1 per tx in block)  │                  │   │
│  │  │  ┌────────────────────────────┐    │                  │   │
│  │  │  │  Transaction (envelope)    │    │                  │   │
│  │  │  │  inner: LegacyTx           │    │                  │   │
│  │  │  │       | AccessListTx       │    │                  │   │
│  │  │  │       | DynamicFeeTx       │    │                  │   │
│  │  │  │       | BlobTx             │    │                  │   │
│  │  │  │       | SetCodeTx          │    │                  │   │
│  │  │  └────────────────────────────┘    │                  │   │
│  │  └────────────────────────────────────┘                  │   │
│  │  Uncles       []*Header  (legacy, always empty post-Merge)│  │
│  │  Withdrawals  []*Withdrawal  (post-Shanghai)             │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Execution produces (not stored in block, but linked via hash): │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Receipts (1 per tx)                                     │   │
│  │  ┌──────────────────────┐                                │   │
│  │  │  Receipt             │                                │   │
│  │  │  Status, GasUsed,    │                                │   │
│  │  │  Logs, Bloom         │                                │   │
│  │  └──────────────────────┘                                │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

All types live in `core/types/`. This chapter walks through each one.

---

## 1. The Block Header — `types.Header`

The header is the most important struct in the system. It commits to the entire state of the chain at a given point: the parent block, the world state, the transactions, and the execution results. Every block hash is the Keccak256 of the RLP-encoded header.

### The Header struct

The `Header` struct has 15 original fields and 6 optional fields added by post-London EIPs:

```go
// core/types/block.go

type Header struct {
    ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`
    UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
    Coinbase    common.Address `json:"miner"`
    Root        common.Hash    `json:"stateRoot"        gencodec:"required"`
    TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`
    ReceiptHash common.Hash    `json:"receiptsRoot"     gencodec:"required"`
    Bloom       Bloom          `json:"logsBloom"        gencodec:"required"`
    Difficulty  *big.Int       `json:"difficulty"       gencodec:"required"`
    Number      *big.Int       `json:"number"           gencodec:"required"`
    GasLimit    uint64         `json:"gasLimit"         gencodec:"required"`
    GasUsed     uint64         `json:"gasUsed"          gencodec:"required"`
    Time        uint64         `json:"timestamp"        gencodec:"required"`
    Extra       []byte         `json:"extraData"        gencodec:"required"`
    MixDigest   common.Hash    `json:"mixHash"`
    Nonce       BlockNonce     `json:"nonce"`

    // BaseFee was added by EIP-1559 and is ignored in legacy headers.
    BaseFee *big.Int `json:"baseFeePerGas" rlp:"optional"`

    // WithdrawalsHash was added by EIP-4895 and is ignored in legacy headers.
    WithdrawalsHash *common.Hash `json:"withdrawalsRoot" rlp:"optional"`

    // BlobGasUsed was added by EIP-4844 and is ignored in legacy headers.
    BlobGasUsed *uint64 `json:"blobGasUsed" rlp:"optional"`

    // ExcessBlobGas was added by EIP-4844 and is ignored in legacy headers.
    ExcessBlobGas *uint64 `json:"excessBlobGas" rlp:"optional"`

    // ParentBeaconRoot was added by EIP-4788 and is ignored in legacy headers.
    ParentBeaconRoot *common.Hash `json:"parentBeaconBlockRoot" rlp:"optional"`

    // RequestsHash was added by EIP-7685 and is ignored in legacy headers.
    RequestsHash *common.Hash `json:"requestsHash" rlp:"optional"`
}
```

The fields break into four groups:

**Chain linkage:**
- `ParentHash` — hash of the parent block's header. This is the single link that forms the chain.
- `UncleHash` — hash of the uncle list. Post-Merge, this is always the hash of an empty list.
- `Number` — block number (height from genesis).
- `Time` — Unix timestamp of the block.

**Execution commitment:**
- `Root` — the state root: the Merkle Patricia Trie root hash of the world state *after* executing all transactions in this block. This is how the header commits to the entire account/storage state.
- `TxHash` — root hash of the trie built from this block's transactions.
- `ReceiptHash` — root hash of the trie built from this block's receipts.
- `Bloom` — a 256-byte (2048-bit) bloom filter aggregating all log entries from all receipts. Allows fast topic/address filtering without scanning every receipt.

**Gas and fees:**
- `GasLimit` — maximum gas allowed in this block.
- `GasUsed` — total gas consumed by all transactions.
- `BaseFee` (EIP-1559) — the base fee per gas for this block. Burned, not paid to the miner.
- `BlobGasUsed` (EIP-4844) — total blob gas consumed by blob transactions.
- `ExcessBlobGas` (EIP-4844) — running excess blob gas, used to calculate the blob base fee for the next block.

**Consensus / builder:**
- `Coinbase` — address of the block builder (receives priority fees).
- `Difficulty` — always zero post-Merge (was used in proof-of-work).
- `MixDigest` — post-Merge, this carries the RANDAO beacon value from the consensus layer. Pre-Merge, it was part of the PoW proof.
- `Nonce` — always zero post-Merge (was part of the PoW proof). Typed as `BlockNonce`, which is a `[8]byte` array.
- `Extra` — up to 32 bytes of arbitrary data set by the block builder.

**Cross-layer references (post-Merge):**
- `WithdrawalsHash` (EIP-4895, Shanghai) — root hash of the withdrawals list.
- `ParentBeaconRoot` (EIP-4788, Cancun) — the beacon chain parent block root, stored for smart contract access.
- `RequestsHash` (EIP-7685, Prague) — hash of execution-layer requests (validator exits, consolidations).

### How block hashing works

The block hash is the Keccak256 hash of the RLP-encoded header:

```go
// core/types/block.go

func (h *Header) Hash() common.Hash {
    return rlpHash(h)
}
```

The `Block` struct caches this hash to avoid recomputing it:

```go
// core/types/block.go

func (b *Block) Hash() common.Hash {
    if hash := b.hash.Load(); hash != nil {
        return *hash
    }
    h := b.header.Hash()
    b.hash.Store(&h)
    return h
}
```

The first call computes `rlpHash(header)` and stores the result in an `atomic.Pointer[common.Hash]`. All subsequent calls return the cached value without any locking — the same atomic pointer pattern described in Chapter 00.

Note that the block hash depends *only* on the header. The body (transactions, uncles, withdrawals) is committed to indirectly — the header contains `TxHash`, `UncleHash`, and `WithdrawalsHash`, which are root hashes of tries built from the body contents. Changing any transaction would change `TxHash`, which would change the header, which would change the block hash.

---

## 2. Body and Block — The Composite

### Body

The `Body` struct holds the non-header contents of a block:

```go
// core/types/block.go

type Body struct {
    Transactions []*Transaction
    Uncles       []*Header
    Withdrawals  []*Withdrawal `rlp:"optional"`
}
```

- `Transactions` — the ordered list of transactions in the block.
- `Uncles` — uncle (ommer) block headers. This is a legacy field from proof-of-work; post-Merge, it is always empty.
- `Withdrawals` — validator withdrawals from the beacon chain (post-Shanghai, EIP-4895). The `rlp:"optional"` tag means pre-Shanghai blocks can omit this field in RLP encoding.

### Block

The `Block` struct combines header and body into a single unit:

```go
// core/types/block.go

type Block struct {
    header       *Header
    uncles       []*Header
    transactions Transactions
    withdrawals  Withdrawals

    witness *ExecutionWitness

    // caches
    hash atomic.Pointer[common.Hash]
    size atomic.Uint64

    ReceivedAt   time.Time
    ReceivedFrom interface{}
}
```

All the content fields (`header`, `uncles`, `transactions`, `withdrawals`) are unexported. Access goes through methods that return copies to prevent callers from mutating block internals:

- `b.Header()` returns a *copy* of the header (via `CopyHeader`)
- `b.Transactions()` returns the transaction slice
- `b.Number()`, `b.Time()`, `b.GasLimit()`, etc. — convenience accessors that read from the header

The `witness` field holds a Verkle execution witness (for stateless verification), and `ReceivedAt` / `ReceivedFrom` are used by the P2P layer to track when and from whom the block was received.

### Constructing a Block

Blocks are created with `NewBlock`, which takes a header, a body, and receipts:

```go
// core/types/block.go

func NewBlock(header *Header, body *Body, receipts []*Receipt, hasher ListHasher) *Block
```

This constructor deep-copies all inputs and recomputes `TxHash`, `ReceiptHash`, `UncleHash`, and `WithdrawalsHash` from the actual body contents and receipts. It also merges all receipt bloom filters into the header's `Bloom` field. This ensures the header's commitment hashes are always consistent with the body.

---

## 3. The Transaction — `types.Transaction`

Transactions are the most structurally complex type because Ethereum supports multiple transaction formats. Geth handles this with an **envelope pattern**: a single `Transaction` wrapper holds an inner value whose concrete type varies.

### Transaction type constants

Each transaction format has a type ID:

```go
// core/types/transaction.go

const (
    LegacyTxType     = 0x00
    AccessListTxType = 0x01  // EIP-2930
    DynamicFeeTxType = 0x02  // EIP-1559
    BlobTxType       = 0x03  // EIP-4844
    SetCodeTxType    = 0x04  // EIP-7702
)
```

### The envelope

The `Transaction` struct itself is thin — it holds the inner data plus caches:

```go
// core/types/transaction.go

type Transaction struct {
    inner TxData    // Consensus contents of a transaction
    time  time.Time // Time first seen locally (spam avoidance)

    // caches
    hash atomic.Pointer[common.Hash]
    size atomic.Uint64
    from atomic.Pointer[sigCache]
}
```

- `inner` holds one of the five concrete transaction types (described below).
- `time` records when the transaction was first seen locally, used for eviction ordering in the transaction pool.
- The three atomic fields cache the transaction hash, encoded size, and recovered sender address to avoid recomputing them.

All public methods on `Transaction` delegate to the `inner` field through the `TxData` interface:

```go
// core/types/transaction.go

type TxData interface {
    txType() byte
    copy() TxData

    chainID() *big.Int
    accessList() AccessList
    data() []byte
    gas() uint64
    gasPrice() *big.Int
    gasTipCap() *big.Int
    gasFeeCap() *big.Int
    value() *big.Int
    nonce() uint64
    to() *common.Address

    rawSignatureValues() (v, r, s *big.Int)
    setSignatureValues(chainID, v, r, s *big.Int)

    effectiveGasPrice(dst *big.Int, baseFee *big.Int) *big.Int

    encode(*bytes.Buffer) error
    decode([]byte) error

    sigHash(*big.Int) common.Hash
}
```

Every method is unexported (lowercase). The `Transaction` wrapper exposes public versions: `tx.Type()` calls `tx.inner.txType()`, `tx.Gas()` calls `tx.inner.gas()`, and so on. This means external code never touches the inner types directly — it always goes through `Transaction`.

### EIP-2718 encoding

The encoding format depends on the transaction type:

- **Legacy transactions** (type 0x00) are encoded as plain RLP: `RLP([nonce, gasPrice, gas, to, value, data, V, R, S])`.
- **Typed transactions** (types 0x01–0x04) use the EIP-2718 envelope: `TypeByte || RLP(inner)`. The first byte is the type ID, followed by the RLP encoding of the inner struct.

During decoding, geth checks the first byte: if it is > 0x7F, the transaction is legacy RLP (since valid RLP list headers are always ≥ 0xC0). Otherwise, the first byte is the type ID.

---

## 4. The Five Transaction Types

Each transaction type is a struct implementing `TxData`. They evolved over time, each adding new capabilities.

### 4.1 LegacyTx (type 0x00) — the original format

```go
// core/types/tx_legacy.go

type LegacyTx struct {
    Nonce    uint64          // nonce of sender account
    GasPrice *big.Int        // wei per gas
    Gas      uint64          // gas limit
    To       *common.Address `rlp:"nil"` // nil means contract creation
    Value    *big.Int        // wei amount
    Data     []byte          // contract invocation input data
    V, R, S  *big.Int        // signature values
}
```

This is the pre-EIP-2930 transaction format. It has a single `GasPrice` field (both `gasTipCap()` and `gasFeeCap()` return `GasPrice`). There is no explicit chain ID field — for EIP-155 replay-protected transactions, the chain ID is encoded in the V signature value. For unprotected (pre-EIP-155) transactions, V is simply 27 or 28.

The `To` field is a pointer: `nil` means a contract creation transaction; non-nil means a call.

### 4.2 AccessListTx (type 0x01) — EIP-2930

```go
// core/types/tx_access_list.go

type AccessListTx struct {
    ChainID    *big.Int        // destination chain ID
    Nonce      uint64          // nonce of sender account
    GasPrice   *big.Int        // wei per gas
    Gas        uint64          // gas limit
    To         *common.Address `rlp:"nil"` // nil means contract creation
    Value      *big.Int        // wei amount
    Data       []byte          // contract invocation input data
    AccessList AccessList      // EIP-2930 access list
    V, R, S    *big.Int        // signature values
}
```

Introduced in Berlin, this adds two things over `LegacyTx`:
- An explicit `ChainID` field (no longer embedded in V).
- An `AccessList` — a list of addresses and storage keys the transaction plans to access. Pre-declaring access reduces gas costs because the EVM can skip the "cold access" surcharge (see Chapter 01 for the `ColdSloadCostEIP2929` and `WarmStorageReadCostEIP2929` constants).

The `AccessList` type is defined as:

```go
// core/types/tx_access_list.go

type AccessList []AccessTuple

type AccessTuple struct {
    Address     common.Address `json:"address"     gencodec:"required"`
    StorageKeys []common.Hash  `json:"storageKeys" gencodec:"required"`
}
```

Like `LegacyTx`, this type still uses a single `GasPrice` (no tip/fee-cap split).

### 4.3 DynamicFeeTx (type 0x02) — EIP-1559

```go
// core/types/tx_dynamic_fee.go

type DynamicFeeTx struct {
    ChainID    *big.Int
    Nonce      uint64
    GasTipCap  *big.Int // a.k.a. maxPriorityFeePerGas
    GasFeeCap  *big.Int // a.k.a. maxFeePerGas
    Gas        uint64
    To         *common.Address `rlp:"nil"` // nil means contract creation
    Value      *big.Int
    Data       []byte
    AccessList AccessList

    // Signature values
    V *big.Int
    R *big.Int
    S *big.Int
}
```

Introduced in London, this replaces the single `GasPrice` with two fields:
- `GasTipCap` (maxPriorityFeePerGas) — the maximum tip the sender is willing to pay the block builder, above the base fee.
- `GasFeeCap` (maxFeePerGas) — the absolute maximum the sender is willing to pay per gas (base fee + tip).

The effective gas price paid is: `min(GasFeeCap, GasTipCap + baseFee)`. The base fee portion is burned; the tip goes to the block builder. This is the most common transaction type on mainnet today.

### 4.4 BlobTx (type 0x03) — EIP-4844

```go
// core/types/tx_blob.go

type BlobTx struct {
    ChainID    *uint256.Int
    Nonce      uint64
    GasTipCap  *uint256.Int // a.k.a. maxPriorityFeePerGas
    GasFeeCap  *uint256.Int // a.k.a. maxFeePerGas
    Gas        uint64
    To         common.Address
    Value      *uint256.Int
    Data       []byte
    AccessList AccessList
    BlobFeeCap *uint256.Int // a.k.a. maxFeePerBlobGas
    BlobHashes []common.Hash

    Sidecar *BlobTxSidecar `rlp:"-"`

    // Signature values
    V *uint256.Int
    R *uint256.Int
    S *uint256.Int
}
```

Introduced in Cancun, blob transactions carry large data blobs for rollup data availability. There are several structural differences from `DynamicFeeTx`:

- **`*uint256.Int` instead of `*big.Int`** — blob transactions use the `holiman/uint256` library for all large numbers. This is a 256-bit integer type optimized for EVM-sized values, avoiding the heap allocations of `*big.Int`.
- **`To` is NOT a pointer** — blob transactions must always have a recipient (no contract creation via blob tx).
- **`BlobFeeCap`** — a separate fee cap for blob gas, independent of the execution gas fee cap. Blob gas has its own fee market with its own base fee.
- **`BlobHashes`** — the versioned hashes of the blobs attached to this transaction. Each hash is computed as `0x01 || SHA256(commitment)[1:]` — a version byte followed by the truncated SHA256 hash of the KZG commitment. The EVM can access these via the `BLOBHASH` opcode.
- **`Sidecar`** — the actual blob data, commitments, and proofs. This is tagged `rlp:"-"` because the sidecar is NOT part of the consensus encoding — it travels separately and is pruned after the blobs are available. See Chapter 01 for KZG commitment types.

The sidecar struct:

```go
// core/types/tx_blob.go

type BlobTxSidecar struct {
    Version     byte                 // Version
    Blobs       []kzg4844.Blob       // Blobs needed by the blob pool
    Commitments []kzg4844.Commitment // Commitments needed by the blob pool
    Proofs      []kzg4844.Proof      // Proofs needed by the blob pool
}
```

### 4.5 SetCodeTx (type 0x04) — EIP-7702

```go
// core/types/tx_setcode.go

type SetCodeTx struct {
    ChainID    *uint256.Int
    Nonce      uint64
    GasTipCap  *uint256.Int // a.k.a. maxPriorityFeePerGas
    GasFeeCap  *uint256.Int // a.k.a. maxFeePerGas
    Gas        uint64
    To         common.Address
    Value      *uint256.Int
    Data       []byte
    AccessList AccessList
    AuthList   []SetCodeAuthorization

    // Signature values
    V *uint256.Int
    R *uint256.Int
    S *uint256.Int
}
```

Introduced in Prague, this transaction type lets an EOA temporarily delegate its code to a contract address. The unique field is `AuthList` — a list of signed authorizations:

```go
// core/types/tx_setcode.go

type SetCodeAuthorization struct {
    ChainID uint256.Int    `json:"chainId" gencodec:"required"`
    Address common.Address `json:"address" gencodec:"required"`
    Nonce   uint64         `json:"nonce" gencodec:"required"`
    V       uint8          `json:"yParity" gencodec:"required"`
    R       uint256.Int    `json:"r" gencodec:"required"`
    S       uint256.Int    `json:"s" gencodec:"required"`
}
```

Each authorization is signed by the EOA owner, granting permission to set the code at their address to the code of the specified `Address`. Like `BlobTx`, `SetCodeTx` uses `*uint256.Int` and has a non-pointer `To` field.

### Summary table

| Type | ID | Gas model | ChainID | `To` | Special fields | EIP |
|------|----|-----------|---------|------|----------------|-----|
| `LegacyTx` | 0x00 | Single `GasPrice` | Encoded in V | `*common.Address` | — | Original |
| `AccessListTx` | 0x01 | Single `GasPrice` | `*big.Int` | `*common.Address` | `AccessList` | EIP-2930 |
| `DynamicFeeTx` | 0x02 | Tip + Fee cap | `*big.Int` | `*common.Address` | — | EIP-1559 |
| `BlobTx` | 0x03 | Tip + Fee cap | `*uint256.Int` | `common.Address` | `BlobFeeCap`, `BlobHashes`, `Sidecar` | EIP-4844 |
| `SetCodeTx` | 0x04 | Tip + Fee cap | `*uint256.Int` | `common.Address` | `AuthList` | EIP-7702 |

---

## 5. Receipts and Logs — Execution Output

When a transaction executes, it produces a `Receipt` recording what happened. Each receipt contains zero or more `Log` entries emitted by smart contracts.

### 5.1 The Receipt struct

```go
// core/types/receipt.go

const (
    ReceiptStatusFailed     = uint64(0)
    ReceiptStatusSuccessful = uint64(1)
)

type Receipt struct {
    // Consensus fields: These fields are defined by the Yellow Paper
    Type              uint8  `json:"type,omitempty"`
    PostState         []byte `json:"root"`
    Status            uint64 `json:"status"`
    CumulativeGasUsed uint64 `json:"cumulativeGasUsed" gencodec:"required"`
    Bloom             Bloom  `json:"logsBloom"         gencodec:"required"`
    Logs              []*Log `json:"logs"              gencodec:"required"`

    // Implementation fields: These fields are added by geth when processing a transaction.
    TxHash            common.Hash    `json:"transactionHash" gencodec:"required"`
    ContractAddress   common.Address `json:"contractAddress"`
    GasUsed           uint64         `json:"gasUsed" gencodec:"required"`
    EffectiveGasPrice *big.Int       `json:"effectiveGasPrice"`
    BlobGasUsed       uint64         `json:"blobGasUsed,omitempty"`
    BlobGasPrice      *big.Int       `json:"blobGasPrice,omitempty"`

    // Inclusion information: These fields provide information about the inclusion of the
    // transaction corresponding to this receipt.
    BlockHash        common.Hash `json:"blockHash,omitempty"`
    BlockNumber      *big.Int    `json:"blockNumber,omitempty"`
    TransactionIndex uint        `json:"transactionIndex"`
}
```

The fields divide into three groups:

**Consensus fields** (RLP-encoded and part of the receipt trie):
- `Type` — mirrors the transaction type.
- `PostState` — in pre-Byzantium blocks, this held the intermediate state root after each tx. Post-Byzantium, it's empty and `Status` is used instead.
- `Status` — 0 for failure, 1 for success. This replaced `PostState` in the Byzantium fork.
- `CumulativeGasUsed` — total gas used in the block up to and including this transaction.
- `Bloom` — a 256-byte bloom filter for this receipt's logs.
- `Logs` — the log entries emitted during execution.

**Implementation fields** (computed by geth, not part of the receipt trie):
- `TxHash` — hash of the transaction that produced this receipt.
- `ContractAddress` — if the transaction was a contract creation, this is the address of the deployed contract.
- `GasUsed` — gas consumed by this specific transaction (derived from `CumulativeGasUsed` differences).
- `EffectiveGasPrice` — the actual gas price paid (after EIP-1559 base fee calculation).
- `BlobGasUsed`, `BlobGasPrice` — blob gas accounting for EIP-4844 transactions.

**Inclusion information** (filled in when the receipt is retrieved in a block context):
- `BlockHash`, `BlockNumber`, `TransactionIndex` — where in the chain this receipt lives.

### 5.2 The Log struct

Each log entry is generated by the EVM's `LOG0`–`LOG4` opcodes:

```go
// core/types/log.go

type Log struct {
    // Consensus fields:
    Address common.Address `json:"address" gencodec:"required"`
    Topics  []common.Hash  `json:"topics" gencodec:"required"`
    Data    []byte         `json:"data" gencodec:"required"`

    // Derived fields. These fields are filled in by the node
    // but not secured by consensus.
    BlockNumber    uint64      `json:"blockNumber" rlp:"-"`
    TxHash         common.Hash `json:"transactionHash" gencodec:"required" rlp:"-"`
    TxIndex        uint        `json:"transactionIndex" rlp:"-"`
    BlockHash      common.Hash `json:"blockHash" rlp:"-"`
    BlockTimestamp uint64      `json:"blockTimestamp" rlp:"-"`
    Index          uint        `json:"logIndex" rlp:"-"`

    Removed bool `json:"removed" rlp:"-"`
}
```

Only three fields are consensus-critical (included in RLP encoding):
- `Address` — the contract that emitted the event.
- `Topics` — up to 4 indexed 32-byte values. The first topic is typically the Keccak256 hash of the event signature (e.g., `Transfer(address,address,uint256)`). The remaining topics are indexed event parameters.
- `Data` — the non-indexed event data, usually ABI-encoded.

All other fields are tagged `rlp:"-"` — they are derived by the node when building the response, not stored in the receipt trie. The `Removed` field is set to `true` if the log was reverted due to a chain reorganization, which is important for clients subscribing to log events via filters.

### 5.3 Bloom filters

The `Bloom` type is a 256-byte (2048-bit) bit array used for fast log filtering:

```go
// core/types/bloom9.go

const (
    BloomByteLength = 256
    BloomBitLength  = 8 * BloomByteLength  // 2048
)

type Bloom [BloomByteLength]byte
```

When a receipt is created, its bloom filter is populated by adding the address and all topics from every log. For each item, Keccak256 is computed and three bit positions (derived from the first 6 bytes of the hash) are set in the filter.

The header's `Bloom` field is the bitwise OR of all receipt bloom filters in the block. This creates a two-level filtering system: to find logs matching a given topic, first check the header bloom (cheap), then only scan individual receipts if the header bloom matches. This is how `eth_getLogs` queries can skip entire blocks quickly.

---

## 6. The Signer — Recovering the Sender

Ethereum transactions don't contain a "from" field. The sender's address is *recovered* from the ECDSA signature using the process described in Chapter 01. Because the signing scheme has changed across forks, geth uses a `Signer` interface to abstract over the different recovery methods.

### The Signer interface

```go
// core/types/transaction_signing.go

type Signer interface {
    Sender(tx *Transaction) (common.Address, error)
    SignatureValues(tx *Transaction, sig []byte) (r, s, v *big.Int, err error)
    ChainID() *big.Int
    Hash(tx *Transaction) common.Hash
    Equal(Signer) bool
}
```

- `Sender` — recovers the sender address from the transaction's signature.
- `Hash` — returns the "signature hash" — the hash that was signed. This is NOT the transaction hash; it's the hash of the transaction data without the signature fields.
- `SignatureValues` — converts a raw 65-byte signature into the R, S, V components.
- `ChainID` — returns the chain ID this signer expects.

### Choosing the right signer

The signer must match the fork rules of the block containing the transaction. `MakeSigner` selects the correct one:

```go
// core/types/transaction_signing.go

func MakeSigner(config *params.ChainConfig, blockNumber *big.Int, blockTime uint64) Signer {
    var signer Signer
    switch {
    case config.IsPrague(blockNumber, blockTime):
        signer = NewPragueSigner(config.ChainID)
    case config.IsCancun(blockNumber, blockTime):
        signer = NewCancunSigner(config.ChainID)
    case config.IsLondon(blockNumber):
        signer = NewLondonSigner(config.ChainID)
    case config.IsBerlin(blockNumber):
        signer = NewEIP2930Signer(config.ChainID)
    case config.IsEIP155(blockNumber):
        signer = NewEIP155Signer(config.ChainID)
    case config.IsHomestead(blockNumber):
        signer = HomesteadSigner{}
    default:
        signer = FrontierSigner{}
    }
    return signer
}
```

The logic falls through from newest to oldest fork — each signer can handle all transaction types up to its fork level plus all earlier types. A Prague signer can validate legacy, access-list, dynamic-fee, blob, *and* set-code transactions.

### Signer evolution

The signers form a chain, each adding support for newer transaction types:

| Signer | Fork | Tx types supported | Key change |
|--------|------|--------------------|------------|
| `FrontierSigner` | Genesis | Legacy (unprotected) | V = 27 or 28 |
| `HomesteadSigner` | Homestead | Legacy (unprotected) | Enforces low-S malleability fix |
| `EIP155Signer` | Spurious Dragon | Legacy (protected) | V encodes chain ID for replay protection |
| `NewEIP2930Signer` | Berlin | + AccessListTx | Typed transaction support |
| `NewLondonSigner` | London | + DynamicFeeTx | EIP-1559 fee market |
| `NewCancunSigner` | Cancun | + BlobTx | Blob transactions |
| `NewPragueSigner` | Prague | + SetCodeTx | Code delegation |

The modern signers (Berlin onward) are implemented internally as a `modernSigner` struct that uses a bitmap to track which transaction types it accepts. For legacy transactions, it delegates to the appropriate legacy signer.

### Sender recovery and caching

The `Sender` function wraps the signer with a cache:

```go
// core/types/transaction_signing.go

func Sender(signer Signer, tx *Transaction) (common.Address, error) {
    if sigCache := tx.from.Load(); sigCache != nil {
        if sigCache.signer.Equal(signer) {
            return sigCache.from, nil
        }
    }
    addr, err := signer.Sender(tx)
    if err != nil {
        return common.Address{}, err
    }
    tx.from.Store(&sigCache{signer: signer, from: addr})
    return addr, nil
}
```

Since sender recovery involves an `Ecrecover` call (elliptic curve math), the result is cached in the transaction's `from` field. The cache is keyed by signer — if someone calls `Sender` with a different signer, the cache is invalidated and the address is re-derived.

---

## 7. Withdrawals — `types.Withdrawal`

Post-Shanghai (EIP-4895), the consensus layer can push validator withdrawals into the execution layer. These appear in the block body alongside transactions:

```go
// core/types/withdrawal.go

type Withdrawal struct {
    Index     uint64         `json:"index"`          // monotonically increasing identifier issued by consensus layer
    Validator uint64         `json:"validatorIndex"` // index of validator associated with withdrawal
    Address   common.Address `json:"address"`        // target address for withdrawn ether
    Amount    uint64         `json:"amount"`         // value of withdrawal in Gwei
}
```

- `Index` — a global counter that increases with every withdrawal across all blocks.
- `Validator` — which validator is withdrawing.
- `Address` — the execution-layer address that receives the ETH.
- `Amount` — the amount in Gwei (not wei). To convert to wei, multiply by 10^9.

Withdrawals are not transactions — they have no sender, no signature, no gas cost, and no execution. They are simply credit operations applied to the state during block finalization. The header commits to them via `WithdrawalsHash`.

---

## What's Next

You now know every major data type that flows through geth. The next chapter, Chapter 03 — Merkle Patricia Trie, explains the data structure that commits to all of this data — the trie that produces the `Root`, `TxHash`, and `ReceiptHash` fields in the header.
