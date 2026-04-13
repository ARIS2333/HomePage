---
title: EELS(5) Q&A Notes
published: 2026-04-03
pinned: false
description: Some QA that might be helpful.
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

1. [What are the five transaction types and when do you use each?](#1-what-are-the-five-transaction-types-and-when-do-you-use-each)
2. [Are access lists manually constructed by users or automatically generated?](#2-are-access-lists-manually-constructed-by-users-or-automatically-generated)
3. [How does `process_transaction` work — line by line?](#3-how-does-process_transaction-work--line-by-line)
4. [How does the gas flow work from upfront payment to final distribution?](#4-how-does-the-gas-flow-work-from-upfront-payment-to-final-distribution)

---

## 1. What are the five transaction types and when do you use each?

Ethereum defines five transaction formats, each introduced by a distinct EIP. All types are defined as a union in `transactions.py`:

```python
Transaction = (
    LegacyTransaction        # Type 0 — Frontier
    | AccessListTransaction  # Type 1 — EIP-2930 (Berlin)
    | FeeMarketTransaction   # Type 2 — EIP-1559 (London)
    | BlobTransaction        # Type 3 — EIP-4844 (Cancun)
    | SetCodeTransaction     # Type 4 — EIP-7702 (Prague)
)
```

Evolution timeline:

```
Legacy (Frontier)
  └─> EIP-2930 (Berlin):  Access lists + explicit chain ID
      └─> EIP-1559 (London): Fee market (base fee + priority fee)
          ├─> EIP-4844 (Cancun): Blob-carrying transactions
          └─> EIP-7702 (Prague): EOA code delegation
```

---

### Type 0: Legacy Transaction

```python
@slotted_freezable
@dataclass
class LegacyTransaction:
    nonce: U256
    gas_price: Uint          # single price — covers BOTH base fee + tip
    gas: Uint
    to: Bytes0 | Address     # Bytes0(b"") = contract creation
    value: U256
    data: Bytes
    v: U256                  # recovery id (27/28 or EIP-155 encoded)
    r: U256
    s: U256
```

**Wire encoding:** RLP-encoded directly, no type prefix.

**When you'd use it:** Almost never on mainnet today. Legacy transactions still work — the protocol never removes old formats — but you'd only see them from:
- Very old wallets or libraries that haven't been updated
- Hardware wallets with outdated firmware
- Test environments where simplicity matters more than gas optimization

**Key limitation:** You bid one fixed `gas_price`. If the base fee drops during the block, you overpay. If it spikes, your transaction gets stuck. No safety net.

**Signing hashes:**
```python
# Pre-EIP-155 (v = 27 or 28)
signing_hash = keccak256(rlp.encode(
    (nonce, gas_price, gas, to, value, data)
))

# EIP-155 (v = chain_id × 2 + 35 or 36)
signing_hash = keccak256(rlp.encode(
    (nonce, gas_price, gas, to, value, data, chain_id, Uint(0), Uint(0))
))
```

---

### Type 1: Access List Transaction (EIP-2930)

```python
@slotted_freezable
@dataclass
class AccessListTransaction:
    chain_id: U64                    # explicit chain ID
    nonce: U256
    gas_price: Uint                  # still single price (pre-EIP-1559)
    gas: Uint
    to: Bytes0 | Address
    value: U256
    data: Bytes
    access_list: Tuple[Access, ...]  # pre-warmed addresses and slots
    y_parity: U256                   # normalized recovery bit (0 or 1)
    r: U256
    s: U256
```

**Wire encoding:** `0x01 || rlp([chain_id, nonce, gas_price, gas, to, value, data, access_list, y_parity, r, s])`

**New fields over Legacy:**

- `chain_id`: Explicit chain identifier, replacing the implicit EIP-155 encoding in `v`. Prevents cross-chain replay without relying on signature encoding tricks.
- `access_list`: Tuple of `Access` objects declaring addresses and storage slots the transaction will touch. Slots declared here are loaded into the warm access set before execution begins, reducing their per-access gas cost (2,400 gas per warm address vs. 2,600 cold; 1,900 per warm slot vs. 2,100 cold).
- `y_parity`: Normalized signature recovery bit (`0` or `1`), replacing the legacy `v` value.

**When you'd use it:** Rarely on its own. Almost nobody sends pure Type 1 transactions. The access list concept is used *internally* by Types 2, 3, and 4, but the user never constructs a pure Type 1 tx. It exists as a stepping stone in the evolution.

---

### Type 2: Fee Market Transaction (EIP-1559)

```python
@slotted_freezable
@dataclass
class FeeMarketTransaction:
    chain_id: U64
    nonce: U256
    max_priority_fee_per_gas: Uint   # tip to validator
    max_fee_per_gas: Uint            # max total per gas
    gas: Uint
    to: Bytes0 | Address
    value: U256
    data: Bytes
    access_list: Tuple[Access, ...]
    y_parity: U256
    r: U256
    s: U256
```

**Wire encoding:** `0x02 || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas, to, value, data, access_list, y_parity, r, s])`

**New fields over Type 1:**

- `max_priority_fee_per_gas`: The tip the sender is willing to pay to the validator, on top of the base fee.
- `max_fee_per_gas`: The maximum total fee per gas unit (base fee + tip) the sender will accept. If the block's `base_fee_per_gas` exceeds this, the transaction is excluded.

**The effective gas price actually charged:**

```python
priority_fee_per_gas = min(
    tx.max_priority_fee_per_gas,
    tx.max_fee_per_gas - block_env.base_fee_per_gas,
)
effective_gas_price = priority_fee_per_gas + block_env.base_fee_per_gas
```

The base fee is **burned**; the priority fee goes to the validator. Any difference between `max_fee_per_gas` and `effective_gas_price` is **refunded** to the sender.

**When you'd use it:** **This is the default for almost all users today.** MetaMask, Rainbow, and every modern wallet send Type 2 by default for normal transactions — sending ETH, swapping on Uniswap, minting an NFT, interacting with any contract.

**How the pricing actually works in practice:**

Say the block's `base_fee_per_gas` is 20 gwei. You set:
- `max_fee_per_gas = 30` gwei
- `max_priority_fee_per_gas = 1.5` gwei

The effective price you pay:

```
priority_fee = min(1.5, 30 - 20) = min(1.5, 10) = 1.5 gwei
effective_gas_price = 1.5 + 20 = 21.5 gwei
```

You pay 21.5 gwei per gas. The 20 gwei base fee is burned. The 1.5 gwei tip goes to the validator. The remaining 8.5 gwei (`30 - 21.5`) of your max fee is refunded to you.

**Why this is better than Type 0:**

| Situation | Type 0 (Legacy) | Type 2 (EIP-1559) |
|-----------|-----------------|-------------------|
| Base fee spikes to 25 gwei | Your 20 gwei tx gets stuck forever | Still works — you set max 30 gwei |
| Base fee drops to 5 gwei | You still pay 20 gwei (waste) | You only pay 5 + 1.5 = 6.5 gwei (refund) |
| Network congestion | You guess wrong, overpay or get stuck | `max_fee` protects you, `priority_fee` ensures inclusion |

---

### Type 3: Blob Transaction (EIP-4844)

```python
@slotted_freezable
@dataclass
class BlobTransaction:
    chain_id: U64
    nonce: U256
    max_priority_fee_per_gas: Uint
    max_fee_per_gas: Uint
    gas: Uint
    to: Address                    # contract creation NOT allowed
    value: U256
    data: Bytes
    access_list: Tuple[Access, ...]
    max_fee_per_blob_gas: U256     # independent blob gas price
    blob_versioned_hashes: Tuple[VersionedHash, ...]
    y_parity: U256
    r: U256
    s: U256
```

**Wire encoding:** `0x03 || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas, to, value, data, access_list, max_fee_per_blob_gas, blob_versioned_hashes, y_parity, r, s])`

**New fields over Type 2:**

- `max_fee_per_blob_gas`: The maximum blob gas price per unit of blob gas the sender is willing to pay. Blob gas is priced independently of execution gas via a separate exponential fee market.
- `blob_versioned_hashes`: KZG commitment hashes for the blobs attached to this transaction. Each hash has a one-byte version prefix (`0x01` for KZG). The actual blob data is propagated over the P2P layer and is not stored on-chain.

**Constraints:**
- `to` must be an `Address`; contract creation is disallowed.
- At least one blob versioned hash must be present.
- The blob count must not exceed `BLOB_COUNT_LIMIT = 6` (Osaka).
- Every versioned hash must start with `VERSIONED_HASH_VERSION_KZG = b"\x01"`.
- `max_fee_per_blob_gas` must be at least the current `blob_gas_price`.

**When you'd use it:** **Almost exclusively by L2 rollup sequencers** — Optimism, Arbitrum, Base, zkSync, Scroll, etc. Regular users almost never send Type 3 transactions directly.

**The L2 workflow:**

1. L2 sequencer collects hundreds of L2 user transactions off-chain.
2. It compresses them into a batch (say, 100 KB of compressed data).
3. It attaches 1 blob (128 KB capacity) to a Type 3 transaction and sends it to the L1 inbox contract.
4. The L1 EVM executes the call. The `BLOBHASH(0)` opcode reads the 32-byte versioned hash from the tx. The inbox contract stores this hash on-chain as a permanent anchor.
5. The raw 128 KB blob data is never stored on-chain permanently. It lives in the consensus layer for ~18 days, then gets pruned.
6. L2 users can later prove that specific data was in the blob using the point-evaluation precompile.

**Cost comparison vs. calldata:**

| Metric | Using calldata (pre-EIP-4844) | Using blobs (EIP-4844) |
|--------|------------------------------|----------------------|
| Cost per byte | ~16 gas (non-zero) or 4 gas (zero) | ~1 wei per blob gas unit |
| Data permanence | Stored forever | Pruned after ~18 days |
| Cost for 100 KB batch | ~$50-200 depending on congestion | ~$0.10-2 |

---

### Type 4: Set Code Transaction (EIP-7702)

```python
@slotted_freezable
@dataclass
class SetCodeTransaction:
    chain_id: U64
    nonce: U64              # Note: U64, not U256
    max_priority_fee_per_gas: Uint
    max_fee_per_gas: Uint
    gas: Uint
    to: Address             # contract creation NOT allowed
    value: U256
    data: Bytes
    access_list: Tuple[Access, ...]
    authorizations: Tuple[Authorization, ...]
    y_parity: U256
    r: U256
    s: U256
```

**Wire encoding:** `0x04 || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas, to, value, data, access_list, authorizations, y_parity, r, s])`

**New field over Type 2:**

- `authorizations`: One or more `Authorization` objects. Each authorization is signed separately by an EOA that wishes to delegate its execution to a target contract. Upon processing, the delegating account's code is set to the delegation designator `0xef0100 || address` (23 bytes), causing calls to that EOA to execute the target contract's code in the EOA's context.

**The `Authorization` struct:**

```python
@slotted_freezable
@dataclass
class Authorization:
    chain_id: U256
    address: Address   # Target contract to delegate to
    nonce: U64
    y_parity: U256
    r: U256
    s: U256
```

Each `Authorization` is **signed by a different EOA** — not by the transaction sender. This means the transaction sender can say "these three EOAs will delegate to this contract" and include their pre-signed authorizations.

**When you'd use it:** Two main scenarios:

**Scenario 1: Account abstraction for EOAs.** Alice has a regular EOA (no smart contract wallet). She wants smart wallet features — spending limits, social recovery, batched transactions. She sends a Type 4 transaction with an `Authorization` pointing to a smart wallet contract. Her EOA's `code_hash` is set to `keccak256(0xef0100 || smart_wallet_address)`. From that point on, when someone calls Alice's address, the smart wallet contract's code runs in her EOA's context.

**Scenario 2: Single-transaction delegation.** Bob wants to do a complex DeFi operation that requires a specific contract's execution context, but only for this one transaction. He includes an `Authorization` in his Type 4 tx. The delegation is processed, the contract code runs, and Bob can clear the delegation afterward.

**Constraints:**
- `to` must be an `Address`; contract creation is disallowed.
- `authorizations` must contain at least one entry.

---

### Transaction Type Comparison

| Feature | Legacy (0) | EIP-2930 (1) | EIP-1559 (2) | EIP-4844 (3) | EIP-7702 (4) |
|---------|------------|--------------|--------------|--------------|--------------|
| Explicit chain ID | ✅ (in `v`) | ✅ | ✅ | ✅ | ✅ |
| Access list | ❌ | ✅ | ✅ | ✅ | ✅ |
| EIP-1559 fee fields | ❌ | ❌ | ✅ | ✅ | ✅ |
| Blob versioned hashes | ❌ | ❌ | ❌ | ✅ | ❌ |
| Authorizations | ❌ | ❌ | ❌ | ❌ | ✅ |
| Contract creation | ✅ | ✅ | ✅ | ❌ | ❌ |

### Quick Decision Guide

| What are you doing? | Use Type |
|---------------------|----------|
| Send ETH, call any contract, normal wallet usage | **Type 2** (default) |
| L2 sequencer posting batched data to L1 | **Type 3** |
| Give your EOA smart wallet features temporarily | **Type 4** |
| Interacting with a very old wallet that only supports legacy | **Type 0** (rare) |
| You specifically want access list optimization without EIP-1559 | **Type 1** (almost never) |

---

## 2. Are access lists manually constructed by users or automatically generated?

They're almost always **automatically constructed by wallets and RPC nodes**, not manually by users. But let's break down exactly how this works.

### The Reality: Nobody Writes Access Lists by Hand

Think about what an access list entry looks like for a typical DeFi transaction:

```python
access_list=(
    Access(
        account=Address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2),  # WETH
        slots=(
            Bytes32(0x5...),  # some specific storage slot
            Bytes32(0x8...),  # another slot
        ),
    ),
    Access(
        account=Address(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48),  # USDC
        slots=(
            Bytes32(0x2...),
            Bytes32(0x9...),
            Bytes32(0xc...),
        ),
    ),
    # ... and so on for every contract touched
)
```

A single Uniswap swap might touch 5-10 contracts, each with 3-10 storage slots. That's dozens of addresses and hundreds of slot hashes. **No human is typing this out.**

### How Access Lists Are Actually Constructed

**Method 1: Wallet auto-generation (most common).** When you submit a transaction in MetaMask or similar, the wallet does a **simulation** first:

1. The wallet calls `eth_createAccessList` (an RPC method) on the node, passing the unsigned transaction parameters (to, data, value, etc.).
2. The node **simulates** the transaction against the current state — it runs the EVM code without committing any changes.
3. During simulation, the node records every address and storage slot the transaction touches.
4. The node returns an optimal access list.
5. The wallet includes this access list in the signed transaction (Type 2).

The user never sees the access list. It's an invisible optimization happening behind the scenes.

**Method 2: Builder/relayer construction.** For MEV-protected transactions (submitted via Flashbots or similar), the **block builder** constructs the access list. The builder already simulates all transactions in the block to optimize ordering and profit. Adding access list construction to that simulation is trivial.

**Method 3: Empty access list (most common in practice).** Here's the thing: **most wallets just send an empty access list** and don't bother. Why?

Because the gas savings from an access list are relatively small for simple transactions. A basic ETH transfer touches no contracts — the access list would be empty anyway. A simple DEX swap might save 5,000-15,000 gas from warm access, but that's only ~$0.10-$0.50 at current gas prices. The overhead of simulating and constructing the access list often isn't worth it for the wallet provider.

### The Cost-Benefit Math

Including an address in the access list **costs** 2,400 gas. Including a storage slot **costs** 1,900 gas. You only save money if the transaction accesses those addresses/slots **more than once** during execution.

| Scenario | Without access list | With access list | Net savings |
|----------|-------------------|-----------------|-------------|
| Simple ETH transfer | 21,000 gas | 21,000 gas (empty list) | 0 |
| Single contract call, accessed once | 2,600 gas (cold access) | 2,400 (list entry) + 100 (warm) = 2,500 gas | 100 gas |
| Single contract call, accessed 3 times | 2,600 + 100 + 100 = 2,800 gas | 2,400 + 100 × 3 = 2,700 gas | 100 gas |
| Contract accessed 10 times | 2,600 + 100 × 9 = 3,500 gas | 2,400 + 100 × 10 = 3,400 gas | 100 gas |

The savings are **per address**: you pay 2,400 once to warm it, then save 2,500 on the first access (2,600 cold → 100 warm). Net savings per address = **100 gas**. For storage slots, you pay 1,900 once, save 2,000 on first access. Net savings per slot = **100 gas**.

So the access list only matters when you touch **many** different addresses and slots. A complex DeFi transaction touching 10 contracts and 50 slots saves:

```
10 addresses × 100 + 50 slots × 100 = 6,000 gas saved
```

At 20 gwei, that's ~$0.25. Not nothing, but not transformative.

### Why Include Access Lists in the Protocol at All?

If the savings are so small, why does every transaction type since Type 1 carry an access list?

1. **It enables Type 2 (EIP-1559) transactions.** The access list was introduced in EIP-2930 (Berlin) and then EIP-1559 (London) built on top of the Type 1 envelope. Type 2 is essentially Type 1 + fee market fields. The access list is there because it was already part of the envelope.

2. **Complex transactions benefit more.** L2 batch submissions, multi-hop DeFi routes, and contract deployments touch many addresses. The savings compound.

3. **It's future-proof.** As contracts become more complex, the access list becomes more valuable. The infrastructure is already in place.

---

## 3. How does `process_transaction` work — line by line?

### The Scenario

```
Alice (0xaaaa...) sends 1 ETH to Bob (0xbbbb...) via a Type 2 (EIP-1559) transaction.

Transaction:
  nonce = 5
  max_priority_fee_per_gas = 1.5 gwei
  max_fee_per_gas = 30 gwei
  gas = 21,000
  to = Bob's address
  value = 1 ETH
  data = b""
  access_list = ()
```

### Lines 1-4: Function signature

```python
def process_transaction(
    block_env: vm.BlockEnvironment,
    block_output: vm.BlockOutput,
    tx: Transaction,
    index: Uint,
) -> None:
```

Four inputs:
- `block_env`: Block-scoped constants (base fee, coinbase, block number, etc.)
- `block_output`: Accumulator being filled (transaction trie, receipt trie, logs, gas counters)
- `tx`: The decoded transaction
- `index`: Position in the block (0, 1, 2, ...)

---

### Lines 5-6: Insert transaction into the transactions trie

```python
trie_set(
    block_output.transactions_trie,
    rlp.encode(index),
    encode_transaction(tx),
)
```

The transaction is immediately added to the output's transaction trie at key `rlp.encode(Uint(0))` (for the first tx). This is an incremental build — by the end of the block, the trie root must match the header's `transactions_root`.

---

### Lines 7-8: Static validation

```python
intrinsic_gas, calldata_floor_gas_cost = validate_transaction(tx)
```

This runs state-independent checks:
- Does `tx.gas` cover the intrinsic cost? For a simple ETH transfer: `GAS_TX_BASE = 21,000` gas. No calldata, no access list, no creation → `intrinsic_gas = 21,000`.
- Is the nonce < 2^64 - 1?
- Is the gas limit within Osaka's `TX_MAX_GAS_LIMIT = 16,777,216`?

Returns: `intrinsic_gas = 21,000`, `calldata_floor_gas_cost = 21,000` (base 21,000 + 0 calldata tokens).

---

### Lines 9-15: Dynamic validation

```python
(
    sender,
    effective_gas_price,
    blob_versioned_hashes,
    tx_blob_gas_used,
) = check_transaction(
    block_env=block_env,
    block_output=block_output,
    tx=tx,
)
```

This runs state-dependent checks:
1. **Block gas available**: Does the tx fit in the remaining block gas?
2. **Blob gas available**: (Not applicable for Type 2)
3. **Recover sender**: ECDSA recovery from `(r, s, y_parity)` → `0xaaaa...`
4. **Fee validation**: `max_fee_per_gas (30) ≥ base_fee (20)` ✓. `max_priority_fee (1.5) ≤ max_fee (30)` ✓.
5. **Blob validation**: (Skipped for Type 2)
6. **Contract creation check**: (Not applicable — `to` is an Address)
7. **Authorization check**: (Not applicable — not Type 4)
8. **Nonce check**: `sender_account.nonce (5) == tx.nonce (5)` ✓
9. **Balance check**: Does Alice have enough for `gas × max_fee + value = 21,000 × 30 gwei + 1 ETH`?
10. **Sender type check**: Is the sender an EOA (or delegated EOA)? ✓

Returns:
- `sender = Address(0xaaaa...)`
- `effective_gas_price = 21.5 gwei` (base 20 + tip 1.5)
- `blob_versioned_hashes = ()` (empty for Type 2)
- `tx_blob_gas_used = 0`

---

### Line 16: Fetch sender account

```python
sender_account = get_account(block_env.state, sender)
```

Gets Alice's full account: `Account(nonce=5, balance=5ETH, code_hash=EMPTY)`.

---

### Lines 17-20: Blob fee calculation

```python
if isinstance(tx, BlobTransaction):
    blob_gas_fee = calculate_data_fee(block_env.excess_blob_gas, tx)
else:
    blob_gas_fee = Uint(0)
```

Not a blob tx → `blob_gas_fee = 0`.

---

### Line 21: Effective gas fee

```python
effective_gas_fee = tx.gas * effective_gas_price
```

```
effective_gas_fee = 21,000 × 21.5 gwei = 451,500,000,000,000 wei = 0.0004515 ETH
```

This is the **maximum** amount of ETH that will be deducted from Alice for gas.

---

### Line 22: Gas available for EVM execution

```python
gas = tx.gas - intrinsic_gas
```

```
gas = 21,000 - 21,000 = 0
```

All 21,000 gas was consumed as intrinsic cost. Zero gas remains for the EVM bytecode loop. This is correct for a simple ETH transfer — there's no contract code to execute, just a balance transfer.

---

### Line 23: Increment sender nonce

```python
increment_nonce(block_env.state, sender)
```

Alice's nonce goes from 5 → 6. This happens **before** execution, preventing replay attacks and ensuring the nonce is consumed even if the tx reverts.

---

### Lines 24-28: Deduct gas fee from sender

```python
sender_balance_after_gas_fee = (
    Uint(sender_account.balance) - effective_gas_fee - blob_gas_fee
)
set_account_balance(
    block_env.state, sender, U256(sender_balance_after_gas_fee)
)
```

```
Alice's balance before: 5 ETH
Minus gas fee:          - 0.0004515 ETH
Minus blob fee:         - 0
Alice's balance now:    4.9995485 ETH
```

The **entire** `effective_gas_fee` (upfront max gas cost) is deducted now. Any unused portion will be refunded after execution.

---

### Lines 29-35: Build warm access sets

```python
access_list_addresses = set()
access_list_storage_keys = set()
access_list_addresses.add(block_env.coinbase)
if has_access_list(tx):
    for access in tx.access_list:
        access_list_addresses.add(access.account)
        for slot in access.slots:
            access_list_storage_keys.add((access.account, slot))
```

Start with empty sets. Always add the **coinbase** (block proposer) — EIP-3651 ensures the fee recipient is warm.

If the tx has an access list (it doesn't in our case), add each address and slot to the warm sets.

For our simple tx:
```
access_list_addresses = {coinbase}
access_list_storage_keys = set()
```

---

### Lines 36-38: Authorization handling (Type 4 only)

```python
authorizations: Tuple[Authorization, ...] = ()
if isinstance(tx, SetCodeTransaction):
    authorizations = tx.authorizations
```

Not a Type 4 tx → `authorizations = ()`.

---

### Lines 39-50: Build TransactionEnvironment

```python
tx_env = vm.TransactionEnvironment(
    origin=sender,
    gas_price=effective_gas_price,
    gas=gas,
    access_list_addresses=access_list_addresses,
    access_list_storage_keys=access_list_storage_keys,
    transient_storage=TransientStorage(),
    blob_versioned_hashes=blob_versioned_hashes,
    authorizations=authorizations,
    index_in_block=index,
    tx_hash=get_transaction_hash(encode_transaction(tx)),
)
```

This bundles all transaction-level context:

| Field | Value |
|-------|-------|
| `origin` | `0xaaaa...` (Alice — fixed across all call frames) |
| `gas_price` | `21.5 gwei` (effective price) |
| `gas` | `0` (remaining after intrinsic deduction) |
| `access_list_addresses` | `{coinbase}` |
| `access_list_storage_keys` | `set()` |
| `transient_storage` | Empty `TransientStorage()` (cleared after this tx) |
| `blob_versioned_hashes` | `()` |
| `authorizations` | `()` |
| `index_in_block` | `Uint(0)` |
| `tx_hash` | `keccak256(0x02 || rlp(tx_payload))` |

---

### Line 51: Build the Message

```python
message = prepare_message(block_env, tx_env, tx)
```

This constructs the `Message` object that the EVM will execute. It resolves:
- `caller = origin = 0xaaaa...`
- `target = Bob's address`
- `value = 1 ETH`
- `data = b""`
- `code = b""` (Bob is an EOA, no code)
- `gas = 0` (remaining after intrinsic)

---

### Line 52: Execute the EVM

```python
tx_output = process_message_call(message)
```

This enters the EVM interpreter. For our simple transfer:

1. `message.target` is an `Address` (not `Bytes0`), so it takes the **call path**.
2. No `authorizations` → skip `set_delegation`.
3. No delegated code → skip delegation resolution.
4. `process_message` runs the bytecode loop. Bob has no code → the loop immediately terminates (`evm.running` is `True` but `evm.pc (0) >= ulen(code) (0)`).
5. The value transfer happens **before** EVM execution in `process_message`:

```python
# Inside process_message:
if message.should_transfer_value and message.value != 0:
    move_ether(state, message.caller, message.current_target, message.value)
```

So 1 ETH moves from Alice to Bob here. This happens inside the snapshot boundary (`begin_transaction` was called inside `process_message`), so if anything goes wrong, it reverts.

6. The `tx_output` is returned:
```python
tx_output = MessageCallOutput(
    gas_left=0,
    refund_counter=0,
    logs=(),
    accounts_to_delete=set(),
    error=None,
    return_data=b"",
)
```

---

### Lines 53-56: Calculate gas used before refund

```python
tx_gas_used_before_refund = tx.gas - tx_output.gas_left
tx_gas_refund = min(
    tx_gas_used_before_refund // Uint(5), Uint(tx_output.refund_counter)
)
tx_gas_used_after_refund = tx_gas_used_before_refund - tx_gas_refund
```

```
tx_gas_used_before_refund = 21,000 - 0 = 21,000
tx_gas_refund = min(21,000 // 5, 0) = min(4,200, 0) = 0
tx_gas_used_after_refund = 21,000 - 0 = 21,000
```

EIP-3529 caps the refund at 1/5 of gas used. In our case there's no refund because no `SSTORE` clearing happened.

---

### Lines 57-60: EIP-7623 calldata floor

```python
tx_gas_used_after_refund = max(
    tx_gas_used_after_refund, calldata_floor_gas_cost
)
```

```
tx_gas_used_after_refund = max(21,000, 21,000) = 21,000
```

The calldata floor (EIP-7623, Osaka) ensures the tx pays at least the floor cost. In this case they're equal — no adjustment needed.

---

### Line 61: Gas left over

```python
tx_gas_left = tx.gas - tx_gas_used_after_refund
```

```
tx_gas_left = 21,000 - 21,000 = 0
```

No unused gas → no refund.

---

### Line 62: Refund amount

```python
gas_refund_amount = tx_gas_left * effective_gas_price
```

```
gas_refund_amount = 0 × 21.5 gwei = 0
```

---

### Lines 64-65: Priority fee extraction

```python
priority_fee_per_gas = effective_gas_price - block_env.base_fee_per_gas
transaction_fee = tx_gas_used_after_refund * priority_fee_per_gas
```

```
priority_fee_per_gas = 21.5 - 20 = 1.5 gwei
transaction_fee = 21,000 × 1.5 gwei = 31,500,000,000,000 wei = 0.0000315 ETH
```

This is the **tip** that goes to the block proposer. The base fee portion (`21,000 × 20 gwei = 0.00042 ETH`) was already burned when we deducted the full `effective_gas_fee` upfront.

---

### Lines 67-70: Refund unused gas to sender

```python
sender_balance_after_refund = get_account(
    block_env.state, sender
).balance + U256(gas_refund_amount)
set_account_balance(block_env.state, sender, sender_balance_after_refund)
```

```
Alice's balance: 3.9995485 ETH + 0 = 3.9995485 ETH (unchanged)
```

If there had been unused gas, this would have added it back.

---

### Lines 72-77: Pay the coinbase (block proposer)

```python
coinbase_balance_after_mining_fee = get_account(
    block_env.state, block_env.coinbase
).balance + U256(transaction_fee)
set_account_balance(
    block_env.state,
    block_env.coinbase,
    coinbase_balance_after_mining_fee,
)
```

```
Coinbase balance: X + 0.0000315 ETH
```

The proposer gets the priority fee (tip). The base fee is already burned — it was deducted from Alice but never credited to anyone.

---

### Lines 78-79: Delete accounts marked for destruction

```python
for address in tx_output.accounts_to_delete:
    destroy_account(block_env.state, address)
```

`tx_output.accounts_to_delete` is empty (no `SELFDESTRUCT` in this tx). If there were, each account's balance, code, and storage would be wiped.

---

### Lines 80-81: Update block-level counters

```python
block_output.block_gas_used += tx_gas_used_after_refund
block_output.blob_gas_used += tx_blob_gas_used
```

```
block_output.block_gas_used = 0 + 21,000 = 21,000
block_output.blob_gas_used = 0 + 0 = 0
```

These accumulate across all transactions in the block.

---

### Lines 82-84: Create the receipt

```python
receipt = make_receipt(
    tx, tx_output.error, block_output.block_gas_used, tx_output.logs
)
```

```python
Receipt(
    succeeded=True,          # error is None
    cumulative_gas_used=21,000,
    bloom=logs_bloom(()),    # empty bloom (no logs)
    logs=(),
)
```

Then `encode_receipt` wraps it with the Type 2 prefix: `b"\x02" + rlp.encode(receipt)`.

---

### Lines 85-86: Record receipt key

```python
receipt_key = rlp.encode(Uint(index))
block_output.receipt_keys += (receipt_key,)
```

The key is `rlp.encode(Uint(0)) = b"\x00"`. This tuple of keys is used later by `parse_deposit_requests` which iterates receipt trie entries.

---

### Lines 87-91: Insert receipt into the receipts trie

```python
trie_set(
    block_output.receipts_trie,
    receipt_key,
    receipt,
)
```

The receipt is stored at key `b"\x00"` in the receipts trie. Like the transaction trie, this is incrementally built — by end of block its root must match the header's `receipt_root`.

---

### Line 92: Append logs to block-wide log list

```python
block_output.block_logs += tx_output.logs
```

Empty in our case. For a tx that emits events (e.g., a `Transfer` log from an ERC-20), these would be appended here. The full list is used to build the block's bloom filter.

---

## 4. How does the gas flow work from upfront payment to final distribution?

Here's the complete gas flow diagram with detailed explanation at each step.

### Step 1: Sender Upfront Payment

```
Sender Upfront Payment: gas_limit × effective_gas_price
```

Before the EVM executes **anything**, the sender pays the maximum possible gas cost upfront.

**Why upfront?** The protocol can't know in advance how much gas a transaction will actually consume. A contract call might hit a `REVERT` on line 3 or run 5,000 steps. The only safe approach is to collect the maximum upfront, then refund what wasn't used.

**Concrete example:**
```
gas_limit = 100,000
effective_gas_price = 21.5 gwei
upfront_payment = 100,000 × 21.5 gwei = 0.00215 ETH
```

This amount is **deducted from the sender's balance immediately**, before any EVM code runs. The sender doesn't lose this permanently — most of it will come back as a refund — but it's locked during execution.

---

### Step 2: EVM Execution

```
                       EVM Execution
```

The transaction runs. The EVM consumes gas at every opcode. Some gas might be added to the refund counter (e.g., `SSTORE` clearing a slot).

**Two paths emerge from here:**

---

### Step 3: Success vs. Revert

```
               ┌─────────────┴─────────────┐
               ▼                           ▼
            Success                      Revert
       (State Committed)           (State Rolled Back)
```

#### Success path

The transaction completed without error. All state changes are kept.

```
gas_used = gas_limit - gas_left
```

**Example:**
```
gas_limit = 100,000
gas_left (unused) = 30,000
gas_used = 100,000 - 30,000 = 70,000
```

70,000 gas was consumed. 30,000 is sitting unused, eligible for refund.

#### Revert path

The transaction hit an error — explicit `REVERT`, `OutOfGasError`, failed assertion, etc.

```
gas_used = gas_limit
```

**All gas is consumed.** Nothing is refunded. This is the protocol's penalty for failed transactions: you pay for the computation you wasted, even though the state changes are rolled back.

```
gas_left = 0
gas_refund = 0
```

The sender's upfront payment is fully consumed. No refund, no priority fee goes to the validator (since there's no successful work to reward).

---

### Step 4: Refund Calculation (Success Path Only)

```
       Apply Refund (≤ 1/5)
       Apply Calldata Floor
```

On the success path, two adjustments happen to `gas_used`.

#### Adjustment 1: SSTORE refund cap (EIP-3529)

If the transaction cleared storage slots (via `SSTORE`), it accumulated refund credits during execution. But the refund is **capped at 1/5 of gas used**:

```python
tx_gas_refund = min(gas_used // 5, refund_counter)
gas_used_after_refund = gas_used - tx_gas_refund
```

**Example with refunds:**
```
gas_used before refund = 70,000
refund_counter (from SSTORE clears) = 19,200
max allowed refund = 70,000 // 5 = 14,000
actual refund = min(14,000, 19,200) = 14,000  ← capped
gas_used_after_refund = 70,000 - 14,000 = 56,000
```

Even though the transaction earned 19,200 in refund credits, it can only use 14,000. The remaining 5,200 is lost. This cap prevents transactions from getting "free gas" by manipulating storage.

**Example without refunds (simple ETH transfer):**
```
gas_used before refund = 21,000
refund_counter = 0
actual refund = 0
gas_used_after_refund = 21,000
```

#### Adjustment 2: Calldata floor (EIP-7623, Osaka)

After the refund is applied, the protocol enforces a **minimum gas cost** based on calldata size:

```python
gas_used_after_refund = max(gas_used_after_refund, calldata_floor_gas_cost)
```

The calldata floor prevents transactions from using cheap calldata as a data-availability channel while avoiding meaningful execution. The floor is calculated during `validate_transaction`:

```python
# EIP-7623 token model: 1 zero byte = 1 token, 1 non-zero byte = 4 tokens
tokens_in_calldata = num_zeros + num_non_zeros * 4
calldata_floor_gas_cost = tokens_in_calldata * 10 + 21,000
```

**Example:**
```
calldata = b'\x00\x00\x00\x01\x02'  # 3 zeros, 2 non-zeros
tokens = 3 + 2*4 = 11
floor = 11 * 10 + 21,000 = 21,110

gas_used_after_refund = 21,000
calldata_floor = 21,110
gas_used_after_refund = max(21,000, 21,110) = 21,110  ← bumped up
```

Even though the EVM only used 21,000 gas, the transaction pays 21,110 because its calldata is expensive relative to its execution.

---

### Step 5: Fee Distribution

```
                     Fee Distribution
               ┌─────────────┴─────────────┐
               ▼                           ▼
       Refund Unused Gas           Final Fee (Price × Used)
     (Limit - Used) × Price        ┌───────┴───────┐
               │                   ▼               ▼
         Back to Sender        Base Fee      Priority Fee
                               (Burned)     (To Validator)
```

Three things happen with the money that was locked upfront.

#### Left branch: Refund unused gas to sender

```
gas_refund_amount = (gas_limit - gas_used_after_refund) × effective_gas_price
```

**Example:**
```
gas_limit = 100,000
gas_used_after_refund = 56,000
unused_gas = 44,000
gas_refund_amount = 44,000 × 21.5 gwei = 0.000946 ETH
```

This goes **back to the sender**. It's the portion of the upfront payment that wasn't consumed.

#### Right branch: Final fee splits into two parts

The `gas_used_after_refund` portion is split between the base fee and the priority fee.

```
priority_fee_per_gas = effective_gas_price - base_fee_per_gas
total_priority_fee = gas_used_after_refund × priority_fee_per_gas
total_base_fee = gas_used_after_refund × base_fee_per_gas
```

**Example:**
```
effective_gas_price = 21.5 gwei
base_fee_per_gas = 20 gwei
priority_fee_per_gas = 1.5 gwei

gas_used_after_refund = 56,000

total_base_fee = 56,000 × 20 gwei = 0.00112 ETH   → BURNED
total_priority_fee = 56,000 × 1.5 gwei = 0.000084 ETH → VALIDATOR
```

**Base fee (burned):** Removed from circulation permanently. No one receives it. This is EIP-1559's deflationary mechanism.

**Priority fee (to validator):** Goes to the block proposer as a reward for including this transaction.

---

### Complete Money Flow Example

```
Transaction:
  gas_limit = 100,000
  max_fee_per_gas = 30 gwei
  max_priority_fee_per_gas = 1.5 gwei
  base_fee_per_gas = 20 gwei (from block header)
  effective_gas_price = 21.5 gwei

  calldata: 3 zero bytes, 2 non-zero bytes → floor = 21,110 gas
  SSTORE refunds earned during execution: 19,200 gas

─────────────────────────────────────────────────────────────

1. Upfront deduction:
   100,000 × 21.5 gwei = 0.00215 ETH (locked from sender)

2. EVM execution:
   gas_left after execution = 30,000
   gas_used before refund = 100,000 - 30,000 = 70,000

3. SSTORE refund:
   max allowed = 70,000 // 5 = 14,000
   earned = 19,200
   actual refund = min(14,000, 19,200) = 14,000  ← capped
   gas_used = 70,000 - 14,000 = 56,000

4. Calldata floor:
   floor = 21,110
   gas_used = max(56,000, 21,110) = 56,000  ← no change, already above floor

5. Unused gas refund to sender:
   (100,000 - 56,000) × 21.5 gwei = 44,000 × 21.5 gwei = 0.000946 ETH

6. Final fee split:
   Base fee burned:      56,000 × 20 gwei    = 0.00112 ETH
   Priority to validator: 56,000 × 1.5 gwei   = 0.000084 ETH
   ──────────────────────────────────────────────
   Total final fee:      56,000 × 21.5 gwei   = 0.001204 ETH

7. Net cost to sender:
   Upfront:          0.00215 ETH
   Refund received: -0.000946 ETH
   ─────────────────────────────
   Net gas cost:     0.001204 ETH (exactly the final fee)
```

The sender pays **exactly** what was used, nothing more. The upfront payment is just a safety deposit — it's fully reconciled at the end.