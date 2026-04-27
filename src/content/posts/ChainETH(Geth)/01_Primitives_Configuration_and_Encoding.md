---
title: Geth(1) Primitives, Configuration, and Encoding
published: 2026-04-10
pinned: false
tags: [BlockChain,Ethereum,Geth]
category: Inside Ethereum
draft: false
---

Before you can read any geth subsystem, you need to know the foundational types and utilities that every other package imports. This chapter covers four building blocks:

1. **`common.Address` and `common.Hash`** — the two types that appear in virtually every function signature
2. **`params.ChainConfig`** — how geth knows which fork rules are active at any block
3. **`rlp`** — RLP encoding, Ethereum's universal serialization format
4. **`crypto`** — Keccak256 hashing, ECDSA signing/recovery, and address derivation

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Every other package in geth imports from these four:        │
  │                                                              │
  │   common/       Address, Hash                                │
  │   params/       ChainConfig, gas constants, fork schedule    │
  │   rlp/          RLP encode/decode                            │
  │   crypto/       Keccak256, ECDSA, address derivation         │
  │                                                              │
  │  These are leaf packages — they depend on nothing else       │
  │  in geth (with minor exceptions), so every chapter after     │
  │  this one can reference them freely.                         │
  └──────────────────────────────────────────────────────────────┘
```

---

## 1. Address and Hash — The Two Universal Types

Open any file in `core/`, `eth/`, or `trie/` and you will find `common.Address` and `common.Hash` within the first few lines of import. They are the most pervasive types in the entire codebase.

### What they are

Both are defined in `common/types.go` as fixed-size byte arrays:

```go
// common/types.go

const (
    HashLength    = 32
    AddressLength = 20
)

// Hash represents the 32 byte Keccak256 hash of arbitrary data.
type Hash [HashLength]byte

// Address represents the 20 byte address of an Ethereum account.
type Address [AddressLength]byte
```

- **`Hash`** is a 32-byte array. It represents Keccak256 hashes — block hashes, transaction hashes, state root hashes, storage slot keys, and any other 32-byte digest in the system.
- **`Address`** is a 20-byte array. It represents Ethereum account addresses — both externally owned accounts (EOAs) and contracts.

Both are value types (arrays, not slices), so they can be used directly as map keys, compared with `==`, and passed by value without worrying about aliasing.

### Constructing them

Both types provide the same trio of constructors:

```go
// common/types.go

// From raw bytes:
func BytesToHash(b []byte) Hash
func BytesToAddress(b []byte) Address

// From a big.Int:
func BigToHash(b *big.Int) Hash
func BigToAddress(b *big.Int) Address

// From a hex string (0x prefix optional):
func HexToHash(s string) Hash
func HexToAddress(s string) Address
```

All of these call `SetBytes()` internally. If the input is longer than 32 (or 20) bytes, it is cropped from the left — only the rightmost bytes are kept.

### Key methods

| Method | Hash | Address | What it does |
|--------|------|---------|--------------|
| `Bytes()` | yes | yes | Returns a `[]byte` slice of the underlying array |
| `Hex()` | yes | yes | Returns hex string with `0x` prefix. Address uses EIP-55 checksum |
| `Big()` | yes | yes | Converts to `*big.Int` |
| `SetBytes(b)` | yes | yes | Copies bytes into the array (right-aligned) |
| `Cmp(other)` | yes | yes | Returns -1, 0, or 1 (lexicographic byte comparison) |
| `String()` | yes | yes | Same as `Hex()` |

Both types also implement `encoding.TextMarshaler`, `encoding.TextUnmarshaler`, `json.Unmarshaler`, and `database/sql` `Scanner`/`Valuer`. Note that neither type has a `MarshalJSON` method — JSON marshaling works because Go's `encoding/json` package falls back to `MarshalText` when `json.Marshaler` is not implemented. The result is that these types serialize correctly everywhere: JSON-RPC responses, database storage, and log output.

### EIP-55 checksummed addresses

When you call `Address.Hex()`, the result is an EIP-55 mixed-case checksum address (e.g. `0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed`). The checksum is computed by hashing the lowercase hex of the address with Keccak256, then capitalizing each hex digit whose corresponding hash nibble is ≥ 8. This provides a lightweight integrity check without changing the address format.

### Utility types

The `common/` package also provides a few supporting types that appear throughout geth:

- **`common/bytes.go`** — hex conversion helpers: `FromHex()`, `Hex2Bytes()`, `Bytes2Hex()`, `LeftPadBytes()`, `RightPadBytes()`
- **`common/big.go`** — pre-allocated `*big.Int` constants: `Big0`, `Big1`, `Big2`, `Big3`, `Big32`, `Big256`, `Big257` (avoids re-allocating common values on every use)

---

## 2. Chain Configuration — `params.ChainConfig`

The Ethereum protocol has evolved through many hard forks, each activating at a specific point in the chain. Geth needs to know *which rules apply* at any given block. That is the job of `ChainConfig`.

### The ChainConfig struct

`ChainConfig` is defined in `params/config.go`. It is stored in the database at genesis and consulted on every block. Here is the full struct:

```go
// params/config.go

