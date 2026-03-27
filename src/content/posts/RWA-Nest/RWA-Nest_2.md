---
title: Nest (RWA Earn) - Part 2
published: 2026-02-10
pinned: false
description: Roles & Security Model
tags: [Smart Contract,RWA]
category: RWA
draft: false
image: ./cover.png
---

## **Role 1:  Owner (Per-Contract)**

The **Owner** is the address returned by `owner()` in each specific contract. Because each contract is initialized separately, the "Owner" of the Vault might not be the "Owner" of the Accountant.

### **A. Vault Owner (NestVault / NestVaultOFT)**

- **Pricing Authority:** Can call `setAccountantWithRateProviders` to change where the vault gets its exchange rate.
- **Fiscal Policy:** Can call `setFee` to modify *Deposit*, *Redemption*, and *Instant Redemption* fees (up to the FEE_CAP).
- **Operational Control:** Inherits all `requiresAuth` powers, including *fulfilling redemptions* and performing *administrative withdrawals*.

### **B. Accountant Owner (NestAccountant)**

- **Economic Guardrails:** Sets `updateUpper` and `updateLower` bounds for exchange rate volatility.
- **Revenue Management:** Updates the `managementFee` percentage and the `payoutAddress` where fees are sent.
- **Oracle Management:** Calls `setRateProviderData` to define which external contracts provide prices for specific assets.
- **Emergency Intervention:** Can `pause()` or `unpause()` the pricing engine, effectively freezing all vault movements.
- **Frequency Control:** Sets `updateDelay` to limit how often the exchange rate can be changed.

### **C. Share Owner (NestShareOFT)**

- **System Mapping:** Calls `setVault` to link specific ERC20 assets to their respective Vault logic.
- **God Mode (`manage`):** The most sensitive power. Can execute arbitrary calls from the Share contract. This can be used to rescue tokens, but also to bypass all logic and drain the underlying assets.
- **Asset Flow Control:** Can manually trigger `enter` (mint) or `exit` (burn/withdraw) functions.

### **D. Proxy Owner (NestVaultPredicateProxy)**

- **Compliance Policy:** Calls `setPolicy` to change the off-chain requirement (Policy ID) for users.
- **Infrastructure:** Calls `setPredicateManager` to point to a new Service Manager.
- **Safety Switch:** Can `pause()` or `unpause()` the entry point to the protocol.

---

## **Role 2: Authorized Address**

**Role:** Addresses authorized via `requiresAuth` modifier

**Scope:** Contract-specific administrative functions

### **In NestVault/NestVaultCore:**

- `fulfillRedeem()` - Process pending redemptions
- `requestRedeem()` - Submit redemption requests (also user-accessible)
- `instantRedeem()` - Execute instant redemptions
- `setAccountantWithRateProviders()` - Update pricing oracle
- `setFee()` - Modify fee parameters

### **In NestAccountant:**

- `updateExchangeRate()` - Update share pricing (critical!)
- `pause()` / `unpause()` - Emergency controls
- `updateDelay()` - Configure update frequency
- `updateUpper()` / `updateLower()` - Set rate bounds
- `updateManagementFee()` - Adjust fee rates
- `updatePayoutAddress()` - Change fee recipient
- `setRateProviderData()` - Configure asset rate providers

### **In NestShareOFT:**

- `setVault()` - Associate asset with vault
- `enter()` / `exit()` - Mint/burn shares (vault-only)
- `manage()` - Execute arbitrary calls (high privilege!)

### **In NestVaultPredicateProxy:**

- `deposit()` (overloaded with depositor param) - Cross-chain composer role
- `setPolicy()` - Update predicate policy
- `setPredicateManager()` - Change authorization oracle
- `pause()` / `unpause()` - Emergency controls

**Security Note:** These roles should be distributed across specialized addresses, for example:

- **Rate Updater Bot:** Only has `updateExchangeRate()` permission
- **Redemption Processor:** Only has `fulfillRedeem()` permission
- **Emergency Responder:** Only has `pause()` permission

---

## **Role 3: ERC-7540 Operators**

**Role:** User-delegated operators via `setOperator()` or `authorizeOperator()`

**Scope:** Act on behalf of specific users

**Powers:**

- Submit redemption requests for the controller
- Update pending redemption amounts
- Withdraw claimable assets

**Implementation:**

```solidity
// Direct authorization
vault.setOperator(operatorAddress, true);

// Signature-based (EIP-7441)
vault.authorizeOperator(
    controller,      // User address
    operator,        // Operator address
    approved,        // true/false
    nonce,          // Unique identifier
    deadline,       // Expiration timestamp
    signature       // User's signature
);
```

---

## **Role 4: Predicate-Verified Users**

**Role:** End users with valid predicate proofs

**Scope:** Deposit permissions only

**Powers:**

- Deposit assets through `NestVaultPredicateProxy`
- Mint shares through proxy

**Implementation:**

```solidity
// User must obtain PredicateMessage from off-chain service
PredicateMessage memory proof = fetchPredicateProof(userAddress);

proxy.deposit(
    depositAsset,
    depositAmount,
    receiver,
    vault,
    proof  // Cryptographic proof of authorization
);
```

**Security Note:** Predicate verification happens on-chain against policies stored in the `ServiceManager`. If the policy changes, previously valid proofs become invalid immediately.