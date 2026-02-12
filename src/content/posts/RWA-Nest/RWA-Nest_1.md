---
title: Analysis Report of Nest (RWA Earn) - Part 1
published: 2026-02-09
pinned: false
description: Overview
tags: [Web3,Dev Handbook,RWA]
category: RWA
draft: false
image: ./cover.png
---
## 1. Executive Summary

The Nest Protocol is a modular **Real World Asset (RWA) Vault** system. It diverges from standard DeFi vaults by separating **Accounting** (Pricing), **Vault Logic** (Orchestration), **Share Storage** (Token/Asset Holding), and **Access Control** (Predicate Proxy).

This architecture is designed to handle:

1. **Illiquidity:** Asynchronous redemption flows via **ERC-7540** (Request → Wait → Fulfill).
2. **Compliance:** Whitelisted access via **Predicates** (Off-chain signatures).
3. **Cross-Chain:** Native **LayerZero OFT** support for the Share token.
4. **Centralized Pricing:** Oracle-driven exchange rates with safety circuit breakers.

**Target Integrators:** Wallets, DeFi protocols, cross-chain bridges, institutional custody systems, compliance tools, and RWA aggregators.

---

## 2. System Components

| Component | Contract | Note |
| --- | --- | --- |
| **Entry Point** | `NestVaultPredicateProxy` | **Users interact here.** Enforces KYC via signatures. |
| **Logic** | `NestVault` & `NestVaultCore` | Manages deposit/redemption flows and fee logic. |
| **Pricing** | `NestAccountant` | Determines Share Price & accrues management fees. |
| **Token** | `NestShareOFT` | Holds the underlying assets and the minted Share supply. |

---

## 3. Prerequisite

This section provides essential background on the key standards and protocols that the Nest Protocol is built upon. Understanding these prerequisites is crucial for comprehending the system's architecture and operation.

### 3.1 ERC-4626: Tokenized Vault Standard

**Overview:**

ERC-4626 is a standard for tokenized vaults that represent shares of a single underlying ERC-20 token. It provides a unified API for deposit/withdrawal operations and share accounting.

**Core Concepts:**

```solidity
interface IERC4626 {
    // Deposit/Withdrawal
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function mint(uint256 shares, address receiver) external returns (uint256 assets);
    function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares);
    function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets);

    // Accounting
    function totalAssets() external view returns (uint256);
    function convertToShares(uint256 assets) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);

    // Limits
    function maxDeposit(address) external view returns (uint256);
    function maxMint(address) external view returns (uint256);
    function maxWithdraw(address owner) external view returns (uint256);
    function maxRedeem(address owner) external view returns (uint256);
}
```

**Key Principles:**

1. **Share Representation:**
    - Shares represent proportional ownership of vault assets
    - Exchange rate: `1 share = (totalAssets / totalSupply) assets`
2. **Deposit vs Mint:**
    
    ```solidity
    // Deposit: Specify asset amount, receive variable shares
    vault.deposit(1000e6, receiver);  // Deposit 1000 USDC
    // Returns: shares based on current rate
    
    // Mint: Specify share amount, deposit variable assets
    vault.mint(1000e6, receiver);     // Mint 1000 shares
    // Returns: assets required based on current rate
    ```
    
3. **Withdraw vs Redeem:**
    
    ```solidity
    // Withdraw: Specify asset amount, burn variable shares
    vault.withdraw(1000e6, receiver, owner);  // Withdraw 1000 USDC
    // Burns: shares based on current rate
    
    // Redeem: Specify share amount, receive variable assets
    vault.redeem(1000e6, receiver, owner);    // Redeem 1000 shares
    // Returns: assets based on current rate
    ```
    

---

### 3.2 ERC-7540: Asynchronous Tokenized Vault Standard

**Overview:**

ERC-7540 extends ERC-4626 to support **asynchronous** deposit and redemption operations. This is essential for real-world assets (RWAs) where liquidation takes time.

**Key Innovation:**

Traditional vaults (ERC-4626):

```
Request → Immediate Execution
```

Asynchronous vaults (ERC-7540):

```
Request → Pending → Fulfillment → Claim
```

**Core Interface:**

```solidity
interface IERC7540 {
    // Request Phase
    function requestDeposit(uint256 assets, address controller, address owner)
        external returns (uint256 requestId);

    function requestRedeem(uint256 shares, address controller, address owner)
        external returns (uint256 requestId);

    // Status Queries
    function pendingDepositRequest(uint256 requestId, address controller)
        external view returns (uint256 pendingAssets);

    function pendingRedeemRequest(uint256 requestId, address controller)
        external view returns (uint256 pendingShares);

    function claimableDepositRequest(uint256 requestId, address controller)
        external view returns (uint256 claimableShares);

    function claimableRedeemRequest(uint256 requestId, address controller)
        external view returns (uint256 claimableAssets);
}
```

**Request Flow:**

![image.png](Nest(RWA%20Earn)-1-Overview/image.png)

**Controller vs Owner Pattern:**