type ChainConfig struct {
    ChainID *big.Int `json:"chainId"`

    // Block-number-based forks (pre-Merge)
    HomesteadBlock      *big.Int `json:"homesteadBlock,omitempty"`
    DAOForkBlock        *big.Int `json:"daoForkBlock,omitempty"`
    DAOForkSupport      bool     `json:"daoForkSupport,omitempty"`
    EIP150Block         *big.Int `json:"eip150Block,omitempty"`
    EIP155Block         *big.Int `json:"eip155Block,omitempty"`
    EIP158Block         *big.Int `json:"eip158Block,omitempty"`
    ByzantiumBlock      *big.Int `json:"byzantiumBlock,omitempty"`
    ConstantinopleBlock *big.Int `json:"constantinopleBlock,omitempty"`
    PetersburgBlock     *big.Int `json:"petersburgBlock,omitempty"`
    IstanbulBlock       *big.Int `json:"istanbulBlock,omitempty"`
    MuirGlacierBlock    *big.Int `json:"muirGlacierBlock,omitempty"`
    BerlinBlock         *big.Int `json:"berlinBlock,omitempty"`
    LondonBlock         *big.Int `json:"londonBlock,omitempty"`
    ArrowGlacierBlock   *big.Int `json:"arrowGlacierBlock,omitempty"`
    GrayGlacierBlock    *big.Int `json:"grayGlacierBlock,omitempty"`
    MergeNetsplitBlock  *big.Int `json:"mergeNetsplitBlock,omitempty"`

    // Timestamp-based forks (post-Merge)
    ShanghaiTime  *uint64 `json:"shanghaiTime,omitempty"`
    CancunTime    *uint64 `json:"cancunTime,omitempty"`
    PragueTime    *uint64 `json:"pragueTime,omitempty"`
    OsakaTime     *uint64 `json:"osakaTime,omitempty"`
    // ...additional future forks (BPO1–BPO5, Amsterdam, Verkle)

    // The Merge transition point
    TerminalTotalDifficulty *big.Int `json:"terminalTotalDifficulty,omitempty"`

    DepositContractAddress common.Address `json:"depositContractAddress,omitempty"`

    // Consensus engine configuration
    Ethash             *EthashConfig       `json:"ethash,omitempty"`
    Clique             *CliqueConfig       `json:"clique,omitempty"`
    BlobScheduleConfig *BlobScheduleConfig `json:"blobSchedule,omitempty"`
}
```

The struct breaks naturally into three sections:

- **`ChainID`** — the unique network identifier (1 for mainnet, 11155111 for Sepolia, etc.). Used for EIP-155 replay protection: a transaction signed for chain 1 cannot be replayed on chain 11155111.

- **Block-number forks** (`*big.Int` pointers) — every pre-Merge fork is activated at a specific block number. A `nil` pointer means "this fork never activates on this network." A value of `0` means "active from genesis."

- **Timestamp forks** (`*uint64` pointers) — after the Merge, fork scheduling switched from block numbers to Unix timestamps. The comment in the source marks this transition explicitly. The same `nil` = "never activates" convention applies.

### Two eras of fork scheduling

This is an important design point: forks before the Merge activate by **block number**, and forks after the Merge activate by **block timestamp**.

The reason is that after the Merge, block times became predictable (12 seconds per slot). Using timestamps lets the community agree on a wall-clock activation time (e.g., "March 13, 2024 at 13:55:00 UTC for Cancun") rather than guessing which block number corresponds to a date.

The two internal helper functions that check fork activation reflect this split:

```go
// params/config.go

