---
title: EELS(0) Codebase Overview
published: 2026-03-26
pinned: false
description: An overview of the Ethereum Execution Layer Specification codebase structure.
tags: [BlockChain,Ethereum,EELS]
category: Inside Ethereum
draft: false
---

## Table of Contents

1. [What Is EELS?](#1-what-is-eels)
2. [What EELS Does NOT Implement](#2-what-eels-does-not-implement)
3. [Architecture Overview](#3-architecture-overview)
4. [File Structure](#4-file-structure)
5. [Blockchain Execution Modules](#5-blockchain-execution-modules)
6. [EVM Modules](#6-evm-modules)
7. [Utility Modules](#7-utility-modules)
8. [Cross-Fork Organization](#8-cross-fork-organization)
9. [Typical Usage Patterns](#9-typical-usage-patterns)
10. [Summary](#summary)
11. [References](#references)

---

## 1. What Is EELS?

**EELS (Ethereum Execution Layer Specification)** is a Python reference implementation of the core components of an Ethereum execution client. It serves as:

| Purpose | Description |
|---------|-------------|
| **Executable Specification** | More programmer-friendly and up-to-date successor to the Yellow Paper |
| **EIP Prototyping Platform** | Used to prototype and test new EIPs before production implementation |
| **Reference Implementation** | Provides ground truth for correct Ethereum protocol behavior |
| **Educational Resource** | Helps developers understand Ethereum internals |
| **Fork Snapshots** | Provides complete protocol snapshots at each fork with rendered diffs |

**Key Design Goals**:
- ✅ **Readability** - Code prioritizes clarity over performance
- ✅ **Correctness** - Serves as authoritative reference for protocol behavior
- ✅ **Completeness** - Implements all consensus-critical rules
- ✅ **Maintainability** - Clear structure enables easy updates for new forks

> **Requirements**: Python 3.11 or later is required to run EELS.

---

## 2. What EELS Does NOT Implement

EELS is a **reference specification**, not a production-ready client. It intentionally omits:

| Not Implemented | Reason |
|-----------------|--------|
| **JSON-RPC API** | Maintained in a separate repository (`ethereum/execution-apis`) |
| **P2P Networking** | Not needed for protocol reference |
| **Database Persistence** | State is in-memory; persistence is an implementation detail |
| **Transaction Pool** | Mempool management is client-specific |
| **Sync Protocols** | Snap sync, full sync are implementation details |
| **Performance Optimizations** | Prioritizes readability over speed |

**Note**: An external RPC provider can be used to fetch and validate blocks against EELS, and the resulting state can be stored in a local database after validation.

---

## 3. Architecture Overview

Within each fork snapshot, the implementation is roughly split into **two main parts**:

```
┌─────────────────────────────────────────────────────────────┐
│                    EELS ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │            BLOCKCHAIN EXECUTION MODULES             │    │
│  │                                                     │    │
│  │   blocks.py        → Block & Header structures      │    │
│  │   transactions.py  → Transaction types & validation │    │
│  │   state.py         → State management               │    │
│  │   trie.py          → Merkle-Patricia Trie           │    │
│  │   bloom.py         → Bloom filters                  │    │
│  │   fork.py          → State transition function      │    │
│  │   fork_types.py    → Core type definitions          │    │
│  │   requests.py      → EL → CL requests               │    │
│  │                                                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                         │                                   │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    EVM MODULES                      │    │
│  │                                                     │    │
│  │   vm/interpreter.py   → Execution loop              │    │
│  │   vm/gas.py           → Gas calculations            │    │
│  │   vm/stack.py         → Stack management            │    │
│  │   vm/memory.py        → Memory management           │    │
│  │   vm/instructions/    → Opcode implementations      │    │
│  │   vm/precompiled_/                                  │    │
│  │       contracts/      → Precompiled contracts       │    │
│  │   vm/runtime.py       → Runtime helpers             │    │
│  │                                                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Blockchain Execution Modules

Define core protocol components and block processing logic:
- Blocks, transactions, and state tries
- Logic for processing and validating blocks
- Logic for adding validated blocks to the chain

### EVM Modules

Implement all core features of the Ethereum Virtual Machine:
- Gas calculations
- Stack and memory management
- Opcode and precompile implementations
- Interpreter that processes EVM messages

---

## 4. File Structure

```
src/ethereum/forks/<fork_name>/
├── __init__.py           # Fork metadata and activation criteria
├── blocks.py             # Block, Header, Receipt, Log, Withdrawal
├── bloom.py              # Bloom filter for logs
├── exceptions.py         # Fork-specific exceptions
├── fork.py               # State transition function (main entry point)
├── fork_types.py         # Core type definitions (Account, Authorization)
├── requests.py           # EL→CL requests (EIP-7685)
├── state.py              # State management and account operations
├── transactions.py       # Transaction types and validation
├── trie.py               # Merkle-Patricia Trie implementation
├── utils/                # Helper functions
│   ├── address.py        # Address computation
│   ├── hexadecimal.py    # Hex parsing utilities
│   └── message.py        # Message preparation
└── vm/                   # EVM implementation
    ├── __init__.py       # EVM data structures
    ├── eoa_delegation.py # EIP-7702 EOA delegation
    ├── exceptions.py     # EVM runtime exceptions
    ├── gas.py            # Gas constants and calculations
    ├── interpreter.py    # EVM execution loop
    ├── memory.py         # Memory operations
    ├── runtime.py        # Runtime helpers
    ├── stack.py          # Stack operations
    ├── instructions/     # Opcode implementations
    │   ├── arithmetic.py
    │   ├── bitwise.py
    │   ├── block.py
    │   ├── comparison.py
    │   ├── control_flow.py
    │   ├── environment.py
    │   ├── keccak.py
    │   ├── log.py
    │   ├── memory.py
    │   ├── stack.py
    │   ├── storage.py
    │   └── system.py
    └── precompiled_contracts/  # Precompiled contracts
        ├── mapping.py
        ├── ecrecover.py
        ├── sha256.py
        ├── ripemd160.py
        ├── identity.py
        ├── modexp.py
        ├── alt_bn128.py
        ├── blake2f.py
        ├── point_evaluation.py
        ├── p256verify.py
        └── bls12_381/
```

**Note**: Depending on the fork being inspected, some files may be added or removed based on the EIPs introduced in that particular version.

---

## 5. Blockchain Execution Modules

### `fork.py` - State Transition Function

**Purpose**: Main entry point for block processing

**Responsibilities**:
- `state_transition()` - Apply a block to the chain
- `apply_body()` - Execute all transactions in a block
- `process_transaction()` - Process a single transaction
- `validate_header()` - Validate block header against parent
- System transaction processing (beacon root, history storage)

**Key Dependencies**: `blocks.py`, `transactions.py`, `state.py`, `vm/interpreter.py`

---

### `blocks.py` - Block Structures

**Purpose**: Define block and related data structures

**Responsibilities**:
- `Header` - Block header with all metadata fields
- `Block` - Complete block structure
- `Receipt` - Transaction execution receipt
- `Log` - Event log structure
- `Withdrawal` - Consensus layer withdrawal

**Key Dependencies**: `transactions.py`, `fork_types.py`

---

### `transactions.py` - Transaction Types

**Purpose**: Define all transaction types and validation logic

**Responsibilities**:
- `LegacyTransaction` - Original transaction format
- `AccessListTransaction` - EIP-2930 access lists
- `FeeMarketTransaction` - EIP-1559 fee market
- `BlobTransaction` - EIP-4844 blob transactions
- `SetCodeTransaction` - EIP-7702 EOA delegation
- `decode_transaction()` - Parse raw transaction bytes
- `validate_transaction()` - Static validation rules
- `recover_sender()` - ECDSA signature recovery

**Key Dependencies**: `fork_types.py`, `exceptions.py`

---

### `state.py` - State Management

**Purpose**: Manage the world state and account operations

**Responsibilities**:
- `State` - World state structure (account trie + storage tries)
- `TransientStorage` - EIP-1153 transient storage
- Account operations (`get_account`, `set_account`, `destroy_account`)
- Storage operations (`get_storage`, `set_storage`)
- Snapshot mechanism (`begin_transaction`, `commit_transaction`, `rollback_transaction`)
- ETH transfers (`move_ether`)
- Code storage (`get_code`, `set_code`)

**Key Dependencies**: `trie.py`, `fork_types.py`

---

### `trie.py` - Merkle-Patricia Trie

**Purpose**: Implement the MPT data structure

**Responsibilities**:
- `Trie` - Generic MPT structure
- `LeafNode`, `ExtensionNode`, `BranchNode` - MPT node types
- `trie_set()`, `trie_get()` - Insert and lookup operations
- `root()` - Compute MPT root hash
- `copy_trie()` - Copy for snapshots
- Nibble encoding/decoding utilities

**Key Dependencies**: `fork_types.py` (for account encoding)

---

### `bloom.py` - Bloom Filters

**Purpose**: Implement bloom filters for log indexing

**Responsibilities**:
- `add_to_bloom()` - Add entry to bloom filter
- `logs_bloom()` - Build bloom from all logs in a block/transaction

**Key Dependencies**: `blocks.py` (Log structure)

---

### `fork_types.py` - Core Type Definitions

**Purpose**: Define Ethereum-specific types

**Responsibilities**:
- `Account` - Account data structure
- `Authorization` - EIP-7702 authorization tuple
- `VersionedHash` - Blob versioned hash type
- `encode_account()` - RLP encoding for accounts
- `Bloom` - Bloom filter type

**Key Dependencies**: None (foundational types)

---

### `requests.py` - EL→CL Requests

**Purpose**: Handle execution layer to consensus layer requests (EIP-7685)

**Responsibilities**:
- `DepositRequest` - Validator deposit (EIP-6110)
- `WithdrawalRequest` - Validator withdrawal (EIP-7002)
- `ConsolidationRequest` - Validator consolidation (EIP-7251)
- `parse_deposit_requests()` - Parse deposit logs from receipts
- `compute_requests_hash()` - Compute SHA-256 chain of requests

**Key Dependencies**: `blocks.py`, `fork_types.py`

---

### `exceptions.py` - Fork-Specific Exceptions

**Purpose**: Define exceptions specific to this fork

**Responsibilities**:
- `TransactionTypeError` - Unknown transaction type
- `BlobGasLimitExceededError` - Blob gas limit exceeded
- `InsufficientMaxFeePerGasError` - Max fee insufficient
- `InvalidBlobVersionedHashError` - Invalid blob version
- Other fork-specific exception types

**Key Dependencies**: Base exceptions from cross-fork utilities

---

## 6. EVM Modules

### `vm/__init__.py` - EVM Data Structures

**Purpose**: Define core EVM context types

**Responsibilities**:
- `BlockEnvironment` - Block-level context (immutable during block)
- `TransactionEnvironment` - Transaction-level context
- `Message` - Unit of EVM execution (CREATE or CALL)
- `Evm` - Live execution state (pc, stack, memory, etc.)
- `BlockOutput` - Accumulates block execution results

**Key Dependencies**: `state.py`, `blocks.py`, `fork_types.py`

---

### `vm/interpreter.py` - EVM Execution Loop

**Purpose**: Main EVM execution logic

**Responsibilities**:
- `process_message_call()` - Entry point for message execution
- `process_create_message()` - Contract creation flow
- `process_message()` - Regular CALL execution
- `execute_code()` - Main fetch-decode-execute loop
- Precompile dispatch

**Key Dependencies**: All vm/ modules, `state.py`, `precompiled_contracts/`

---

### `vm/gas.py` - Gas Calculations

**Purpose**: All gas constants and calculations

**Responsibilities**:
- Gas constants (`GAS_VERY_LOW`, `GAS_COLD_ACCOUNT_ACCESS`, etc.)
- `charge_gas()` - Consume gas, raise OutOfGasError
- `calculate_memory_gas_cost()` - Quadratic memory pricing
- `calculate_excess_blob_gas()` - EIP-4844 blob gas calculation
- `calculate_blob_gas_price()` - Blob fee calculation

**Key Dependencies**: None (foundational gas logic)

---

### `vm/stack.py` - Stack Operations

**Purpose**: EVM stack management

**Responsibilities**:
- `push()` - Push to stack (with overflow check)
- `pop()` - Pop from stack (with underflow check)

**Key Dependencies**: None

---

### `vm/memory.py` - Memory Operations

**Purpose**: EVM memory management

**Responsibilities**:
- `memory_write()` - Write to memory (with expansion)
- `memory_read_bytes()` - Read from memory
- `buffer_read()` - Read from buffer with zero-padding

**Key Dependencies**: `vm/gas.py` (for memory expansion cost)

---

### `vm/runtime.py` - Runtime Helpers

**Purpose**: Utility functions for EVM execution

**Responsibilities**:
- `get_valid_jump_destinations()` - Pre-scan bytecode for JUMPDESTs
- `get_max_call_gas()` - 63/64 rule (EIP-150)

**Key Dependencies**: None

---

### `vm/instructions/` - Opcode Implementations

**Purpose**: Implement all EVM opcodes

| File | Opcodes |
|------|---------|
| `arithmetic.py` | ADD, MUL, SUB, DIV, SDIV, MOD, SMOD, ADDMOD, MULMOD, EXP, SIGNEXTEND |
| `bitwise.py` | AND, OR, XOR, NOT, BYTE, SHL, SHR, SAR |
| `block.py` | BLOCKHASH, COINBASE, TIMESTAMP, NUMBER, PREVRANDAO, GASLIMIT, CHAINID, SELFBALANCE, BASEFEE, BLOBHASH, BLOBBASEFEE |
| `comparison.py` | LT, GT, SLT, SGT, EQ, ISZERO |
| `control_flow.py` | STOP, JUMP, JUMPI, PC, GAS, JUMPDEST, RETURN, REVERT |
| `environment.py` | ADDRESS, BALANCE, ORIGIN, CALLER, CALLVALUE, CALLDATALOAD, CALLDATASIZE, CALLDATACOPY, CODESIZE, CODECOPY, GASPRICE, EXTCODESIZE, EXTCODECOPY, RETURNDATASIZE, RETURNDATACOPY, EXTCODEHASH |
| `keccak.py` | KECCAK256 |
| `log.py` | LOG0-LOG4 |
| `memory.py` | MLOAD, MSTORE, MSTORE8, MSIZE, MCOPY |
| `stack.py` | POP, PUSH0-PUSH32, DUP1-DUP16, SWAP1-SWAP16 |
| `storage.py` | SLOAD, SSTORE, TLOAD, TSTORE |
| `system.py` | CREATE, CREATE2, CALL, CALLCODE, DELEGATECALL, STATICCALL, SELFDESTRUCT |

**Key Dependencies**: `vm/gas.py`, `vm/stack.py`, `vm/memory.py`, `state.py`

---

### `vm/precompiled_contracts/` - Precompiled Contracts

**Purpose**: Implement built-in contracts at fixed addresses

| File | Address | Purpose |
|------|---------|---------|
| `mapping.py` | - | Dispatch table (`PRE_COMPILED_CONTRACTS`) |
| `ecrecover.py` | 0x01 | ECDSA signature recovery |
| `sha256.py` | 0x02 | SHA-256 hash |
| `ripemd160.py` | 0x03 | RIPEMD-160 hash |
| `identity.py` | 0x04 | Copy input to output |
| `modexp.py` | 0x05 | Modular exponentiation |
| `alt_bn128.py` | 0x06-0x08 | BN254 pairing operations |
| `blake2f.py` | 0x09 | BLAKE2b compression |
| `point_evaluation.py` | 0x0a | KZG point evaluation (EIP-4844) |
| `p256verify.py` | 0x100 | secp256r1 verification (Osaka) |
| `bls12_381/` | 0x0b-0x12 | BLS12-381 operations (Osaka) |

**Key Dependencies**: Cryptographic libraries

---

### `vm/eoa_delegation.py` - EOA Delegation (EIP-7702)

**Purpose**: Handle EOA code delegation (Prague+)

**Responsibilities**:
- `is_valid_delegation()` - Check if code is a delegation
- `get_delegated_code_address()` - Extract delegated address
- `set_delegation()` - Process authorizations
- `access_delegation()` - Resolve delegation

**Key Dependencies**: `vm/interpreter.py`, `state.py`

---

### `vm/exceptions.py` - EVM Runtime Exceptions

**Purpose**: Define EVM execution exceptions

**Responsibilities**:
- `OutOfGasError` - Ran out of gas
- `StackUnderflowError` - Stack underflow
- `StackOverflowError` - Stack overflow
- `InvalidJumpDestError` - Invalid jump destination
- `InvalidOpcode` - Unknown opcode
- `Revert` - REVERT opcode (preserves gas)
- `WriteInStaticContext` - State write during STATICCALL

**Key Dependencies**: Base exceptions from cross-fork utilities

---

## 7. Utility Modules

### `utils/address.py` - Address Computation

**Purpose**: Address derivation utilities

**Responsibilities**:
- `compute_contract_address()` - CREATE address derivation
- `compute_create2_address()` - CREATE2 deterministic address

**Key Dependencies**: Cryptographic hash functions

---

### `utils/hexadecimal.py` - Hex Parsing

**Purpose**: Hexadecimal string parsing

**Responsibilities**:
- `hex_to_address()` - Parse hex string to Address
- Other hex parsing utilities

**Key Dependencies**: None

---

### `utils/message.py` - Message Preparation

**Purpose**: Bridge between transactions and EVM messages

**Responsibilities**:
- `prepare_message()` - Convert transaction to EVM Message

**Key Dependencies**: `vm/__init__.py`, `transactions.py`

---

## 8. Cross-Fork Organization

### Fork Directory Structure

Each fork lives as a self-contained directory snapshot. The repository also tracks the fork under active development in a dedicated branch (`forks/<fork_name>`), which is merged to `master` only after mainnet deployment.

```
src/ethereum/forks/
├── frontier/           # Genesis (Jul 2015) — block 1
├── frontier_thawing/   # Sep 2015 — block 200,000
├── homestead/          # Mar 2016 — block 1,150,000
├── dao_fork/           # Jul 2016 — block 1,920,000
├── tangerine_whistle/  # Oct 2016 — block 2,463,000
├── spurious_dragon/    # Nov 2016 — block 2,675,000
├── byzantium/          # Oct 2017 — block 4,370,000
├── constantinople/     # Feb 2019 — block 7,280,000 (same block as Petersburg)
├── petersburg/         # Feb 2019 — block 7,280,000 (removed EIP-1283)
├── istanbul/           # Dec 2019 — block 9,069,000
├── muir_glacier/       # Jan 2020 — block 9,200,000
├── berlin/             # Apr 2021 — block 12,244,000
├── london/             # Aug 2021 — block 12,965,000
├── arrow_glacier/      # Dec 2021 — block 13,773,000
├── gray_glacier/       # Jun 2022 — block 15,050,000
├── paris/              # Sep 2022 (The Merge) — block 15,537,394
├── shanghai/           # Apr 2023 — timestamp 1681338455
├── cancun/             # Mar 2024 — timestamp 1710338135
├── prague/             # May 2025 (Pectra) — block 22,431,084
└── osaka/              # Dec 2025 (Fusaka EL) — block 23,935,694
```

> **`frontier_thawing` and `petersburg`** are real fork directories in EELS (verifiable in the repo), so both must be included in any complete listing.

### Fork Activation

**Pre-Paris (Frontier through Gray Glacier)**: Activated at a specific block number.
```python
FORK_BLOCK = 12965000  # London
```

**Paris**: Uniquely activated by Proof-of-Work Total Difficulty (TTD), not block number.
```python
# Paris activated when PoW TTD reached 58,750,000,000,000,000,000,000
# (not by block number)
```

**Shanghai onward**: Activated at a specific timestamp. The transition to timestamp-based activation started with Shanghai, not after it.
```python
FORK_CRITERIA = ByTimestamp(1764798551)  # Osaka
```

### Cross-Fork Utilities

Shared utilities live outside fork directories:
```
src/ethereum/
├── crypto/           # Cryptographic primitives
├── exceptions.py     # Base exception classes
├── state.py          # Base state types (Account, Address)
├── trace.py          # Execution tracing
└── utils/            # Shared utilities
```

---

## 9. Typical Usage Patterns

### Validating a Block

```python
from ethereum.forks.osaka.fork import state_transition
from ethereum.forks.osaka.blocks import Block
from ethereum.state import State

# Create chain state
chain = BlockChain(
    blocks=[genesis_block],
    state=initial_state,
    chain_id=U64(1),
)

# Validate and apply block
block = fetch_block()
state_transition(chain, block)

# State is now updated
current_state = chain.state
```

---

### Executing a Transaction

```python
from ethereum.forks.osaka.fork import process_transaction
from ethereum.vm import BlockEnvironment

# Create block environment
block_env = BlockEnvironment(
    chain_id=U64(1),
    state=chain.state,
    block_gas_limit=block.header.gas_limit,
    # ... more fields ...
)

# Process transaction
block_output = BlockOutput()
process_transaction(block_env, block_output, tx, index=0)
```

---

### Fetching State

```python
from ethereum.forks.osaka.state import get_account, get_storage

# Get account
account = get_account(state, address)

# Get storage slot
value = get_storage(state, address, slot)

# Get code
code = get_code(state, account.code_hash)
```

---

## Summary

### Module Responsibilities

| Module | Responsibility |
|--------|---------------|
| `fork.py` | State transition function (main entry point) |
| `blocks.py` | Block, Header, Receipt, Log structures |
| `transactions.py` | Transaction types and validation |
| `state.py` | World state and account operations |
| `trie.py` | Merkle-Patricia Trie |
| `bloom.py` | Bloom filters for logs |
| `fork_types.py` | Core type definitions |
| `requests.py` | EL→CL requests |
| `vm/interpreter.py` | EVM execution loop |
| `vm/gas.py` | Gas calculations |
| `vm/instructions/` | Opcode implementations |
| `vm/precompiled_contracts/` | Precompiled contracts |

### Key Design Principles

1. **Readability over performance** - Code prioritizes clarity
2. **Correctness** - Serves as authoritative reference
3. **Modularity** - Clear separation between EVM and blockchain logic
4. **Fork isolation** - Each fork is a self-contained snapshot
5. **Type safety** - Extensive use of Python type hints

---

## References

- [Ethereum Execution Specs Repository](https://github.com/ethereum/execution-specs)
- [Rendered Specification](https://ethereum.github.io/execution-specs/)
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
- [EIPs](https://eips.ethereum.org/)