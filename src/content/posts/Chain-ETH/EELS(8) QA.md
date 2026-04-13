---
title: EELS(8) Q&A Notes
published: 2026-04-06
pinned: false
description: Some QA that might be helpful.
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---


## Table of Contents

0. [Introduction to the Consensus Layer (CL) and Beacon Chain](#0-introduction-to-the-consensus-layer-cl-and-beacon-chain)
1. [What is the relationship between the four protocol execution functions?](#1-what-is-the-relationship-between-the-four-protocol-execution-functions)
2. [Why "checked" vs "unchecked" system transactions?](#2-why-checked-vs-unchecked-system-transactions)
3. [Why do we have both EIP-4788 and EIP-2935? Isn't one enough?](#3-why-do-we-have-both-eip-4788-and-eip-2935-isnt-one-enough)
4. [How do validators/users actually initiate deposits, withdrawals, and consolidations?](#4-how-do-validatorsusers-actually-initiate-deposits-withdrawals-and-consolidations)
5. [If users initiate these via normal transactions, why are they called "system transactions"?](#5-if-users-initiate-these-via-normal-transactions-why-are-they-called-system-transactions)

---

## 0. Introduction to the Consensus Layer (CL) and Beacon Chain

Before understanding *System Transactions*, you must understand *why* they exist. They exist because modern Ethereum is split into two layers: the **Execution Layer (EL)** and the **Consensus Layer (CL)**.

### The Split: The Merge (2022)

Originally, Ethereum was a single monolithic chain. Miners did everything: they ordered transactions, executed them, and secured the network via Proof of Work.

To switch to Proof of Stake (PoS), Ethereum split into two layers:

| Feature | **Execution Layer (EL)** | **Consensus Layer (CL)** |
| :--- | :--- | :--- |
| **Analogy** | The **"Body"** | The **"Brain"** |
| **Role** | Does the actual work (running code, managing state). | Decides *what* work gets done and in *what* order. |
| **Key Tasks** | Transaction execution, smart contracts, gas accounting. | Validator management, block ordering, finality, fork choice. |
| **Client Software** | Geth, Nethermind, Erigon, Besu. | Lighthouse, Prysm, Teku, Nimbus. |

### How They Communicate: The Engine API

The two layers are separate pieces of software that talk to each other via the **Engine API** (HTTP/JSON-RPC over localhost).

The CL is the "Brain." It never executes transactions or runs the EVM. It simply tells the EL (the "Body") what to do.

**The Flow of Block Production:**

1.  **Selection:** The CL selects a validator to propose the next block (using a weighted lottery based on stake and a random seed called `RANDAO`).
2.  **Request:** The CL calls `engine_forkchoiceUpdatedV3` on the EL: *"Start building a block on this parent."*
3.  **Execution:** The EL executes transactions from the mempool, updates the state, and produces an "Execution Payload" (transactions, receipts, state root).
4.  **Retrieval:** The CL calls `engine_getPayloadV3` to get that payload from the EL.
5.  **Proposal:** The CL proposes the block to the network.

### The "Blind" Validator

A critical realization: **The CL is "blind" to transaction content.** It does not know if a transaction is a Uniswap swap or an ETH transfer. It only sees:
*   "Is the state root valid?"
*   "Did the EL sign off on this payload?"

Because the CL relies on the EL to process state, the EL must actively report certain things back to the CL. This is where **System Transactions** come in.

### Validator Selection: How does the CL pick a proposer?

The CL uses a **weighted lottery** to select who proposes the next block in a "Slot" (every 12 seconds).
1.  **RANDAO:** Validators contribute random numbers to a pool to create a random seed (`randao_mix`) that no one can manipulate.
2.  **Selection:** The algorithm looks at all active validators and picks one based on probability. (A validator with 64 ETH is twice as likely to be picked as one with 32 ETH).
3.  **Schedule:** Because the random seed is derived from previous epochs, the schedule is calculated in advance. Validators know *when* they will propose, but they cannot manipulate the *order*.

---

## 1. What is the relationship between the four protocol execution functions?

The four functions are:
1. `process_unchecked_system_transaction`
2. `process_withdrawals`
3. `process_general_purpose_requests`

(Plus `process_checked_system_transaction` which is used internally by #3.)

**They are all called by exactly one caller: `apply_body()`.** They form the four phases of block execution that happen outside of user transactions.

```python
def apply_body(block_env, transactions, withdrawals) -> vm.BlockOutput:
    block_output = vm.BlockOutput()

    # Phase 1: System transactions (unchecked) — runs BEFORE user txs
    process_unchecked_system_transaction(
        block_env=block_env,
        target_address=BEACON_ROOTS_ADDRESS,
        data=block_env.parent_beacon_block_root,
    )

    process_unchecked_system_transaction(
        block_env=block_env,
        target_address=HISTORY_STORAGE_ADDRESS,
        data=block_env.block_hashes[-1],  # The parent hash
    )

    # Phase 2: User transactions
    for i, tx in enumerate(map(decode_transaction, transactions)):
        process_transaction(block_env, block_output, tx, Uint(i))

    # Phase 3: Consensus-layer withdrawals
    process_withdrawals(block_env, block_output, withdrawals)

    # Phase 4: General-purpose requests (EL→CL)
    process_general_purpose_requests(
        block_env=block_env,
        block_output=block_output,
    )

    return block_output
```

The execution order is fixed and mandatory. Each phase feeds into the next.

---

### Phase 1: `process_unchecked_system_transaction`

**What it does:** Calls system contracts on behalf of the protocol. These contracts are deployed by the protocol itself and update protocol-level state.

**Two calls per block:**

**Call 1: Beacon Roots contract (EIP-4788)**

```python
process_unchecked_system_transaction(
    block_env=block_env,
    target_address=BEACON_ROOTS_ADDRESS,       # 0x000F...Beac02
    data=block_env.parent_beacon_block_root,   # 32 bytes
)
```

The BeaconRoots contract stores the parent beacon block root in a ring buffer. This gives the execution layer access to recent beacon chain data. The contract stores 8,191 roots in a ring buffer — each call writes one entry.

**Call 2: History Storage contract (EIP-2935)**

```python
process_unchecked_system_transaction(
    block_env=block_env,
    target_address=HISTORY_STORAGE_ADDRESS,     # 0x0000...2935
    data=block_env.block_hashes[-1],            # the parent block hash
)
```

The History Storage contract stores recent block hashes. This extends the `BLOCKHASH` opcode's reach beyond the last 256 blocks. After the merge, the execution layer needs a place to store block history that the EVM can query.

---

### Phase 2: User Transactions

```python
for i, tx in enumerate(map(decode_transaction, transactions)):
    process_transaction(block_env, block_output, tx, Uint(i))
```

This is the main loop. Every user transaction is executed sequentially, each one seeing the state as modified by all previous ones (including the system transactions from Phase 1).

---

### Phase 3: `process_withdrawals`

**What it does:** Applies consensus-layer validator withdrawals to the execution layer state.

```python
def process_withdrawals(block_env, block_output, withdrawals):
    def increase_recipient_balance(recipient):
        recipient.balance += wd.amount * U256(10**9)  # Gwei → Wei

    for i, wd in enumerate(withdrawals):
        trie_set(
            block_output.withdrawals_trie,
            rlp.encode(Uint(i)),
            rlp.encode(wd),
        )
        modify_state(block_env.state, wd.address, increase_recipient_balance)
```

Each withdrawal is a direct balance increment — no EVM execution, no gas, no possibility of failure. The consensus layer has already validated these withdrawals; the execution layer just applies them.

**Two side effects per withdrawal:**
1. **State mutation:** The recipient's balance increases by `amount × 10^9` wei (since `amount` is in Gwei).
2. **Trie insertion:** The withdrawal is inserted into `block_output.withdrawals_trie` at its sequential index. This trie's root is later verified against the header's `withdrawals_root`.

---

### Phase 4: `process_general_purpose_requests`

**What it does:** Collects all EL→CL (execution layer to consensus layer) requests for this block and assembles them into `block_output.requests`.

```python
def process_general_purpose_requests(block_env, block_output):
    # 1. Scan receipts for deposit requests
    deposit_requests = parse_deposit_requests(block_output)
    requests_from_execution = block_output.requests
    if len(deposit_requests) > 0:
        requests_from_execution.append(DEPOSIT_REQUEST_TYPE + deposit_requests)

    # 2. Call WithdrawalRequest contract
    system_withdrawal_tx_output = process_checked_system_transaction(
        block_env=block_env,
        target_address=WITHDRAWAL_REQUEST_PREDEPLOY_ADDRESS,   # 0x0000...7002
        data=b"",
    )
    if len(system_withdrawal_tx_output.return_data) > 0:
        requests_from_execution.append(
            WITHDRAWAL_REQUEST_TYPE + system_withdrawal_tx_output.return_data
        )

    # 3. Call ConsolidationRequest contract
    system_consolidation_tx_output = process_checked_system_transaction(
        block_env=block_env,
        target_address=CONSOLIDATION_REQUEST_PREDEPLOY_ADDRESS,  # 0x0000...7251
        data=b"",
    )
    if len(system_consolidation_tx_output.return_data) > 0:
        requests_from_execution.append(
            CONSOLIDATION_REQUEST_TYPE + system_consolidation_tx_output.return_data
        )
```

This phase collects three types of requests:

**Type 0: Deposit Requests (EIP-6110)**
Collected by **scanning the receipts trie** — not by calling a contract. After all user transactions have executed, `parse_deposit_requests` iterates every receipt, looks at every log, and checks if any log came from the deposit contract (`0x0000...05fa`). If so, the log data is extracted as a deposit request.

**Type 1: Withdrawal Requests (EIP-7002)**
Collected by **calling the WithdrawalRequest predeploy** via `process_checked_system_transaction`. The contract accumulated withdrawal requests submitted by users during Phase 2. Calling it with empty calldata triggers the contract to dequeue and return the accumulated requests as raw bytes.

**Type 2: Consolidation Requests (EIP-7251)**
Same pattern as withdrawals — **calling the ConsolidationRequest predeploy** via `process_checked_system_transaction`.

---

### The Complete Relationship Diagram

```
apply_body()
    │
    ├── Phase 1: process_unchecked_system_transaction
    │   ├── BEACON_ROOTS_ADDRESS      → stores parent beacon root
    │   └── HISTORY_STORAGE_ADDRESS   → stores parent block hash
    │   (tolerates missing code / failures)
    │
    ├── Phase 2: process_transaction (user txs)
    │   ├── Executes all user transactions
    │   ├── User txs may call deposit contract → generates deposit logs
    │   ├── User txs may call withdrawal contract → queues withdrawal requests
    │   └── User txs may call consolidation contract → queues consolidation requests
    │
    ├── Phase 3: process_withdrawals
    │   ├── Applies CL validator withdrawals (balance increments)
    │   └── Builds withdrawals_trie
    │
    └── Phase 4: process_general_purpose_requests
        ├── Scans receipts trie for deposit logs → deposit requests
        ├── Calls WITHDRAWAL_REQUEST contract → withdrawal requests
        ├── Calls CONSOLIDATION_REQUEST contract → consolidation requests
        └── Appends all to block_output.requests
            │
            ▼
        Later hashed by compute_requests_hash() → requests_hash → block header
```

**Data flows:**
- **Phase 1** modifies state (BeaconRoots, HistoryStorage contracts). No output collected.
- **Phase 2** produces receipts and logs. These are the **input** for Phase 4's deposit scanning.
- **Phase 3** modifies state (balance increments) and builds the withdrawals trie.
- **Phase 4** collects everything from Phase 2's logs and calls two more system contracts. Output is `block_output.requests`, which is later hashed into the block header's `requests_hash`.

---

## 2. Why "checked" vs "unchecked" system transactions?

The difference is in how they handle failures.

### `process_unchecked_system_transaction`

```python
def process_unchecked_system_transaction(block_env, target_address, data):
    system_contract_code = get_code(
        block_env.state,
        get_account(block_env.state, target_address).code_hash,
    )
    # NO check for empty code — silently proceeds with empty code

    return process_system_transaction(
        block_env, target_address, system_contract_code, data,
    )
    # NO check for execution error — returns result regardless
```

1. **No code check.** If the contract hasn't been deployed yet (e.g., before the fork activates), the call simply executes empty code (immediate `STOP`). The block remains valid.
2. **No error check.** If the contract reverts or runs out of gas, the error is silently ignored. The block remains valid.

**Why?** These contracts (BeaconRoots, HistoryStorage) may not exist on older testnets or before their introducing fork. The protocol must tolerate their absence without rejecting the block.

### `process_checked_system_transaction`

```python
def process_checked_system_transaction(block_env, target_address, data):
    system_contract_code = get_code(...)

    if len(system_contract_code) == 0:
        raise InvalidBlock("System contract address does not contain code")

    system_tx_output = process_system_transaction(...)

    if system_tx_output.error:
        raise InvalidBlock("System contract call failed")

    return system_tx_output
```

1. **Code check.** If the contract doesn't exist, the block is **invalid**.
2. **Error check.** If the contract reverts, the block is **invalid**.

**Why?** These contracts (WithdrawalRequest, ConsolidationRequest) are mandatory after their introducing fork. If they're missing or fail, something is fundamentally wrong with the block — it must be rejected.

| Feature | Unchecked | Checked |
|---------|-----------|---------|
| **Empty code** | Silently proceeds (STOP) | Block rejected (`InvalidBlock`) |
| **Execution error** | Silently ignored | Block rejected (`InvalidBlock`) |
| **Used for** | BeaconRoots, HistoryStorage | WithdrawalRequest, ConsolidationRequest |
| **Why** | May not exist pre-fork | Mandatory after fork activation |

---

## 3. Why do we have both EIP-4788 and EIP-2935? Isn't one enough?

They store **completely different data** from **completely different sources** for **completely different consumers**. They're not redundant — they're two separate bridges going in opposite directions.

### EIP-4788 (BeaconRoots) — CL data flowing INTO the EL

**What it stores:** Beacon chain block roots. The consensus layer passes `parent_beacon_block_root` into the execution layer via the block header. The BeaconRoots contract stores it in a ring buffer.

**Why the EL needs CL data:** Smart contracts on the execution layer need access to beacon chain information. For example, a staking derivatives contract needs to know which validator produced a block, or a restaking protocol needs to verify beacon chain state. Before EIP-4788, the EL had zero visibility into the CL.

**Direction:** CL → EL

**Data source:** The beacon block root is computed by the consensus layer and passed down through the Engine API when proposing a block.

**Ring buffer size:** 8,191 entries (~27 days of beacon blocks at 12-second slots).

---

### EIP-2935 (History Storage) — EL data preserved FOR the EL

**What it stores:** Execution layer block hashes. The parent block hash (`block_env.block_hashes[-1]`) is stored in the HistoryStorage contract.

**Why the EL needs its own history:** The `BLOCKHASH` opcode can only access the last 256 block hashes. After the merge, old block hashes were no longer stored in block headers the same way, so the 256-block limit became a hard wall. EIP-2935 extends this window by persisting block hashes in a contract's storage, allowing contracts to verify historical EL blocks going back much further.

**Direction:** EL → EL (historical self-reference)

**Data source:** The execution layer already knows its own block hashes. It stores them proactively before they fall off the 256-block cliff.

**Ring buffer size:** 8,192 entries (~27 days of EL blocks at 12-second slots).

---

### Why can't one do both?

| Aspect | EIP-4788 (BeaconRoots) | EIP-2935 (HistoryStorage) |
|--------|------------------------|---------------------------|
| **Data stored** | Beacon block root (CL data) | Block hash (EL data) |
| **Data source** | Passed from CL via header | Generated by EL itself |
| **Who consumes it** | EL smart contracts wanting CL info | EL smart contracts wanting EL history |
| **What if you removed it** | EL loses all visibility into CL | EL loses ability to verify blocks older than 256 |
| **Input to the contract** | `parent_beacon_block_root` (32 bytes) | `block_hashes[-1]` (32 bytes, but different data) |

Even though both store 32-byte hashes in ring-buffer contracts, the hashes represent different things. The beacon root is a hash of a *consensus layer* block. The block hash is a hash of an *execution layer* block. They're different chains, different data, different use cases.

Think of it this way: EIP-4788 is a **window into the CL** for EL contracts. EIP-2935 is a **rear-view mirror** for the EL to see its own history beyond 256 blocks.

---

## 4. How do validators/users actually initiate deposits, withdrawals, and consolidations?

The key insight: **These are initiated on the Execution Layer via regular transactions, not through the consensus client.** The CL doesn't "initiate" anything — it **reads** the results.

### Deposits (EIP-6110) — initiated by the user on the EL

**Step-by-step:**

1. A user (who wants to become a validator) goes to the official **Ethereum Launchpad** (launchpad.ethereum.org) or uses a staking service.

2. The Launchpad generates the validator keys (BLS public/private keypair for the validator, withdrawal credentials, signature, deposit index) and constructs a deposit transaction.

3. The user sends a **regular Type 2 transaction** to the deposit contract at `0x00000000219ab540356cBB839Cbe05303d7705Fa` with:
   - `value = 32 ETH` (the stake amount)
   - `data` = ABI-encoded deposit parameters (pubkey, withdrawal credentials, signature, deposit index)

4. This transaction is processed like any other user transaction in Phase 2 of `apply_body`. It emits a `DepositEvent` log.

5. After all user transactions finish, `process_general_purpose_requests` calls `parse_deposit_requests(block_output)`, which scans the receipts trie for logs from the deposit contract.

6. The deposit log data is extracted and appended to `block_output.requests` as `DEPOSIT_REQUEST_TYPE + deposit_data`.

7. The requests are hashed into `requests_hash` in the block header.

8. The **Consensus Layer reads** the requests via the Engine API (`engine_getPayloadV3` response). The CL sees the deposit request, adds the validator to its pending deposits queue, and eventually activates it.

**The user never touches the CL client.** They send a normal transaction from MetaMask (or any wallet) to the deposit contract.

---

### Withdrawals (EIP-7002) — initiated by the validator on the EL

**Step-by-step:**

1. A validator wants to withdraw ETH from their stake. They construct a **regular Type 2 transaction** to the WithdrawalRequest predeploy contract at `0x00000961Ef480Eb55e80D19ad83579A64c007002`.

2. The transaction includes:
   - The validator's BLS public key
   - The withdrawal amount (0 = full exit, non-zero = partial withdrawal)
   - A signature from the withdrawal credentials (the EL address that controls the validator's withdrawal key)

3. The user pays a fee to the contract (set by the contract). The contract verifies the signature and queues the request.

4. The request sits in the contract's internal queue until `process_general_purpose_requests` calls the contract with empty calldata at the end of the block.

5. The contract dequeues and returns the queued requests as raw bytes.

6. These are appended to `block_output.requests` as `WITHDRAWAL_REQUEST_TYPE + withdrawal_data`.

7. The CL reads the requests via the Engine API, processes them in its own queue system, and eventually executes the withdrawal on the CL side (adjusting validator balances, removing validators, etc.).

**Again: the user sends a regular transaction on the EL.** They don't click anything in the CL client. The CL is a passive consumer of the EL's output.

---

### Consolidations (EIP-7251) — same pattern

A validator who wants to merge two validators into one sends a regular Type 2 transaction to the ConsolidationRequest predeploy at `0x0000BBdDc7CE488642fb579F8B00f3a590007251`. The transaction includes the source and target validator pubkeys and a signature. The contract queues the request. The CL reads it at the end of the block.

---

### The Big Picture: EL → CL Request Flow

```
User/Validator
    │
    │  Sends regular Type 2 transaction
    ▼
┌─────────────────────────────────────────────┐
│          EXECUTION LAYER (EL)               │
│                                             │
│  Deposit tx → deposit contract → emits log  │
│  Withdrawal tx → WithdrawalRequest contract │
│  Consolidation tx → ConsolidationRequest    │
│                                             │
│  End of block:                              │
│  parse_deposit_requests() → scans logs      │
│  process_checked_system_transaction()       │
│    → dequeues withdrawal/consolidation      │
│                                             │
│  block_output.requests = [...]              │
│  requests_hash = compute_requests_hash()    │
│  → committed to block header                │
└──────────────┬──────────────────────────────┘
               │
               │ Engine API: engine_getPayloadV*
               │ (returns execution payload + requests)
               ▼
┌─────────────────────────────────────────────┐
│          CONSENSUS LAYER (CL)               │
│                                             │
│  Reads requests from the payload            │
│  Processes each request type:               │
│    - Deposits → add to validator queue      │
│    - Withdrawals → execute withdrawals      │
│    - Consolidations → merge validators      │
│                                             │
│  CL state changes in the next epoch         │
└─────────────────────────────────────────────┘
```

The **EL produces** requests. The **CL consumes** them. The user interacts with the EL. The CL never initiates anything on its own — it's driven entirely by what the EL produces.

This is the whole point of EIP-7685: a unified, typed, ordered request channel from EL to CL. Before this framework, each cross-layer mechanism had its own ad-hoc signalling channel. Now there's one standard pipeline.

---

## 5. If users initiate these via normal transactions, why are they called "system transactions"?

The confusion comes from mixing up the **initiation** (which is a user tx) with the **collection** (which is a system tx).

### The Two-Step Lifecycle of a Withdrawal

#### Step 1: The User Transaction (Initiation)
You want to withdraw ETH. You send a normal transaction from your wallet to the `WithdrawalRequest` contract.
*   **Caller:** You (EOA).
*   **Gas:** You pay gas fees.
*   **Action:** You pay a fee to the contract to "queue" your withdrawal request.
*   **Result:** The contract stores your request in an internal queue. **The CL hasn't seen this yet.**

#### Step 2: The System Transaction (Collection)
At the end of the block, the protocol calls `process_general_purpose_requests`. It executes:

```python
system_withdrawal_tx_output = process_checked_system_transaction(
    block_env=block_env,
    target_address=WITHDRAWAL_REQUEST_PREDEPLOY_ADDRESS,
    data=b"",  # <-- Empty data
)
```

*   **Caller:** `SYSTEM_ADDRESS` (Protocol).
*   **Gas:** Free (`SYSTEM_TRANSACTION_GAS`).
*   **Action:** The protocol asks the contract: "Give me everything in your queue."
*   **Result:** The contract returns the raw bytes of your request. These bytes are added to `block_output.requests` so the CL can finally read them.

### Why is Step 2 a "System Transaction"?

If we used a normal transaction for Step 2, we'd have problems:
1.  **Who pays the gas?** No specific user benefits from *collecting* the requests—it's a protocol-level duty. If we made a user pay, the system would be fragile.
2.  **Reliability:** We *need* this collection to happen every single block. If we relied on a user transaction, and no one sent one, the requests would sit in the queue forever.
3.  **Identity:** The CL expects these requests to come from the protocol, not from a random user address.

### The "Pure" System Transactions (Phase 1)

BeaconRoots and HistoryStorage are even clearer examples.
*   **No user initiates them.**
*   The protocol *automatically* calls them at the start of every block to store the latest beacon root and block hash.
*   There is no "User Tx" counterpart for these. They are purely protocol maintenance.

### Summary Table

| Feature | User Transaction | System Transaction |
| :--- | :--- | :--- |
| **Caller** | User (EOA) | `SYSTEM_ADDRESS` |
| **Gas** | User pays | Free (Protocol) |
| **Purpose** | Initiate a request / Use a contract | Update protocol state / Collect requests |
| **Timing** | Included if selected by builder | Mandatory (Start and End of block) |
| **Failure** | User loses gas, tx reverts | Block is rejected (Checked) or ignored (Unchecked) |

So, we name them "System Transactions" because they are **executed by the system, for the system, at the system's expense**, whereas the user transaction is the **trigger** provided by the user.