func isBlockForked(s, head *big.Int) bool {
    if s == nil || head == nil {
        return false
    }
    return s.Cmp(head) <= 0
}

func isTimestampForked(s *uint64, head uint64) bool {
    if s == nil {
        return false
    }
    return *s <= head
}
```

- `isBlockForked` returns true if the fork block `s` is ≤ the current block number `head`.
- `isTimestampForked` returns true if the fork timestamp `s` is ≤ the current block timestamp `head`.
- In both cases, `nil` means "fork not scheduled" and the function returns `false`.

### Fork-checking methods

`ChainConfig` has a `IsXxx()` method for every fork. Pre-Merge forks take only a block number:

```go
// params/config.go

func (c *ChainConfig) IsHomestead(num *big.Int) bool {
    return isBlockForked(c.HomesteadBlock, num)
}

func (c *ChainConfig) IsLondon(num *big.Int) bool {
    return isBlockForked(c.LondonBlock, num)
}
```

Post-Merge forks take *both* a block number and a timestamp, because they additionally require London to be active (all post-Merge forks build on London):

```go
// params/config.go

func (c *ChainConfig) IsShanghai(num *big.Int, time uint64) bool {
    return c.IsLondon(num) && isTimestampForked(c.ShanghaiTime, time)
}

func (c *ChainConfig) IsCancun(num *big.Int, time uint64) bool {
    return c.IsLondon(num) && isTimestampForked(c.CancunTime, time)
}
```

This pattern repeats for Prague, Osaka, and all subsequent forks.

### The Rules snapshot

Calling these `IsXxx()` methods individually on every opcode execution would be wasteful. Instead, at the start of each block, geth snapshots all the fork flags into a flat `Rules` struct:

```go
// params/config.go