```solidity
// controller: Who receives the assets/shares
// owner: Who provides the assets/shares

// Scenario 1: Direct user operation
vault.requestRedeem(shares, userAddress, userAddress);
// controller = owner = user

// Scenario 2: Protocol integration
vault.requestRedeem(shares, protocolAddress, userAddress);
// controller = protocol (receives assets)
// owner = user (provides shares)

// Scenario 3: Delegated operation
vault.requestRedeem(shares, beneficiary, owner);
// controller = beneficiary (receives assets)
// owner = original shareholder
// Requires: owner approved msg.sender as operator
```

---

### 3.3 ERC-7575: Partial & Extended Vault Standard

**Overview:**

ERC-7575 is a modular vault standard that separates the **share token** from the **vault logic**. It introduces the concept of `share()` - an external share token contract.

**Key Innovation:**

Traditional ERC-4626:

```
Vault Contract = Shares (inherited ERC20)
```

ERC-7575:

```
Vault Contract → Share Contract (separate)
```

**Core Interface:**

```solidity
interface IERC7575 {
    /// @notice The address of the share token
    function share() external view returns (address shareTokenAddress);

    /// @notice The address of the underlying asset
    function asset() external view returns (address assetTokenAddress);
}
```

---

### 3.4 CCTP: Cross-Chain Transfer Protocol

**Overview:**

Circle's Cross-Chain Transfer Protocol (CCTP) enables **native USDC transfers** between blockchains. Unlike traditional bridges that lock-and-mint wrapped tokens, CCTP burns USDC on the source chain and mints native USDC on the destination chain.

**Key Mechanism:**

![image.png](Nest(RWA%20Earn)-1-Overview/image%201.png)

**Core Operations:**

```solidity
interface ITokenMessenger {
    /**
     * @notice Deposits and burns tokens from sender to be minted on destination domain
     * @param amount Amount of tokens to burn
     * @param destinationDomain Destination domain identifier
     * @param mintRecipient Address receiving minted tokens on destination
     * @param burnToken Address of token to burn
     */
    function depositForBurn(
        uint256 amount,
        uint32 destinationDomain,
        bytes32 mintRecipient,
        address burnToken
    ) external returns (uint64 nonce);
}

interface IMessageTransmitter {
    /**
     * @notice Receives a message with attestation and mints tokens
     * @param message Formatted message bytes
     * @param attestation Attestation signature from Circle
     */
    function receiveMessage(
        bytes calldata message,
        bytes calldata attestation
    ) external returns (bool success);
}
```

---

### 3.5 LayerZero: Omnichain Interoperability Protocol

**Overview:**

LayerZero is a generic messaging protocol that enables **omnichain applications** - smart contracts that can seamlessly interact across multiple blockchains. Nest Protocol uses LayerZero's OFT (Omnichain Fungible Token) standard to enable cross-chain share transfers.

**Core Architecture:**

![image.png](Nest(RWA%20Earn)-1-Overview/image%202.png)

**Key Components:**

1. **Endpoint:**
    - On-chain contract deployed on each chain
    - Manages message sending and receiving
    - Handles nonce tracking and security verification
2. **Security Stack (DVN + Executor):**
    - **DVN (Decentralized Verifier Network):** Validates cross-chain messages
    - **Executor:** Delivers messages and pays destination gas
3. **User Application (OApp):**
    - Your contract that inherits from LayerZero base contracts
    - Defines custom cross-chain logic

**OFT (Omnichain Fungible Token):**

OFT is LayerZero's standard for tokens that can move across chains:

```solidity
interface IOFT {
    struct SendParam {
        uint32 dstEid;           // Destination endpoint ID
        bytes32 to;              // Recipient (bytes32 format)
        uint256 amountLD;        // Amount in local decimals
        uint256 minAmountLD;     // Minimum amount (slippage protection)
        bytes extraOptions;      // Gas/execution options
        bytes composeMsg;        // Additional message data
        bytes oftCmd;            // OFT-specific commands
    }

    struct MessagingFee {
        uint256 nativeFee;       // Gas in native token (ETH, etc.)
        uint256 lzTokenFee;      // Optional ZRO token payment
    }

    function send(
        SendParam calldata sendParam,
        MessagingFee calldata fee,
        address refundAddress
    ) external payable returns (
        MessagingReceipt memory,
        OFTReceipt memory
    );
}
```

**Message Flow:**

```solidity
// Source Chain (Ethereum)
User calls: vault.send(sendParam, fee, refund)
  ↓
vault._debit(user, amount)
  ↓
SHARE.exit() → Burns shares (NO assets moved)
  ↓
Endpoint emits Packet event
  ↓
DVN observes and validates
Executor picks up for delivery
  ↓

// Destination Chain (Arbitrum)
Executor calls: endpoint.lzReceive(message)
  ↓
Endpoint validates message
  ↓
vault._credit(user, amount)
  ↓
SHARE.enter() → Mints shares (NO assets moved)
  ↓
User receives shares on destination chain
```

**Security Model:**

LayerZero uses a **configurable security stack**:

```solidity
// Example: Configure DVNs
contract MyOFT is OFTUpgradeable {
    function setEnforcedOptions(
        uint32 _dstEid,
        bytes calldata _options
    ) external onlyOwner {
        // Set required DVNs, gas limits, etc.
        enforcedOptions[_dstEid] = _options;
    }
}

// Typical configuration:
// - Required DVNs: 2-5 independent verifiers
// - Optional DVNs: Additional verifiers for extra security
// - Executor: Usually LayerZero Labs (can be custom)
```