---
title: EELS(2) Q&A Notes
published: 2026-03-31
pinned: false
description: Some QA that might be helpful.
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

1. [How do `bytes_to_nibble_list` and `encode_node` work?](#1-how-do-bytes_to_nibble_list-and-encode_node-work)
2. [Why doesn't `Account` contain `storage_root`?](#2-why-doesnt-account-contain-storage_root)
3. [Why the `assert storage_root is not None` guards?](#3-why-the-assert-storage_root-is-not-none-guards)
4. [Why does Ethereum use RLP encoding?](#4-why-does-ethereum-use-rlp-encoding)
5. [How do `get_storage` and `set_storage` work with the two-trie structure?](#5-how-do-get_storage-and-set_storage-work-with-the-two-trie-structure)
6. [Why `assert isinstance(value, U256)` in `get_storage`?](#6-why-assert-isinstancevalue-u256-in-get_storage)
7. [How does `patricialize` work?](#7-how-does-patricialize-work--line-by-line)
8. [How do `nibble_list_to_compact` and `encode_internal_node` work?](#8-how-do-nibble_list_to_compact-and-encode_internal_node-work)
9. [How are `encode_internal_node` return values used inside `patricialize`?](#9-how-are-encode_internal_node-return-values-used-inside-patricialize)
10. [How does Geth store data in practice vs. EELS?](#10-how-does-geth-store-data-in-practice-vs-eels)
11. [How does Geth find data when a user queries an account balance?](#11-how-does-geth-find-data-when-a-user-queries-an-account-balance)

---

## 1. How do `bytes_to_nibble_list` and `encode_node` work?

### `bytes_to_nibble_list`

**What it does:** Takes any byte string and splits each byte into two nibbles (high 4 bits + low 4 bits), doubling the array length. The result is a sequence of values where each value is between 0 and 15.

**The code:**

```python
def bytes_to_nibble_list(bytes_: Bytes) -> Bytes:
    nibble_list = bytearray(2 * len(bytes_))
    for byte_index, byte in enumerate(bytes_):
        nibble_list[byte_index * 2] = (byte & 0xF0) >> 4   # high nibble
        nibble_list[byte_index * 2 + 1] = byte & 0x0F       # low nibble
    return Bytes(nibble_list)
```

**Concrete example** — converting the first 4 bytes of an address `0xd73be913...`:

The raw bytes:

```
[0xd7, 0x3b, 0xe9, 0x13]
```

The function processes each byte one at a time:

| Byte | Binary | `byte & 0xF0 >> 4` (high) | `byte & 0x0F` (low) |
|------|--------|---------------------------|---------------------|
| `0xd7` | `1101 0111` | `0x0d` (13) | `0x07` (7) |
| `0x3b` | `0011 1011` | `0x03` (3)  | `0x0b` (11) |
| `0xe9` | `1110 1001` | `0x0e` (14) | `0x09` (9) |
| `0x13` | `0001 0011` | `0x01` (1)  | `0x03` (3) |

**Result:** `[13, 7, 3, 11, 14, 9, 1, 3]` — 8 nibbles from 4 bytes. For a full 20-byte address this produces 40 nibbles. For a `keccak256`-hashed key (32 bytes), it produces 64 nibbles.

**Why this step?** The MPT is a hexary (16-way) trie. Each branch decision consumes exactly one nibble (0–15). The trie needs to navigate nibble-by-nibble, not byte-by-byte. Without this expansion, the trie couldn't branch on the 16 possible values at each level.

---

### `encode_node`

**What it does:** A type dispatcher that serializes different Python value types into bytes suitable for trie leaf storage. Different value types need different encoding strategies.

**The code:**

```python
def encode_node(node: Node, storage_root: Optional[Bytes] = None) -> Bytes:
    if isinstance(node, Account):
        assert storage_root is not None
        return encode_account(node, storage_root)
    elif isinstance(
        node, (LegacyTransaction, Receipt, Withdrawal, U256, Uint)
    ):
        return rlp.encode(node)
    elif isinstance(node, Bytes):
        return node
    else:
        return previous_trie.encode_node(node, storage_root)
```

This is a **type dispatcher**. The trie stores many different kinds of values — accounts in the state trie, transactions in the transactions trie, receipts in the receipts trie, storage slot values (U256) in storage tries. Each type has its own encoding path.

**Concrete example** — encoding an Account:

Our account:

```python
Account(
    nonce=Uint(5),
    balance=U256(2_000_000_000_000_000_000),  # 2 ETH in wei
    code_hash=EMPTY_CODE_HASH,                  # keccak256(b"")
)
```

No storage → `storage_root = EMPTY_TRIE_ROOT` (`0x56e81f17...`, 32 bytes)

`encode_node` calls `encode_account`:

```python
def encode_account(raw_account_data: Account, storage_root: Bytes) -> Bytes:
    return rlp.encode((
        raw_account_data.nonce,       # 5
        raw_account_data.balance,     # 2_000_000_000_000_000_000
        storage_root,                 # EMPTY_TRIE_ROOT (32 bytes)
        raw_account_data.code_hash,   # EMPTY_CODE_HASH  (32 bytes)
    ))
```

So it RLP-encodes this 4-tuple:

```
(5, 2000000000000000000, 0x56e81f17..., 0xc5d24601...)
```

The RLP output is roughly **~75 bytes** of serialized data — something like:

```
b'\xf8u\x05\x88\x1b\xe14\xa7e\x80\x00\xa0V\xe8\x1f\x17...\xa0\xc5\xd2...'
```

This byte string is the **value** that will be stored in the trie leaf.

---

### How `_prepare_trie` combines them

`_prepare_trie` is the bridge between the flat dictionary and the trie builder. It applies both functions to every entry:

```python
def _prepare_trie(trie, get_storage_root=None):
    mapped = {}

    for preimage, value in trie._data.items():
        # preimage = b'\xd7\x3b\xe9\x13...'   (raw address bytes)
        # value    = Account(nonce=5, balance=2ETH, ...)

        # STEP 1: encode the value
        encoded_value = encode_node(value, get_storage_root(address))
        # encoded_value = b'\xf8u\x05\x88\x1b...'  (~75 bytes RLP)

        # STEP 2: hash the key (secured trie)
        key = keccak256(preimage)
        # key = 32-byte hash, e.g. b'\xa7\x11\x35\x5f...'

        # STEP 3: expand key into nibbles
        mapped[bytes_to_nibble_list(key)] = encoded_value
        # mapped key = [10, 7, 1, 1, 3, 5, 5, 15, ...]  (64 nibbles)
        # mapped value = b'\xf8u\x05\x88\x1b...'        (RLP account bytes)

    return mapped
```

The output is a **flat dictionary** ready for `patricialize`:

```python
{
    # key: 64 nibbles                           # value: RLP-encoded account
    bytes([10, 7, 1, 1, 3, 5, 5, 15, ...]): b'\xf8u\x05\x88\x1b...',
}
```

The big picture pipeline:

```
[Raw address bytes]          [Account object]
          │                           │
          ▼                           ▼
   [keccak256()]              [encode_node()]
     (32 bytes)                (RLP encoding)
          │                           │
          ▼                           │
 [bytes_to_nibble_list()]             │
     (64 nibbles)                     │
          │                           │
          └──────────────┬────────────┘
                         ▼
            [_prepare_trie output dict]
                         │
                         ▼
                  [patricialize()]
                 (builds the trie)
```

The two functions serve orthogonal purposes:
- **`bytes_to_nibble_list`** transforms the **key** so the trie can navigate nibble-by-nibble.
- **`encode_node`** transforms the **value** into a serializable byte blob.

Neither function touches the tree structure itself — they just prepare the raw ingredients so `patricialize` can consume them.

---

## 2. Why doesn't `Account` contain `storage_root`?

### The structural split

In EELS, storage and account metadata live in two completely separate places:

```python
# Account — only metadata
@slotted_freezable
@dataclass
class Account:
    nonce: Uint
    balance: U256
    code_hash: Hash32
    # No storage_root

# State — storage lives separately
@dataclass
class State:
    _main_trie: Trie[Address, Optional[Account]] = ...       # one trie, all accounts
    _storage_tries: Dict[Address, Trie[Bytes32, U256]] = ... # many tries, one per contract
```

The `_storage_tries` dictionary holds one independent storage trie per contract address. The `Account` struct knows nothing about it. The storage root is only computed on demand by calling `storage_root(state, address)`, which walks the actual storage trie and hashes it.

### Reasons for this design

**Single source of truth.** The storage trie *is* the storage. If `Account` also carried a `storage_root` field, that field could theoretically drift out of sync with the actual trie data. EELS avoids this by never caching the root — it's always computed fresh from the real trie. There's only one place where storage lives, and only one way to get its root: walk the trie.

**Storage changes constantly, accounts don't carry substructure.** `nonce`, `balance`, and `code_hash` are simple scalars. They're just numbers or a 32-byte hash. There's no substructure to manage. You read them, you overwrite them, done. `storage_root` is not a simple value — it's the root hash of an entire Merkle Patricia Trie. That trie can contain thousands of slot entries, each with its own key, value, node hashes, and tree structure. It's an entirely separate data structure with its own lifecycle, its own mutation API (`trie_set`/`trie_get`), its own snapshot mechanism, and its own original-value lookups for EIP-2200 gas pricing. So the question becomes: should `Account` own that entire trie, or should `State` own it? EELS puts it in `State` because the storage trie doesn't conceptually belong *inside* the account — it belongs *alongside* it.

**Immutability matters.** `Account` is decorated with `@slotted_freezable`, meaning it's designed to be frozen into an immutable object after creation. Storage, by contrast, is highly mutable during execution. Keeping them separate lets you freeze the account metadata while the storage trie remains a live, mutable data structure.

**Storage can be created lazily.** A contract might be deployed but never write to storage. In EELS, no entry exists in `_storage_tries` until `set_storage` is called for the first time. If `Account` contained the storage trie, every contract would carry an empty trie from creation, even if it never uses it.

**The root is only meaningful at serialization time.** During execution, no one needs the storage root. It's not read by any opcode, not used in any gas calculation, not checked against any invariant. The only thing that ever reads it is `state_root()` at the end of the block, which needs it to build the RLP-encoded account for the state trie. Computing it on demand from the actual trie data is simpler than maintaining a cached field that must be invalidated after every `SSTORE`.

**EELS is a specification, not a production client.** This is the real reason. Geth caches `storage_root` inside its `StateObject` for performance — recomputing a Merkle root from a large trie is expensive, and production clients need to avoid redundant work. But EELS prioritizes correctness and readability over speed. There's no performance penalty in a reference implementation for computing the root on demand, and doing so makes the code simpler and harder to get wrong. Geth actually does the same structural separation — it stores the storage trie as a separate pointer (`stateObject.trie`) alongside the account data (`stateObject.data`). Geth caches the root hash in `stateObject.data.Root` for performance, but the trie itself is still a separate object. EELS just takes the same architecture one step further: don't cache the root at all, compute it when needed.

### When does the storage root actually get used?

Only at one precise moment: when computing the state root at the end of block execution.

The call chain is:

```
state_root()
  → defines get_storage_root(address) callback
    → root(main_trie, get_storage_root)
      → _prepare_trie(main_trie, get_storage_root)
        → for each Account: encode_account(account, get_storage_root(address))
          → storage_root(state, address)
            → root(storage_trie)   ← walks the trie, computes the hash
```

Until that moment, storage is just a mutable trie sitting in `_storage_tries`. No one cares about its root. No one caches it. It doesn't need to exist until the header demands it.

---

## 3. Why the `assert storage_root is not None` guards?

### In `encode_node`:

```python
def encode_node(node: Node, storage_root: Optional[Bytes] = None) -> Bytes:
    if isinstance(node, Account):
        assert storage_root is not None
        return encode_account(node, storage_root)
```

The RLP encoding of an account is a mandatory 4-tuple: `(nonce, balance, storage_root, code_hash)`. This is the Ethereum wire format mandated by the protocol. If `storage_root` were `None`, we couldn't produce a valid encoding. The `assert` is a defensive check: if someone calls `encode_node(account)` without providing the storage root, the code should fail loudly rather than silently produce garbage.

### In `_prepare_trie`:

```python
def _prepare_trie(trie, get_storage_root=None):
    for preimage, value in trie._data.items():
        if isinstance(value, Account):
            assert get_storage_root is not None
            encoded_value = encode_node(value, get_storage_root(address))
```

`_prepare_trie` is called from `root()`, which is called from `state_root()`. `state_root()` always provides the callback. The chain is: `state_root()` → `root()` → `_prepare_trie()` → `encode_node()`. At every step, if the trie contains accounts, the callback **must** be provided. The `assert` catches a programming error — calling `state_root()` on an incomplete state or calling `root()` directly on an account trie without supplying the callback — and fails loudly at the source of the mistake.

---

## 4. Why does Ethereum use RLP encoding?

### Reason 1: Determinism (canonical encoding)

This is the **single most important reason**. In a consensus protocol, every node must independently compute the *exact same* state root, transaction root, and receipt root. If two nodes serialize the same data differently, their roots diverge and consensus breaks.

JSON is not deterministic:

```json
{"balance": 100, "nonce": 5}   ← key order A
{"nonce": 5, "balance": 100}   ← key order B (same data, different bytes, different hash)
```

RLP is deterministic by construction:

```python
rlp.encode([5, 100])  # always b'\xc6\x05\x82\x00d' — no ambiguity
```

RLP has no key names, no whitespace choices, no number formatting decisions. A list always encodes to the same bytes.

### Reason 2: Simplicity — only two types

RLP handles exactly two things:
1. **Byte strings** (raw bytes)
2. **Lists** (ordered collections of byte strings and/or nested lists)

That's it. No integers (they're just byte strings), no booleans, no maps, no nulls. This simplicity means:
- Easy to implement correctly in any language (critical for a multi-client ecosystem)
- Easy to formally verify
- Fewer edge cases for consensus bugs

### Reason 3: Recursive structure enables nesting

Transactions are lists. Blocks are lists of transactions plus a header. MPT nodes are lists. An account is a list of `[nonce, balance, storage_root, code_hash]`, where `storage_root` and `code_hash` are themselves hashes of structures that are also RLP-encoded.

```
Block header → transactions_root → RLP(tx_list) → each tx is RLP(...)
             → state_root → MPT → each account is RLP([nonce, balance, RLP(storage_trie), code_hash])
```

RLP's recursive list structure maps naturally onto Ethereum's recursive data structures.

### Reason 4: Length-prefixing enables streaming parsing

The first byte of an RLP-encoded item tells you exactly how long the payload is. A parser can read items sequentially without loading the entire blob into memory:

```
0x83 0x01 0x02 0x03
  ↑    ↑   ↑   ↑
  │    └───┴───┴── 3 bytes of payload
  └─ 0x80 + 3 = length ≤ 55, so payload is 3 bytes
```

This matters when parsing large blocks or large MPT nodes.

### Why not alternatives?

| Format | Why not |
|--------|---------|
| JSON | Non-deterministic (key order, whitespace, number precision). No native byte string type. |
| Protobuf | Requires schema files. Field numbering is not self-describing. Overly complex for consensus. |
| MessagePack | Multiple valid encodings for the same integer (e.g., `1` can be 1 byte or 5 bytes). |
| CBOR | Good alternative, but didn't exist when Ethereum was designed (2013). |
| ASN.1/DER | Used in X.509 certs. Works, but far more complex than needed. |

### The trade-off

RLP is **not** space-efficient (small integers aren't varint-encoded), **not** human-readable, and **not** self-describing (you need external schema to know what `[5, 2000000000000000000, root, hash]` means). But none of those matter for a consensus protocol. What matters is: **same input → same bytes → same hash → universal agreement**. RLP delivers that with minimal complexity.

---

## 5. How do `get_storage` and `set_storage` work with the two-trie structure?

### The two-trie structure

```
State
  ├── _main_trie      → Trie[Address, Account]    (one trie, holds all accounts)
  └── _storage_tries  → Dict[Address, Trie]        (many tries, one per contract)
```

**`_main_trie`** is a single MPT. Its keys are addresses, its values are `Account` objects. Every address in Ethereum has at most one entry here.

**`_storage_tries`** is a Python `dict` mapping addresses to *separate* MPTs. Each contract that uses `SSTORE` gets its own independent storage trie. An EOA (externally owned account) never appears here.

### Visual example: a real contract

Let's say we have a simple ERC-20 token at address `0x1234...`. After some transfers, its storage looks like this conceptually:

```
slot 0 (totalSupply):     1_000_000
slot 1 (balances[alice]): 500_000
slot 2 (balances[bob]):   300_000
```

**What `_main_trie` looks like:**

```python
_main_trie._data = {
    Address(0x1234...): Account(
        nonce=Uint(1),
        balance=U256(0),
        code_hash=keccak256(erc20_bytecode),
    ),
    Address(0xaaaa...): Account(   # alice
        nonce=Uint(5),
        balance=U256(1_000_000_000_000_000_000),  # 1 ETH
        code_hash=EMPTY_CODE_HASH,
    ),
    # ... more accounts
}
```

One flat dict: address → Account. No storage info here at all.

**What `_storage_tries` looks like:**

```python
_storage_tries = {
    Address(0x1234...): Trie(
        secured=True,
        default=U256(0),
        _data={
            Bytes32(0x00...00): U256(1_000_000),     # slot 0: totalSupply
            Bytes32(0x00...01): U256(500_000),        # slot 1: alice's balance
            Bytes32(0x00...02): U256(300_000),        # slot 2: bob's balance
        },
    ),
    # ... other contracts with storage
}
```

Only the contract address `0x1234...` appears here, not alice's EOA address. EOAs don't have storage.

### Note on how "ownership" works in storage

The storage trie doesn't know who owns what balance, and it doesn't need to. The storage trie is just a raw **slot number → value** map. "Ownership" is determined entirely by the **contract's code**, not by the storage structure.

In an ERC-20 contract, balances are stored via a Solidity mapping:

```solidity
mapping(address => uint256) public balances;
```

The compiler assigns this to a base slot — say slot 1. But the actual storage slot for a specific address is computed as:

```
actual_slot = keccak256(key_padded ++ base_slot)
```

Where `key_padded` is the address zero-padded to 32 bytes. So Alice's balance isn't at slot 1 — it's at a hashed slot that only the contract's bytecode knows how to compute.

The storage trie has no idea that `0x7f3d8c1a...` represents "Alice's balance." It's just a 32-byte key with a number at it. The EVM bytecode computes the hash and `SLOAD`s the result. The trie is a dumb key-value store; the semantics live in the contract code.

### How `get_storage` works

```python
def get_storage(state: State, address: Address, key: Bytes32) -> U256:
    trie = state._storage_tries.get(address)
    if trie is None:
        return U256(0)          # no storage trie → all slots are 0

    value = trie_get(trie, key)

    assert isinstance(value, U256)
    return value
```

Concrete example — reading slot 1 of the ERC-20 contract:

```python
get_storage(state, Address(0x1234...), Bytes32(0x00...01))
```

Step by step:

1. Look up `state._storage_tries.get(Address(0x1234...))`. Found the storage trie for that contract.
2. Call `trie_get(trie, Bytes32(0x00...01))`. This looks up the key in the storage trie's `_data` dict.
3. The key `0x00...01` maps to `U256(500_000)`.
4. Return `U256(500_000)`.

Now try reading a slot that was never set:

```python
get_storage(state, Address(0x1234...), Bytes32(0x00...ff))
```

1. The storage trie exists.
2. `trie_get(trie, Bytes32(0x00...ff))` — the key is not in `_data`.
3. `trie_get` returns `trie.default`, which is `U256(0)`.
4. Return `U256(0)`.

Now try reading from an EOA that has no storage:

```python
get_storage(state, Address(0xaaaa...), Bytes32(0x00...00))
```

1. `state._storage_tries.get(Address(0xaaaa...))` → `None` (alice is an EOA, no entry).
2. Return `U256(0)` immediately.

### How `set_storage` works

```python
def set_storage(state: State, address: Address, key: Bytes32, value: U256) -> None:
    assert trie_get(state._main_trie, address) is not None  # account must exist

    trie = state._storage_tries.get(address)
    if trie is None:
        trie = Trie(secured=True, default=U256(0))
        state._storage_tries[address] = trie
    trie_set(trie, key, value)
    if trie._data == {}:
        del state._storage_tries[address]
```

Concrete example — a contract writes to slot 3:

```python
set_storage(state, Address(0x1234...), Bytes32(0x00...03), U256(999))
```

Step by step:

1. **Assert account exists.** The contract must already be in `_main_trie`. You can't write storage for a non-existent account.
2. `state._storage_tries.get(Address(0x1234...))` → the storage trie already exists from earlier writes.
3. `trie_set(trie, Bytes32(0x00...03), U256(999))` — inserts `{0x00...03: 999}` into the trie's `_data` dict.
4. The dict is not empty, so we keep the trie.

Now the first-ever storage write to this contract:

```python
# Before this call, _storage_tries has no entry for 0x1234...
set_storage(state, Address(0x1234...), Bytes32(0x00...00), U256(1_000_000))
```

1. Account exists in `_main_trie` — passes the assert.
2. `state._storage_tries.get(...)` → `None`. This is the first storage write.
3. Create a new empty trie: `Trie(secured=True, default=U256(0))`.
4. Store it: `state._storage_tries[0x1234...] = trie`.
5. `trie_set(trie, Bytes32(0x00...00), U256(1_000_000))` — inserts the first entry.

Now the deletion case — setting a slot back to zero:

```python
set_storage(state, Address(0x1234...), Bytes32(0x00...03), U256(0))
```

1. The storage trie exists.
2. `trie_set(trie, Bytes32(0x00...03), U256(0))`. But `U256(0)` equals `trie.default`, so `trie_set` **deletes the key** from `_data` rather than storing it.
3. If this was the last entry and `_data` is now empty → `del state._storage_tries[address]`. The entire storage trie is removed.

### The key insight

The two tries answer fundamentally different questions:

**`_main_trie`**: "What is the account at this address?" → returns nonce, balance, code_hash. One trie for all addresses.

**`_storage_tries[address]`**: "What value does this contract have at this storage slot?" → returns a `U256`. One trie per contract, created on first write, deleted when last slot is cleared.

The link between them is the **address**. The same `Address(0x1234...)` appears as a key in both `_main_trie._data` and `_storage_tries`. But they store completely different things and are mutated by different operations.

---

## 6. Why `assert isinstance(value, U256)` in `get_storage`?

It's a defensive check to catch programming errors, because **Python dictionaries don't enforce types at runtime**.

`trie_get` is a **generic function** that works for *any* trie in the entire codebase. It returns whatever `V` happens to be for that specific trie.

```python
def trie_get(trie: Trie[K, V], key: K) -> V:
    return trie._data.get(key, trie.default)
```

For `_storage_tries`, the type annotation *says* `V` is `U256`:

```python
_storage_tries: Dict[Address, Trie[Bytes32, U256]]
```

But Python ignores type annotations at runtime. If a bug somewhere else in the code accidentally inserted the wrong type into that trie's `_data` dictionary — say, a `Uint` instead of a `U256`, or even an `Account` if the wrong trie got passed around — `trie_get` would happily return that wrong value. The type checker (`mypy`) can't catch runtime mistakes.

The `assert` catches that immediately:

```python
value = trie_get(trie, key)
assert isinstance(value, U256)  # crash here if something else is in the dict
return value
```

Without the assert, the wrong type would silently propagate into the EVM's gas calculations or opcode logic, producing incorrect execution results that would be very hard to debug. The assert turns a subtle logic bug into a loud, immediate crash with a clear error message.

---

## 7. How does `patricialize` work — line by line?

### The input

`patricialize` receives a **flat dictionary** where keys are nibble lists and values are RLP-encoded data:

```python
keyA = bytes([3, 7, 10, 2, 9, 15])     # 6 nibbles — Account A
keyB = bytes([3, 7, 10, 5, 1, 12])     # 6 nibbles — Account B
keyC = bytes([8, 11, 0, 4, 14, 3])     # 6 nibbles — Account C

obj = {
    keyA: valA,    # b'\xd7\x80\x64...'
    keyB: valB,    # b'\xd7\x80\xc8...'
    keyC: valC,    # b'\xd7\x80\x01...'
}
```

A and B share the prefix `[3, 7, 10]`. C diverges immediately at the first nibble (`8` vs `3`).

The function's job: **turn this flat dict into a tree of Branch/Extension/Leaf nodes.**

---

### The full code with annotations

#### Block 1: Empty dict

```python
if len(obj) == 0:
    return None
```

If no keys route through this subtree, there's nothing here. Return `None`, which `encode_internal_node` turns into `b""`.

#### Block 2: Single key → Leaf

```python
arbitrary_key = next(iter(obj))

if len(obj) == 1:
    leaf = LeafNode(arbitrary_key[level:], obj[arbitrary_key])
    return leaf
```

Only one key remains. No branching is needed. The entire remaining path from `level` onward becomes a Leaf node.

#### Block 3: Find the longest shared prefix

```python
substring = arbitrary_key[level:]
prefix_length = len(substring)
for key in obj:
    prefix_length = min(
        prefix_length,
        common_prefix_length(substring, key[level:])
    )
    if prefix_length == 0:
        break
```

This walks every key and keeps the **shortest** common prefix length found so far. That shortest length is the prefix **all** keys share. If any key has zero common prefix at the current level, we break early.

#### Block 4: Shared prefix → Extension

```python
if prefix_length > 0:
    prefix = arbitrary_key[int(level) : int(level) + prefix_length]
    return ExtensionNode(
        prefix,
        encode_internal_node(
            patricialize(obj, level + Uint(prefix_length))
        ),
    )
```

All keys share a prefix. Create an Extension node for that prefix, then **recurse** past the shared prefix — same `obj`, but `level` advanced.

#### Block 5: No shared prefix → Branch

```python
branches: List[MutableMapping[Bytes, Bytes]] = []
for _ in range(16):
    branches.append({})
value = b""
for key in obj:
    if len(key) == level:
        if isinstance(obj[key], (Account, Receipt, Uint)):
            raise AssertionError
        value = obj[key]
    else:
        branches[key[level]][key] = obj[key]

subnodes = tuple(
    encode_internal_node(patricialize(branches[k], level + Uint(1)))
    for k in range(16)
)
return BranchNode(
    cast(BranchSubnodes, assert_type(subnodes, Tuple[Extended, ...])),
    value,
)
```

Keys diverge here. Create 16 buckets (one per nibble 0–15). Route each key into the bucket matching its nibble at position `level`. Then recurse into all 16 buckets.

---

### Concrete trace with our 3 accounts

#### CALL 1: `patricialize(obj, level=Uint(0))`

```
Line 1:  def patricialize(obj, level: Uint):
         → obj has 3 entries, level = 0

Line 2:  if len(obj) == 0:
         → len(obj) = 3 → False, skip

Line 3:  arbitrary_key = next(iter(obj))
         → arbitrary_key = keyA = bytes([3, 7, 10, 2, 9, 15])

Line 4:  if len(obj) == 1:
         → 3 == 1 → False, skip

Line 5:  substring = arbitrary_key[level:]
         → keyA[0:] = bytes([3, 7, 10, 2, 9, 15])
         → substring = bytes([3, 7, 10, 2, 9, 15])

Line 6:  prefix_length = len(substring)
         → prefix_length = 6

Line 7:  for key in obj:
         → Iteration begins. 3 keys to check.

         ─── Iteration 1: key = keyA ───

Line 8:  prefix_length = min(
             prefix_length,
             common_prefix_length(substring, key[level:])
         )
         → key[level:] = keyA[0:] = bytes([3, 7, 10, 2, 9, 15])
         → common_prefix_length([3,7,10,2,9,15], [3,7,10,2,9,15])
         → Comparing element by element: all 6 match → returns 6
         → prefix_length = min(6, 6) = 6

Line 9:  if prefix_length == 0:
         → 6 == 0 → False, continue

         ─── Iteration 2: key = keyB ───

Line 8:  common_prefix_length(substring, key[level:])
         → key[level:] = keyB[0:] = bytes([3, 7, 10, 5, 1, 12])
         → common_prefix_length([3,7,10,2,9,15], [3,7,10,5,1,12])
         → index 0: 3==3 ✓
         → index 1: 7==7 ✓
         → index 2: 10==10 ✓
         → index 3: 2≠5 ✗ → returns 3
         → prefix_length = min(6, 3) = 3

Line 9:  if prefix_length == 0:
         → 3 == 0 → False, continue

         ─── Iteration 3: key = keyC ───

Line 8:  common_prefix_length(substring, key[level:])
         → key[level:] = keyC[0:] = bytes([8, 11, 0, 4, 14, 3])
         → common_prefix_length([3,7,10,2,9,15], [8,11,0,4,14,3])
         → index 0: 3≠8 ✗ → returns 0
         → prefix_length = min(3, 0) = 0

Line 9:  if prefix_length == 0:
         → 0 == 0 → True → break

Line 10: (loop exited early)

Line 11: if prefix_length > 0:
         → 0 > 0 → False → skip Extension block

Line 12: branches: List[MutableMapping[Bytes, Bytes]] = []
         → branches = []

Line 13: for _ in range(16):
              branches.append({})
         → branches = [{}, {}, {}, {}, {}, {}, {}, {},
                       {}, {}, {}, {}, {}, {}, {}, {}]
         → 16 empty dicts, indices 0–15

Line 14: value = b""
         → value = b""

Line 15: for key in obj:
         → Iteration begins. 3 keys.

         ─── Iteration 1: key = keyA ───

Line 16: if len(key) == level:
         → len(keyA) = 6, level = 0
         → 6 == 0 → False

Line 17: else:
              branches[key[level]][key] = obj[key]
         → key[level] = keyA[0] = 3
         → branches[3][keyA] = valA
         → branches[3] is now {keyA: valA}

         ─── Iteration 2: key = keyB ───

Line 16: len(keyB) = 6, level = 0 → 6 == 0 → False
Line 17: keyB[0] = 3
         → branches[3][keyB] = valB
         → branches[3] is now {keyA: valA, keyB: valB}

         ─── Iteration 3: key = keyC ───

Line 16: len(keyC) = 6, level = 0 → 6 == 0 → False
Line 17: keyC[0] = 8
         → branches[8][keyC] = valC
         → branches[8] is now {keyC: valC}

Line 18: (loop done)
         → Final branches state:
           branches[3] = {keyA: valA, keyB: valB}
           branches[8] = {keyC: valC}
           all others = {}

Line 19: subnodes = tuple(
             encode_internal_node(patricialize(branches[k], level + Uint(1)))
             for k in range(16)
         )
         → 16 recursive calls, one per bucket.
```

#### CALL 1.0 through 1.2: `patricialize(branches[0..2], level=1)`

All three are empty dicts.

```
Line 2:  if len(obj) == 0:
         → True → return None
```

Result: `None` → `encode_internal_node(None)` → `b""`

#### CALL 1.3: `patricialize(branches[3], level=1)`

```
obj = {keyA: valA, keyB: valB}    # 2 entries
level = 1
```

```
Line 2:  len(obj) = 2 → False, skip

Line 3:  arbitrary_key = keyA = bytes([3, 7, 10, 2, 9, 15])

Line 4:  len(obj) == 1 → 2 == 1 → False, skip

Line 5:  substring = arbitrary_key[level:]
         → keyA[1:] = bytes([7, 10, 2, 9, 15])
         → substring = bytes([7, 10, 2, 9, 15])

Line 6:  prefix_length = len(substring)
         → prefix_length = 5

Line 7:  for key in obj:
         → 2 keys.

         ─── Iteration 1: key = keyA ───

Line 8:  key[level:] = keyA[1:] = bytes([7, 10, 2, 9, 15])
         → common_prefix_length([7,10,2,9,15], [7,10,2,9,15])
         → All 5 match → returns 5
         → prefix_length = min(5, 5) = 5

Line 9:  5 == 0 → False, continue

         ─── Iteration 2: key = keyB ───

Line 8:  key[level:] = keyB[1:] = bytes([7, 10, 5, 1, 12])
         → common_prefix_length([7,10,2,9,15], [7,10,5,1,12])
         → index 0: 7==7 ✓
         → index 1: 10==10 ✓
         → index 2: 2≠5 ✗ → returns 2
         → prefix_length = min(5, 2) = 2

Line 9:  2 == 0 → False, continue

Line 7:  (loop done, both keys checked)

Line 11: if prefix_length > 0:
         → 2 > 0 → True → enter Extension block

Line 12: prefix = arbitrary_key[int(level) : int(level) + prefix_length]
         → keyA[1 : 1+2] = keyA[1:3]
         → keyA = bytes([3, 7, 10, 2, 9, 15])
         → keyA[1:3] = bytes([7, 10])
         → prefix = bytes([7, 10])

Line 13: return ExtensionNode(
              prefix,
              encode_internal_node(
                  patricialize(obj, level + Uint(prefix_length))
              ),
          )
         → patricialize({keyA:valA, keyB:valB}, level = 1 + 2 = 3)
         → Need to make a recursive call. Pausing CALL 1.3.
```

#### CALL 1.3.1: `patricialize({keyA:valA, keyB:valB}, level=3)`

```
obj = {keyA: valA, keyB: valB}
level = 3
```

```
Line 2:  len(obj) = 2 → False

Line 3:  arbitrary_key = keyA = bytes([3, 7, 10, 2, 9, 15])

Line 4:  2 == 1 → False

Line 5:  substring = arbitrary_key[level:]
         → keyA[3:] = bytes([2, 9, 15])
         → substring = bytes([2, 9, 15])

Line 6:  prefix_length = len(substring)
         → prefix_length = 3

Line 7:  for key in obj:
         → 2 keys.

         ─── Iteration 1: key = keyA ───

Line 8:  key[3:] = bytes([2, 9, 15])
         → common_prefix_length([2,9,15], [2,9,15])
         → All 3 match → returns 3
         → prefix_length = min(3, 3) = 3

Line 9:  3 == 0 → False, continue

         ─── Iteration 2: key = keyB ───

Line 8:  keyB[3:] = bytes([5, 1, 12])
         → common_prefix_length([2,9,15], [5,1,12])
         → index 0: 2≠5 ✗ → returns 0
         → prefix_length = min(3, 0) = 0

Line 9:  0 == 0 → True → break

Line 11: if prefix_length > 0:
         → 0 > 0 → False → skip Extension

Line 12-14: branches = [{} for _ in range(16)]
             value = b""

Line 15: for key in obj:
         → 2 keys.

         ─── key = keyA ───

Line 16: len(keyA) = 6, level = 3 → 6 == 3 → False
Line 17: keyA[3] = 2
         → branches[2][keyA] = valA

         ─── key = keyB ───

Line 16: len(keyB) = 6, level = 3 → 6 == 3 → False
Line 17: keyB[3] = 5
         → branches[5][keyB] = valB

         branches state:
           branches[2] = {keyA: valA}
           branches[5] = {keyB: valB}
           all others = {}

Line 19: subnodes = tuple(
             encode_internal_node(patricialize(branches[k], level + Uint(1)))
             for k in range(16)
         )
         → 16 calls at level=4. Tracing the interesting ones:
```

**CALL 1.3.1.2: `patricialize({keyA: valA}, level=4)`**

```
Line 2:  len = 1 → False

Line 3:  arbitrary_key = keyA

Line 4:  len(obj) == 1 → True → enter Leaf block

Line 5:  leaf = LeafNode(arbitrary_key[level:], obj[arbitrary_key])
         → arbitrary_key[4:] = keyA[4:] = bytes([9, 15])
         → obj[keyA] = valA
         → leaf = LeafNode(rest_of_key=bytes([9, 15]), value=valA)
         → return leaf
```

Back to Line 19:

```
encode_internal_node(LeafNode(bytes([9,15]), valA))
→ LeafNode: unencoded = (nibble_list_to_compact([9,15], True), valA)
→ [9,15] is even length, is_leaf=True
→ compact: prefix byte = 16*(2*True) = 0x20, then pack [9,15] → 0x9f
→ unencoded = (b'\x20\x9f', valA)
→ rlp.encode(unencoded) → ~80 bytes (valA is ~75 bytes)
→ 80 ≥ 32 → return keccak256(rlp.encode(unencoded)) = hashA
```

So `subnodes[2] = hashA` (a 32-byte hash).

**CALL 1.3.1.5: `patricialize({keyB: valB}, level=4)`**

```
Line 3:  arbitrary_key = keyB
Line 4:  len == 1 → True

Line 5:  leaf = LeafNode(keyB[4:], valB)
         → keyB[4:] = bytes([1, 12])
         → leaf = LeafNode(rest_of_key=bytes([1, 12]), value=valB)
         → return leaf
```

```
encode_internal_node(LeafNode(bytes([1,12]), valB))
→ unencoded = (nibble_list_to_compact([1,12], True), valB)
→ [1,12] even, leaf → prefix = 0x20, pack → 0x1c
→ unencoded = (b'\x20\x1c', valB)
→ rlp.encode(...) ≥ 32 bytes → return keccak256(...) = hashB
```

So `subnodes[5] = hashB`.

All other buckets (0, 1, 3, 4, 6–15) are empty → return `None` → encoded = `b""`.

**Back to Line 19 of CALL 1.3.1 — all 16 subnodes resolved:**

```
subnodes = (
    b"", b"", hashA, b"", b"", hashB,
    b"", b"", b"", b"", b"", b"",
    b"", b"", b"", b""
)
value = b""
```

```
Line 20: return BranchNode(subnodes, value)
         → BranchNode(
               subnodes=(b"", b"", hashA, b"", b"", hashB, ...),
               value=b""
           )
```

CALL 1.3.1 returns this BranchNode.

#### Back to CALL 1.3 — resuming from the pause

```
Line 13: return ExtensionNode(
              prefix=bytes([7, 10]),
              subnode=encode_internal_node(BranchNode_from_above),
          )
```

```
encode_internal_node(BranchNode(subnodes=(b"",b"",hashA,...), value=b""))
→ unencoded = [b"", b"", hashA, ..., b"", b""] + [b""]
→ that's 17 items in the list
→ rlp.encode(...) is certainly ≥ 32 bytes
→ return keccak256(rlp.encode(...)) = hashInner
```

```
→ return ExtensionNode(key_segment=bytes([7, 10]), subnode=hashInner)
```

CALL 1.3 returns this ExtensionNode.

#### CALL 1.4 through 1.7 (k=4,5,6,7): empty
→ All `return None` → encoded = `b""`

#### CALL 1.8: `patricialize({keyC: valC}, level=1)`

```
obj = {keyC: valC}
level = 1
```

```
Line 2:  len = 1 → False

Line 3:  arbitrary_key = keyC = bytes([8, 11, 0, 4, 14, 3])

Line 4:  len(obj) == 1 → True

Line 5:  leaf = LeafNode(arbitrary_key[level:], obj[arbitrary_key])
         → keyC[1:] = bytes([11, 0, 4, 14, 3])
         → leaf = LeafNode(rest_of_key=bytes([11, 0, 4, 14, 3]), value=valC)
         → return leaf
```

```
encode_internal_node(LeafNode(bytes([11,0,4,14,3]), valC))
→ unencoded = (nibble_list_to_compact([11,0,4,14,3], True), valC)
→ [11,0,4,14,3] has odd length (5)
→ prefix = 16*((2*True)+1) + 11 = 16*3+11 = 0x3b
→ then pack [0,4,14,3] → 0x04, 0xe3
→ compact = bytes([0x3b, 0x04, 0xe3])
→ unencoded = (b'\x3b\x04\xe3', valC)
→ rlp.encode(...) ≥ 32 bytes → return keccak256(...) = hashC
```

CALL 1.8 returns `hashC`.

#### CALL 1.9 through 1.15 (k=9–15): empty
→ All `return None` → encoded = `b""`

#### Back to CALL 1 — all 16 subnodes resolved

```
subnodes = (
    b"",                        # k=0
    b"",                        # k=1
    b"",                        # k=2
    ExtensionNode([7,10], hashInner),  # k=3
    b"",                        # k=4
    b"",                        # k=5
    b"",                        # k=6
    b"",                        # k=7
    hashC,                      # k=8
    b"",                        # k=9
    ...b""...                   # k=10-15
)
value = b""
```

```
Line 20: return BranchNode(subnodes, value)
```

CALL 1 returns this root BranchNode.

### Resulting tree structure

```
BranchNode (root)
├── slot 0: b""  (empty)
├── slot 1: b""  (empty)
├── slot 2: b""  (empty)
├── slot 3: ExtensionNode([7, 10], →)
│           └── BranchNode
│               ├── slot 0-1: b""
│               ├── slot 2: LeafNode([9,15], valA)  → hashed to hashA
│               ├── slot 3-4: b""
│               ├── slot 5: LeafNode([1,12], valB)  → hashed to hashB
│               └── slots 6-15: b""
├── slot 4-7: b""
├── slot 8: LeafNode([11,0,4,14,3], valC)  → hashed to hashC
├── slots 9-15: b""
└── value: b"" (no key terminates at root)
```

---

## 8. How do `nibble_list_to_compact` and `encode_internal_node` work?

### `nibble_list_to_compact`

This function takes a list of nibbles (values 0–15) and packs them into a compact byte array with a flag byte prepended. The flag byte encodes two pieces of information: whether this is a leaf or extension node, and whether the path has even or odd length.

#### Case A: Even-length leaf

Input from our trace — the leaf for Account A:

```python
x = bytes([9, 15])     # rest_of_key = [9, 15]
is_leaf = True
```

```
Line 1:  def nibble_list_to_compact(x: Bytes, is_leaf: bool) -> Bytes:
         → x = bytes([9, 15]), is_leaf = True

Line 2:  compact = bytearray()
         → compact = bytearray()

Line 3:  if len(x) % 2 == 0:
         → len([9, 15]) = 2
         → 2 % 2 == 0 → True → enter the even branch

Line 4:  compact.append(16 * (2 * is_leaf))
         → 16 * (2 * True) = 16 * 2 = 32 = 0x20
         → compact = bytearray([0x20])

         Explanation of this byte:
         0x20 = 0010 0000 in binary
         The high nibble (0010) encodes:
           - bit 1 = 1 (is_leaf = True)
           - bit 0 = 0 (even parity)
         So 0x20 = "leaf, even length"

Line 5:  for i in range(0, len(x), 2):
         → i = 0 only (range(0, 2, 2) = [0])

Line 6:      compact.append(16 * x[i] + x[i + 1])
         → i = 0:
         → 16 * x[0] + x[1] = 16 * 9 + 15 = 144 + 15 = 159 = 0x9f
         → compact = bytearray([0x20, 0x9f])

Line 7:  return Bytes(compact)
         → return b'\x20\x9f'
```

Result: **`b'\x20\x9f'`** — 2 bytes representing the nibble list `[9, 15]` with a "leaf, even" flag.

#### Case B: Odd-length leaf

Input from our trace — the leaf for Account C:

```python
x = bytes([11, 0, 4, 14, 3])   # rest_of_key = [11, 0, 4, 14, 3] — 5 nibbles
is_leaf = True
```

```
Line 1:  x = bytes([11, 0, 4, 14, 3]), is_leaf = True

Line 2:  compact = bytearray()

Line 3:  if len(x) % 2 == 0:
         → len = 5
         → 5 % 2 == 0 → False → enter the odd branch

Line 4:  compact.append(16 * ((2 * is_leaf) + 1) + x[0])
         → (2 * True) + 1 = 3
         → 16 * 3 + x[0] = 48 + 11 = 59 = 0x3b
         → compact = bytearray([0x3b])

         Explanation:
         0x3b = 0011 1011 in binary
         High nibble 0x3 = 0011:
           - bit 1 = 1 (is_leaf = True)
           - bit 0 = 1 (odd length, first nibble absorbed into prefix byte)
         The low nibble of the prefix byte (0xb = 11) is the first nibble of the path.

Line 5:  for i in range(1, len(x), 2):
         → range(1, 5, 2) = [1, 3]
         → i = 1 and i = 3

         ─── i = 1 ───

Line 6:      compact.append(16 * x[i] + x[i + 1])
         → 16 * x[1] + x[2] = 16 * 0 + 4 = 4 = 0x04
         → compact = bytearray([0x3b, 0x04])

         ─── i = 3 ───

Line 6:      compact.append(16 * x[i] + x[i + 1])
         → 16 * x[3] + x[4] = 16 * 14 + 3 = 224 + 3 = 227 = 0xe3
         → compact = bytearray([0x3b, 0x04, 0xe3])

Line 7:  return Bytes(compact)
         → return b'\x3b\x04\xe3'
```

Result: **`b'\x3b\x04\xe3'`** — 3 bytes. The original 5 nibbles are packed into 2.5 bytes, but the flag byte absorbs the half, giving us exactly 3 bytes.

#### Case C: Even-length extension

```python
x = bytes([7, 10])    # key_segment = [7, 10]
is_leaf = False
```

```
Line 3:  len = 2, even → True

Line 4:  compact.append(16 * (2 * is_leaf))
         → 16 * (2 * False) = 16 * 0 = 0 = 0x00
         → compact = bytearray([0x00])

         0x00 = 0000 0000:
           - bit 1 = 0 (extension, not leaf)
           - bit 0 = 0 (even length)
         So 0x00 = "extension, even"

Line 5:  for i in range(0, 2, 2):
         → i = 0

Line 6:      compact.append(16 * x[0] + x[1])
         → 16 * 7 + 10 = 112 + 10 = 122 = 0x7a
         → compact = bytearray([0x00, 0x7a])

Line 7:  return b'\x00\x7a'
```

Result: **`b'\x00\x7a'`** — 2 bytes.

#### Prefix byte summary

| First byte | Meaning | Formula used |
|---|---|---|
| `0x00` | Extension, even path | `16 * (2*False) = 0x00` |
| `0x1X` | Extension, odd path | `16 * (2*False+1) + X` |
| `0x20` | Leaf, even path | `16 * (2*True) = 0x20` |
| `0x3X` | Leaf, odd path | `16 * (2*True+1) + X` |

The pattern: bits 0–1 of the high nibble encode `(is_leaf << 1) | (length_parity)`. When odd, the first nibble is absorbed into the low nibble of the prefix byte. When even, a `0` padding nibble is added after the prefix.

---

### `encode_internal_node`

This function takes a Python tree node and turns it into either an **inline value** or a **32-byte hash**. It's the function that makes the trie a **Merkle** trie.

#### Case A: Encoding `None`

```python
node = None
```

```
Line 1:  def encode_internal_node(node):

Line 2:  if node is None:
         → True

Line 3:      unencoded = b""

Line 4:  encoded = rlp.encode(unencoded)
         → rlp.encode(b"") = b'\x80'
         → len(b'\x80') = 1

Line 5:  if len(encoded) < 32:
         → 1 < 32 → True

Line 6:      return unencoded
         → return b""
```

An empty branch slot encodes as `b""`. The parent will include this `b""` directly in its own RLP.

#### Case B: Encoding a LeafNode (large — gets hashed)

```python
node = LeafNode(
    rest_of_key=bytes([9, 15]),
    value=b'\xd7\x80\x64...',    # RLP-encoded account, ~75 bytes
)
```

```
Line 1:  node is LeafNode → enter that branch

Line 2:  unencoded = (
             nibble_list_to_compact(node.rest_of_key, True),
             node.value,
         )
         → nibble_list_to_compact([9, 15], True) = b'\x20\x9f'  (from Case A above)
         → unencoded = (b'\x20\x9f', b'\xd7\x80\x64...')

Line 3:  encoded = rlp.encode(unencoded)
         → rlp.encode((b'\x20\x9f', b'\xd7\x80\x64...'))
         → The value is ~75 bytes, plus 2 bytes for the key prefix
         → Total RLP encoding is roughly 80 bytes
         → encoded = b'\xf8...' (some 80-ish byte blob)

Line 4:  if len(encoded) < 32:
         → 80 < 32 → False

Line 5:  else:
             return keccak256(encoded)
         → return keccak256(b'\xf8...')
         → = Hash32, some 32-byte value like:
           b'\xa7\x11\x35\x5f...'
```

The leaf is **too big to inline**, so it gets hashed. The hash is what the parent Extension/Branch node stores as a reference to this leaf.

#### Case C: Encoding a tiny inline node (hypothetical)

Imagine a tiny storage trie with a very short key and a small value:

```python
node = LeafNode(
    rest_of_key=bytes([1]),       # 1 nibble
    value=b'\x01',                # just 1 byte
)
```

```
Line 1:  unencoded = (nibble_list_to_compact([1], True), b'\x01')
         → [1] is odd, leaf → prefix = 16*3+1 = 0x31
         → unencoded = (b'\x31', b'\x01')

Line 2:  encoded = rlp.encode((b'\x31', b'\x01'))
         → b'\x31' is a single byte in [0x00, 0x7f] range → encoded as itself
         → b'\x01' ditto
         → The list has 2 bytes of payload → list prefix = 0xc0 + 2 = 0xc2
         → encoded = b'\xc2\x31\x01'
         → len(encoded) = 3

Line 3:  if len(encoded) < 32:
         → 3 < 32 → True

Line 4:      return unencoded
         → return (b'\x31', b'\x01')
```

This node is **inlined**. Its parent will receive the raw Python structure `(b'\x31', b'\x01')` and include it directly in its own RLP encoding. No separate hash lookup needed.

#### Case D: Encoding a BranchNode

```python
node = BranchNode(
    subnodes=(
        b"",                                          # slot 0
        b"",                                          # slot 1
        b'\xa7\x11\x35\x5f...',                       # slot 2: hash of Leaf A
        b"",                                          # slot 3
        b"",                                          # slot 4
        b'\xb2\xc8\x4d\x1a...',                       # slot 5: hash of Leaf B
        b"", b"", b"", b"", b"", b"", b"", b"", b"", b"",  # slots 6-15
    ),
    value=b"",                                        # nothing terminates at root
)
```

```
Line 1:  node is BranchNode

Line 2:  unencoded = list(node.subnodes) + [node.value]
         → unencoded = [
               b"", b"", hashA, b"", b"", hashB,
               b"", b"", b"", b"", b"", b"",
               b"", b"", b"", b"",
               b""                    # the value slot (index 16)
           ]
         → 17 items total

Line 3:  encoded = rlp.encode(unencoded)
         → Each b"" encodes as 1 byte (0x80)
         → Each 32-byte hash encodes as 0xa0 ++ 32 bytes = 33 bytes
         → 14 empty slots × 1 byte = 14
         → 2 hash slots × 33 bytes = 66
         → 1 value slot × 1 byte = 1
         → List prefix: 14 + 66 + 1 = 81 bytes of payload
         → 81 > 55, so list prefix is 0xf7 + len(81) = 0xf8 0x51 = 2 bytes
         → Total: ~83 bytes
         → encoded = b'\xf8\x51\x80\x80\xa0\xa7\x11...'

Line 4:  if len(encoded) < 32:
         → 83 < 32 → False

Line 5:  else:
             return keccak256(encoded)
         → return keccak256(b'\xf8\x51\x80\x80\xa0...')
         → = some 32-byte root hash
```

This BranchNode is large, so it gets hashed. That hash is what becomes the **state root** (or, if this is a sub-branch, the reference its parent stores).

---

## 9. How are `encode_internal_node` return values used inside `patricialize`?

The return value of `encode_internal_node` **becomes data inside the parent node**. It doesn't get "stored" anywhere — it becomes a field value of the parent Python object. `patricialize` doesn't care whether the value is a hash or a raw structure. It just passes it through.

Two call sites inside `patricialize`:

**Extension node:**

```python
return ExtensionNode(
    prefix,
    encode_internal_node(patricialize(obj, level + Uint(prefix_length))),
    #  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    #  This return value becomes the `subnode` field
)
```

**Branch node:**

```python
subnodes = tuple(
    encode_internal_node(patricialize(branches[k], level + Uint(1)))
    for k in range(16)
)
# Each return value becomes one element of the `subnodes` tuple
```

The recursion is **bottom-up**. Every parent waits for its children to return before it can be constructed:

```python
# Inside patricialize, when building a Branch:
subnodes = tuple(
    encode_internal_node(patricialize(branches[k], level + Uint(1)))
    #                ^^^^^^^^^^^^                    ^^^^^^^^^^^^^^^^
    #   1. Recurse into child                     2. Child returns encoded
    #      (blocks until done)                       value (hash or raw)
    for k in range(16)
)

# 3. Only NOW, with all 16 encoded values,
#    can we build the parent BranchNode
return BranchNode(subnodes, value)
```

The same pattern holds for Extension:

```python
return ExtensionNode(
    prefix,
    encode_internal_node(
        patricialize(obj, level + Uint(prefix_length))
    ),
    #     ^^^^^^^^^^^^  waits for child to return
    #          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ encoded value goes into subnode field
)
```

The tree grows from leaves upward. By the time the top-level `patricialize` returns, every node has already been encoded and either hashed or inlined by its parent.

### Inlined (small child) vs Hashed (large child)

**Inlined (small child):**

```python
BranchNode(
    subnodes=(
        b"", b"",
        (b'\x37', b'\x01'),    # raw leaf structure, NOT a hash
        ...
    ),
    value=b""
)
```

When this Branch is RLP-encoded, the raw structure `(b'\x37', b'\x01')` is included directly as part of the parent's RLP. The parent's single hash covers everything. No separate hash for the child.

**Hashed (large child):**

```python
BranchNode(
    subnodes=(
        b"", b"",
        hashA,    # 32-byte hash
        hashB,    # 32-byte hash
        ...
    ),
    value=b""
)
```

When this Branch is RLP-encoded, `hashA` and `hashB` are embedded as raw 32-byte blobs. They are **not re-hashed**. The root hash commits to them, which in turn commit to leaf data. The chain of hashes is the Merkle proof.

### Why two return types?

Because of the **< 32 bytes optimization**. If every node were always hashed, even a tiny trie like `[{bytes([1]): b'\x01'}]` would require two hashes for essentially 2 bytes of data. Wasteful.

Instead, small nodes are **inlined into their parent's RLP**. The parent's RLP encoding includes the child's raw data. Then the parent gets hashed once. One hash covers everything. No separate hash for the child.

- **Small nodes**: inlined, zero extra DB lookups, one hash at the parent level
- **Large nodes**: hashed, the hash is stored as a reference, allows verifying a leaf without downloading the entire tree (you only need the hash path from root to leaf)

---

## 10. How does Geth store data in practice vs. EELS?

### EELS: state machine, not a database

EELS rebuilds the trie from scratch every time `root()` is called and throws everything away afterward. It's a **state machine**, not a database. Suitable for verification and specification, not for serving queries or persisting data. EELS asks: *"Given this set of changes, what is the correct state root?"*

### Geth: key-value database with incremental mutations

Geth uses an embedded key-value store — originally **LevelDB**, now also **Pebble**. Everything is a `(key, value)` pair. There are no tables, no rows, no SQL. Just a flat namespace of byte strings.

Geth asks: *"Given this state root, how do I serve queries, sync new nodes, and apply the next block as fast as possible?"*

They're solving the same problem from opposite directions.

### The Foundation: Content-Addressed Storage

The trie nodes are stored using **content-addressed storage**:

```
Key   = keccak256(RLP(node))
Value = RLP(node)
```

If you know the hash of a node, you look it up directly. This is a **hash-based scheme**. The root hash of the state trie is the only thing you need to find everything else — you fetch the root node by its hash, decode its RLP, find child hashes, fetch those, and so on.

A real LevelDB entry looks like:

```
Key:   0xa711355f... (32 bytes, hash of the node)
Value: 0xf8518080a0... (RLP-encoded node, ~80 bytes)
```

Leaf nodes store the actual account data. Branch and Extension nodes store references (hashes) to children. The entire world state is one giant web of these hash-linked entries.

### The Mutation Model: Dirty Nodes and Journaling

Unlike EELS, which rebuilds the whole trie from a flat dictionary, Geth maintains a **live in-memory trie** and only writes the *differences* to disk.

When a transaction executes:

1. **Mark nodes dirty.** As the EVM writes to storage, Geth modifies the relevant leaf nodes, then their parent branches, then the extension nodes above them. Each modified node is marked `dirty` in memory.
2. **Compute the root hash.** The dirty nodes are re-hashed bottom-up, just like `encode_internal_node` does in EELS. New hashes become new LevelDB keys.
3. **Commit a batch.** At the end of the block, all dirty nodes are written to LevelDB in a single atomic batch. The old nodes they replace are eventually deleted (garbage collected).

This is called a **trie journal**. Each block produces a journal of "insert these nodes, delete those nodes." The database is an append-mostly, delete-later structure.

### The Snapshot Architecture: Diff Layers

Geth doesn't just have one trie on disk. It has a **stack of diff layers** that allow it to serve state queries for recent blocks without re-executing them.

```
Disk Layer (persistent, on-disk trie)
    │
    ├── Diff Layer 1 (block N-127, in memory)
    │       └── Diff Layer 2 (block N-63, in memory)
    │               └── ...
    │                       └── Diff Layer 128 (current block, in-memory)
```

Each diff layer records only the nodes that changed at that block height. When you query an account at block N-50, Geth walks down the diff stack: if the account was modified in a diff layer, return that version; otherwise fall through to the disk layer.

This is why Geth can serve `eth_getStorageAt` for any of the last ~128 blocks instantly — it never touches LevelDB for those queries.

### Path-Based Storage (Modern Geth)

The hash-based scheme has a problem: **iterating over all accounts is slow**. You can't list all accounts without traversing the entire trie from the root, following every hash pointer. This made state sync and state heals very slow.

Modern Geth (since ~2022) uses a **path-based scheme** alongside the hash-based one:

```
Path-based key = "a" ++ keccak256(address)              (for accounts)
               = "h" ++ keccak256(address) ++ keccak256(slot)   (for storage)
Value = RLP-encoded data
```

Now you can iterate all accounts by doing a range scan over `"a" ++ 0x00...` to `"a" ++ 0xff...`. This is dramatically faster for state sync, snapshots, and historical queries. The hash-based nodes are still kept for Merkle proof verification, but the path-based nodes are the primary storage.

### How Account and Storage Tries Are Stored

Geth stores them separately, similar to EELS, but on disk:

**Account trie:**

```
Key = keccak256(address) → hashed to distribute uniformly
Value = RLP([nonce, balance, storageRoot, codeHash])
```

**Storage trie (one per contract):**

```
Key = keccak256(slot) → hashed
Value = RLP(value)
```

The `storageRoot` in the account's RLP is the root hash of that contract's storage trie. It links the two tries cryptographically.

### EELS vs. Geth: The Core Difference

| Aspect | EELS | Geth |
|--------|------|------|
| **Trie lifetime** | Rebuilt from scratch on every `root()` call | Kept alive across blocks, mutated in place |
| **Storage** | In-memory Python dict | LevelDB / Pebble on disk |
| **Persistence** | None | Every node persisted as `(hash, RLP(node))` |
| **Mutations** | `trie_set` modifies a flat dict, then rebuild | Dirty nodes marked, hashed, batch-committed |
| **History** | None — only current state | Diff layers + archive nodes store all historical states |
| **Iteration** | Walk the Python dict | Range scans over path-based keys |
| **Purpose** | Specification, verification, prototyping | Production client serving JSON-RPC and P2P |

---

## 11. How does Geth find data when a user queries an account balance?

### The Query Arrives

```
User → JSON-RPC → eth_getBalance("0xd73be913704abA45a9926E549F60db05C93Da54e", "latest")
```

Geth receives the address and the target block ("latest" = current state).

### Step 1: Hash the Address

Geth computes the trie key:

```python
hashed_key = keccak256(0xd73be913704abA45a9926E549F60db05C93Da54e)
# = some 32-byte hash, e.g. 0xa711355f...
```

This is the same `secured` key that EELS uses. The trie doesn't store accounts by raw address — it stores them by `keccak256(address)`.

### Step 2: Check Diff Layers (Fast Path)

If the query is for `"latest"` or a recent block (within the last ~128 blocks), Geth **doesn't touch LevelDB at all**. It checks the in-memory diff layers first:

```
Diff Layer 1 (current block, in memory)
    → Was this account modified? Yes → return cached version
    → No → fall through to Diff Layer 2
        → ...
            → Not found → fall through to disk layer
```

This is why querying recent state is fast — most accounts are served from RAM.

### Step 3: Disk Lookup — Two Modes

Modern Geth maintains **two parallel storage schemes**. The query hits whichever is available.

#### Path-based mode (the fast path, modern default)

Geth has a secondary index that maps paths directly to data on disk:

```
LevelDB key = "a" ++ hashed_address
              ^-- prefix: "account"
                 ^-- the trie key

LevelDB value = RLP([nonce, balance, storageRoot, codeHash])
```

Geth does a **single direct lookup**:

```python
data = db.Get(b"a" + hashed_address)
# → one disk read → b'\xf8u\x05\x88\x1b\xe14\xa7e\x80\x00...'

account = rlp.decode(data)
# → [nonce=5, balance=2000000000000000000, storageRoot=..., codeHash=...]

return account[1]   # balance = 2 ETH
```

One disk read. Done. No trie traversal needed.

#### Hash-based mode (the traditional path, trie walk)

In hash-based mode, Geth doesn't know where the account lives on disk — it only knows the **root hash** of the state trie. It has to **walk the trie** from the root down to the leaf.

The root hash comes from the latest block header:

```
stateRoot = 0x5f4e2a8b...   (from block header)
```

Now Geth fetches the root node:

```
LevelDB key = 0x5f4e2a8b...   (the root hash itself)
Value = RLP(root node)
```

It decodes the RLP. Suppose the root is a Branch node:

```
BranchNode.subnodes = [b"", b"", hashC, ..., hashA, ..., b""]
```

Geth looks at the **first nibble** of `hashed_address`:

```
hashed_address nibbles: [10, 7, 1, 1, 3, 5, ...]
                          ^^
                       first nibble = 10 (0xa)
```

It follows slot `10` of the Branch. Suppose slot 10 contains `hashD`. Geth fetches that node:

```
LevelDB key = hashD
Value = RLP(ExtensionNode([10, 7], hashE))
```

The extension says "the next two nibbles are `10, 7`". Geth checks: does `hashed_address` have nibbles `[10, 7]` at position 0? Yes. It consumes them and moves to the child `hashE`.

```
LevelDB key = hashE
Value = RLP(BranchNode(...))
```

Another Branch. Next nibble of `hashed_address` is `1`. Follow slot 1. Continue until reaching a Leaf node.

Eventually Geth reaches:

```
LevelDB key = hashLeaf
Value = RLP((compact_path, RLP([nonce, balance, storageRoot, codeHash])))
```

It decodes the leaf, extracts the account RLP, decodes that, and reads the balance.

This takes **5–10 disk reads** for a typical account, one per trie level.

### Step 4: Return the Balance

Once Geth has the decoded account:

```python
balance = account.balance   # U256(2000000000000000000)
```

It formats it as a hex string and returns it over JSON-RPC:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "0x1bc16d674ec80000"   // 2 ETH in wei
}
```

### Summary

| Mode | Disk reads | How it works |
|------|-----------|--------------|
| **Diff layer** | 0 | In-memory cache, no disk access, for recent blocks |
| **Path-based** | 1 | Direct lookup: `db["a" + hashed_address]` |
| **Hash-based** | 5–10 | Walk the trie from root, following nibbles |

Path-based mode is essentially an **index** over the trie. It trades extra disk space (storing both path-based and hash-based copies) for dramatically faster lookups and the ability to iterate all accounts efficiently. Hash-based mode is the "pure" Merkle trie approach — slower for lookups but cryptographically self-contained (you can verify any node by its hash without trusting the database).