type Rules struct {
    ChainID                                                 *big.Int
    IsHomestead, IsEIP150, IsEIP155, IsEIP158               bool
    IsEIP2929, IsEIP4762                                    bool
    IsByzantium, IsConstantinople, IsPetersburg, IsIstanbul bool
    IsBerlin, IsLondon                                      bool
    IsMerge, IsShanghai, IsCancun, IsPrague, IsOsaka        bool
    IsAmsterdam, IsVerkle                                   bool
}
```

`Rules` is created by calling `ChainConfig.Rules(blockNum, isMerge, timestamp)`, which populates every boolean field in one shot. The EVM's jump table and gas calculations then read from this struct — a simple boolean field access instead of a method call and pointer comparison.

---

## 3. Gas Constants — `params/protocol_params.go`

Every opcode, transaction type, and protocol action has a gas cost defined as a constant. These live in `params/protocol_params.go`. You will see these constants referenced throughout the EVM, transaction validation, and block building chapters. Here are the most important ones:

### Transaction costs

| Constant | Value | Meaning |
|----------|-------|---------|
| `TxGas` | 21,000 | Base cost of a simple ETH transfer |
| `TxGasContractCreation` | 53,000 | Base cost of a contract-creating transaction |
| `TxDataZeroGas` | 4 | Per zero byte in transaction calldata |
| `TxDataNonZeroGasFrontier` | 68 | Per non-zero byte (pre-Istanbul) |
| `TxDataNonZeroGasEIP2028` | 16 | Per non-zero byte (Istanbul onward) |
| `TxAccessListAddressGas` | 2,400 | Per address in an EIP-2930 access list |
| `TxAccessListStorageKeyGas` | 1,900 | Per storage key in an EIP-2930 access list |
| `TxAuthTupleGas` | 12,500 | Per authorization tuple in an EIP-7702 transaction |

### EVM limits and memory

| Constant | Value | Meaning |
|----------|-------|---------|
| `StackLimit` | 1,024 | Maximum VM stack depth |
| `CallCreateDepth` | 1,024 | Maximum call/create nesting depth |
| `MemoryGas` | 3 | Per byte of memory expansion |
| `MaxCodeSize` | 24,576 | Maximum deployed contract bytecode (bytes) |
| `MaxInitCodeSize` | 49,152 | Maximum init code size (2 × MaxCodeSize) |

### Storage operations (SSTORE)

Gas costs for SSTORE have been revised multiple times. The constants track each version:

| Constant | Value | Context |
|----------|-------|---------|
| `SstoreSetGas` | 20,000 | Writing to a fresh storage slot |
| `SstoreResetGas` | 5,000 | Changing an existing non-zero slot |
| `ColdSloadCostEIP2929` | 2,100 | First read of a storage slot in a transaction (Berlin+) |
| `WarmStorageReadCostEIP2929` | 100 | Subsequent reads of an already-accessed slot |
| `ColdAccountAccessCostEIP2929` | 2,600 | First access to an account in a transaction (Berlin+) |

### EIP-1559 base fee

| Constant | Value | Meaning |
|----------|-------|---------|
| `DefaultBaseFeeChangeDenominator` | 8 | Base fee can change by up to 1/8 per block |
| `DefaultElasticityMultiplier` | 2 | Block gas limit = 2× the target |
| `InitialBaseFee` | 1,000,000,000 | Initial base fee (1 gwei) when EIP-1559 activated |

### Blob transaction constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `BlobTxBlobGasPerBlob` | 131,072 | Gas charged per blob (2^17) |
| `BlobTxMinBlobGasprice` | 1 | Minimum blob gas price (1 wei) |
| `BlobTxFieldElementsPerBlob` | 4,096 | Field elements per blob |
| `BlobTxBytesPerFieldElement` | 32 | Bytes per field element |

These constants are referenced heavily in Chapter 06 (Transaction Execution), Chapter 07 (The EVM Deep Dive), and Chapter 09 (Block Production).

---

## 4. RLP Encoding — `rlp/`

RLP (Recursive Length Prefix) is Ethereum's serialization format. It is used to encode *everything* that goes over the wire or into the database: transactions, block headers, receipts, trie nodes, and protocol messages. Understanding RLP is essential because you will see `rlp.Encode`, `rlp.DecodeBytes`, and the `rlp` struct tag throughout the codebase.

### What RLP can represent

RLP encodes exactly two things:

1. **Strings** — arbitrary-length byte sequences (including single bytes and empty strings)
2. **Lists** — ordered sequences of strings or other lists (recursive)

That's it. RLP has no concept of integers, booleans, or field names. Higher-level types (like Go structs) are mapped onto these two primitives by the `rlp` package using reflection.

### Encoding rules

Each RLP value starts with a type-tag byte that tells the decoder what follows:

| Byte range | Meaning |
|-----------|---------|
| `[0x00, 0x7F]` | Single byte — the value IS its own encoding |
| `[0x80, 0xB7]` | Short string (0–55 bytes): `0x80 + length`, then data |
| `[0xB8, 0xBF]` | Long string (56+ bytes): `0xB7 + length-of-length`, then length, then data |
| `[0xC0, 0xF7]` | Short list (0–55 bytes total payload): `0xC0 + length`, then items |
| `[0xF8, 0xFF]` | Long list (56+ bytes total payload): `0xF7 + length-of-length`, then length, then items |

A few examples:

- The empty string encodes as `0x80`
- The empty list encodes as `0xC0`
- The integer 0 encodes as `0x80` (same as empty string — zero is the empty byte sequence)
- The integer 127 encodes as `0x7F` (single byte, no prefix)
- The integer 128 encodes as `0x81 0x80` (one-byte string containing `0x80`)
- The string "dog" encodes as `0x83 d o g` (`0x80 + 3` = `0x83`, then three ASCII bytes)

Integers are always encoded as big-endian with no leading zeroes. The decoder enforces canonical encoding — leading-zero integers or single bytes with unnecessary length prefixes are rejected (`ErrCanonInt`, `ErrCanonSize`).

### The Go API

The `rlp` package provides two main entry points for encoding:

```go
// rlp/encode.go

func Encode(w io.Writer, val interface{}) error
func EncodeToBytes(val interface{}) ([]byte, error)
```

And two for decoding:

```go
// rlp/decode.go

func Decode(r io.Reader, val interface{}) error
func DecodeBytes(b []byte, val interface{}) error
```

The package uses Go reflection to map types automatically:

| Go type | RLP encoding |
|---------|-------------|
| `uint`, `uint64`, etc. | String (big-endian, no leading zeroes) |
| `*big.Int` | String (big-endian, no leading zeroes) |
| `bool` | `0x80` (false) or `0x01` (true) |
| `string`, `[]byte`, `[N]byte` | String |
| Struct | List (one element per exported field, in declaration order) |
| Slice / array (except byte) | List |

Signed integers, floats, maps, channels, and functions are not supported.

For custom encoding, a type can implement the `Encoder` or `Decoder` interface:

```go
// rlp/encode.go
type Encoder interface {
    EncodeRLP(io.Writer) error
}

// rlp/decode.go
type Decoder interface {
    DecodeRLP(*Stream) error
}
```

### Struct tags

The `rlp` struct tag controls how individual fields are encoded:

| Tag | Applies to | Effect |
|-----|-----------|--------|
| `rlp:"-"` | Any field | Skip this field entirely |
| `rlp:"optional"` | Any field | May be omitted if zero-valued; all subsequent fields must also be optional |
| `rlp:"tail"` | Last field (slice only) | Absorbs all remaining list elements |
| `rlp:"nil"` | Pointer fields | Nil encodes as empty string or empty list (auto-detected) |
| `rlp:"nilString"` | Pointer fields | Nil encodes as empty string (`0x80`) |
| `rlp:"nilList"` | Pointer fields | Nil encodes as empty list (`0xC0`) |

Here is an example from the codebase showing `optional` and `nil` in practice:

```go
type StructWithOptionalFields struct {
    Required  uint
    Optional1 uint `rlp:"optional"`
    Optional2 uint `rlp:"optional"`
}
```

When encoding, trailing zero-valued optional fields are omitted. When decoding, the input list may have 1, 2, or 3 elements.

### Code generation with rlpgen

For performance-critical types (like `types.Header` and `types.Log`), geth uses the `rlp/rlpgen` tool to generate specialized `EncodeRLP` and `DecodeRLP` methods. These avoid reflection overhead by emitting direct field-by-field encoding code. You will see files named `gen_*_rlp.go` (e.g., `core/types/gen_header_rlp.go`, `core/types/gen_log_rlp.go`) throughout `core/types/` — these are generated, not hand-written.

---

## 5. Cryptographic Primitives — `crypto/`

The `crypto/` package provides three capabilities that the rest of geth depends on: hashing, signing, and address derivation. Together, they form the cryptographic foundation of Ethereum's account model.

### 5.1 Keccak256 hashing

Ethereum uses Keccak256 (the pre-standardization version of SHA-3) as its universal hash function. Block hashes, state roots, transaction hashes, storage keys, and contract addresses are all computed with Keccak256.

The primary functions are in `crypto/keccak.go`:

```go
// crypto/keccak.go

func Keccak256(data ...[]byte) []byte
func Keccak256Hash(data ...[]byte) (h common.Hash)
```

- `Keccak256` accepts one or more byte slices, hashes their concatenation, and returns a 32-byte slice.
- `Keccak256Hash` does the same but returns a `common.Hash` directly.

Both functions use a `sync.Pool` of hasher objects to avoid allocating a new SHA3 state on every call — an important optimization since hashing is one of the most frequent operations in geth.

### 5.2 ECDSA signing and recovery

Ethereum uses the **secp256k1** elliptic curve for all digital signatures. The `crypto` package defines three constants related to signatures:

```go
// crypto/crypto.go

const SignatureLength = 64 + 1  // 64 bytes ECDSA signature + 1 byte recovery id
const RecoveryIDOffset = 64     // byte position of recovery id
const DigestLength = 32         // signature digest length
```

An Ethereum signature is 65 bytes: 32 bytes for R, 32 bytes for S, and 1 byte for the recovery ID (V, either 0 or 1). The recovery ID is what makes it possible to recover the signer's public key — and therefore their address — from just the signature and the message hash.

The key signing and recovery functions:

```go
// crypto/signature_cgo.go (or signature_nocgo.go)

func Sign(digestHash []byte, prv *ecdsa.PrivateKey) (sig []byte, err error)
func Ecrecover(hash, sig []byte) ([]byte, error)
func SigToPub(hash, sig []byte) (*ecdsa.PublicKey, error)
func VerifySignature(pubkey, digestHash, signature []byte) bool
```

- **`Sign`** takes a 32-byte hash and a private key, returns a 65-byte `[R || S || V]` signature. Uses RFC 6979 deterministic nonce generation (so signing the same hash with the same key always produces the same signature).
- **`Ecrecover`** takes a hash and a 65-byte signature, returns the 65-byte uncompressed public key of the signer. This is the function used to determine who sent a transaction — without it, Ethereum's account model wouldn't work.
- **`SigToPub`** is a convenience wrapper around `Ecrecover` that returns a typed `*ecdsa.PublicKey`.
- **`VerifySignature`** checks a 64-byte `[R || S]` signature against a known public key (no recovery needed).

Geth supports two backends for secp256k1 operations, selected at compile time:

- **CGO backend** (`signature_cgo.go`) — wraps the C `libsecp256k1` library for maximum performance. This is the default on most platforms.
- **Pure Go backend** (`signature_nocgo.go`) — uses the `decred/dcrd` secp256k1 implementation. Used when CGO is unavailable (e.g., WebAssembly builds).

The two backends produce identical results — only performance differs.

### 5.3 Address derivation

The most important function in the crypto package is `PubkeyToAddress`. This is how Ethereum addresses are derived from public keys:

```go
// crypto/crypto.go

func PubkeyToAddress(p ecdsa.PublicKey) common.Address {
    pubBytes := FromECDSAPub(&p)
    return common.BytesToAddress(Keccak256(pubBytes[1:])[12:])
}
```

The derivation pipeline:

```
Private Key (32 bytes)
    │
    │  secp256k1 scalar multiplication
    ▼
Public Key (65 bytes: 0x04 || X || Y)
    │
    │  Strip the 0x04 prefix → 64 bytes
    │  Keccak256 hash → 32 bytes
    │  Take last 20 bytes (bytes [12:32])
    ▼
Address (20 bytes)
```

Walking through the code:
- `FromECDSAPub(&p)` serializes the public key as 65 bytes in uncompressed form: the prefix byte `0x04` followed by the 32-byte X coordinate and the 32-byte Y coordinate.
- `pubBytes[1:]` strips the `0x04` prefix, leaving 64 bytes.
- `Keccak256(...)` hashes those 64 bytes into a 32-byte digest.
- `[12:]` takes the last 20 bytes of that digest — this is the Ethereum address.

This is a one-way function. You cannot recover the public key from an address (the Keccak256 hash is irreversible). However, you *can* recover the public key from a signature (via `Ecrecover`), and then derive the address from that. This is exactly what happens when geth validates a transaction — it recovers the sender's public key from the transaction signature, derives the address, and uses that address to look up the account balance and nonce.

### 5.4 Signature validation

Before accepting a signature, geth validates that R and S are in the correct range:

```go
// crypto/crypto.go

func ValidateSignatureValues(v byte, r, s *big.Int, homestead bool) bool {
    if r.Cmp(common.Big1) < 0 || s.Cmp(common.Big1) < 0 {
        return false
    }
    if homestead && s.Cmp(secp256k1halfN) > 0 {
        return false
    }
    return r.Cmp(secp256k1N) < 0 && s.Cmp(secp256k1N) < 0 && (v == 0 || v == 1)
}
```

- R and S must both be ≥ 1 and < N (the curve order).
- After the Homestead fork, S must also be ≤ N/2. This prevents **ECDSA malleability** — without this check, anyone could take a valid signature `(R, S, V)` and produce a second valid signature `(R, N-S, 1-V)` for the same message. The malleability fix ensures only one canonical signature exists per message.

### 5.5 Contract address creation

The `crypto` package also computes the addresses of newly created contracts:

```go
// crypto/crypto.go

// CREATE: address = keccak256(rlp([sender, nonce]))[12:]
func CreateAddress(b common.Address, nonce uint64) common.Address {
    data, _ := rlp.EncodeToBytes([]interface{}{b, nonce})
    return common.BytesToAddress(Keccak256(data)[12:])
}

// CREATE2: address = keccak256(0xff ++ sender ++ salt ++ keccak256(initcode))[12:]
func CreateAddress2(b common.Address, salt [32]byte, inithash []byte) common.Address {
    return common.BytesToAddress(Keccak256([]byte{0xff}, b.Bytes(), salt[:], inithash)[12:])
}
```

- **`CreateAddress`** (used by the `CREATE` opcode) — the new contract's address is determined by the deployer's address and their current nonce. This means the address is predictable if you know the sender and nonce, but changes every time the sender deploys a new contract.
- **`CreateAddress2`** (used by `CREATE2`, EIP-1014) — the address is determined by the deployer's address, a 32-byte salt, and the hash of the init code. This makes the address fully deterministic and independent of the nonce — the same init code with the same salt always deploys to the same address.

### 5.6 KZG commitments — `crypto/kzg4844/`

EIP-4844 (Dencun upgrade) introduced **blob transactions** — a way to attach large data blobs to transactions for rollup data availability. These blobs use KZG polynomial commitments for integrity verification.

The `crypto/kzg4844` package defines the blob-related types:

```go
// crypto/kzg4844/kzg4844.go

type Blob [131072]byte       // 128 KB data blob
type Commitment [48]byte     // KZG commitment (BLS12-381 point)
type Proof [48]byte          // KZG proof (quotient polynomial commitment)
```

The key operations are:
- `BlobToCommitment(blob *Blob) (Commitment, error)` — create a cryptographic commitment to the blob data
- `VerifyBlobProof(blob *Blob, commitment Commitment, proof Proof) error` — verify that a blob matches its commitment

Like secp256k1 signing, KZG supports two backends: a C implementation (`ckzg`) for production performance and a pure Go implementation (`gokzg`) for portability. Both load a shared trusted setup from an embedded `trusted_setup.json` file containing the KZG parameters for 4096-coefficient polynomials.

Blob transactions are covered in detail in Chapter 02 (Core Data Types).

---

## What's Next

With these primitives in hand, you now know the vocabulary that every other package speaks. Chapter 02 — Core Data Types introduces the "nouns" of the system — `Block`, `Header`, `Transaction`, `Receipt`, and `Log` — the structs that flow through every subsystem from the transaction pool to the blockchain database.
