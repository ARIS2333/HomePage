---
title: CrossCurve Attak Analysis
published: 2026-02-03
pinned: false
description: Comprehensive report on how the attack was implemented and how to fix the vulnerabilities.
tags: [Smart Contract, Cross-Chain]
category: Cross-Chain
draft: false
image: ./cover.png
---
## 1. Overview

On February 2, 2026, CrossCurve (formerly Eywa), a multi-chain cross-chain liquidity bridge, suffered a security exploit resulting in the theft of **$1,441,892.31** in assets across Ethereum, Arbitrum, and other supported chains.

The root cause of the attack was a critical **oversight in contract inheritance**: the CrossCurve team failed to override the externally callable `expressExecute()` function inherited from Axelar’s `AxelarExpressExecutable` contract. This allowed attackers to bypass Axelar Gateway’s mandatory cross-chain message validation, directly invoking the sensitive `_execute()` function with attacker-controlled parameters. By crafting malicious payloads, the attacker was able to trigger unauthorized asset unlocks from the Eywa CLP Portal, leading to the loss of funds.

---

## 2. Exploit Flow: Step-by-Step Attack Path

This attack unfolded through a chain of unvalidated function calls, leveraging inherited code flaws to bypass security controls:

### 1️⃣ **Initial Function Call**

The attacker initiated the exploit by calling the `ReceiverAxelar.expressExecute()` function directly. This function was externally accessible and required no prior validation from the Axelar Gateway.

![image.png](CrossCurve%20Attack/image.png)

[https://app.blocksec.com/phalcon/explorer/tx/eth/0x37d9b911ef710be851a2e08e1cfc61c2544db0f208faeade29ee98cc7506ccc2](https://app.blocksec.com/phalcon/explorer/tx/eth/0x37d9b911ef710be851a2e08e1cfc61c2544db0f208faeade29ee98cc7506ccc2)

### 2️⃣ **Inherited Function Misuse**

The `ReceiverAxelar` contract inherits from Axelar’s `AxelarExpressExecutable`, but the CrossCurve team did not override the sensitive `expressExecute()` function within the inherited contract. This left the default, unsecure implementation intact.

![image.png](CrossCurve%20Attack/image%201.png)

[https://github.com/eywa-protocol/eywa-cdp/blob/5f4a603c20e177b850cd13fdb3d831d5db2d7cd1/contracts/bridge/receive/ReceiverAxelar.sol#L11](https://github.com/eywa-protocol/eywa-cdp/blob/5f4a603c20e177b850cd13fdb3d831d5db2d7cd1/contracts/bridge/receive/ReceiverAxelar.sol#L11)

### 3️⃣ **Bypassing Gateway Validation**

The native `expressExecute()` function only checked if a `commandId` had been used before. It completely skipped Axelar’s mandatory `validateContractCall()` validation, which is designed to confirm cross-chain messages are approved by the Axelar Gateway. Instead, it accepted attacker-controlled values for `sourceChain` and `sourceAddress`, then passed them directly to the `_execute()` function.

![image.png](CrossCurve%20Attack/image%202.png)

[https://github.com/axelarnetwork/axelar-gmp-sdk-solidity/blob/releases/5.10.x/contracts/express/AxelarExpressExecutable.sol#L92](https://github.com/axelarnetwork/axelar-gmp-sdk-solidity/blob/releases/5.10.x/contracts/express/AxelarExpressExecutable.sol#L92)

### 4️⃣ Malicious Payload Execution

The CrossCurve team did override `_execute()`  function, but its validation only checked if the attacker-provided `sourceAddress` matched the `peers[sourceChain]` mapping. The attacker easily satisfied this by crafting a valid `sourceChain` / `sourceAddress` pair. They then appended a `0x00` suffix to their payload to trigger the `receiveData` function, which executed an arbitrary payload calling the `unlock()` function on the Eywa CLP Portal, draining cross-chain assets.

![image.png](CrossCurve%20Attack/image%203.png)

[https://github.com/eywa-protocol/eywa-cdp/blob/5f4a603c20e177b850cd13fdb3d831d5db2d7cd1/contracts/bridge/receive/ReceiverAxelar.sol#L11](https://github.com/eywa-protocol/eywa-cdp/blob/5f4a603c20e177b850cd13fdb3d831d5db2d7cd1/contracts/bridge/receive/ReceiverAxelar.sol#L11)

By crafting the payload to invoke `expressExecute()`, the attacker ultimately triggers the `unlock` function on the Eywa CLP Portal. This function then authorized the transfer of cross-chain assets to the attacker’s address.

![image.png](CrossCurve%20Attack/image%204.png)

[https://app.blocksec.com/phalcon/explorer/tx/eth/0x37d9b911ef710be851a2e08e1cfc61c2544db0f208faeade29ee98cc7506ccc2?debugLine=78&line=78](https://app.blocksec.com/phalcon/explorer/tx/eth/0x37d9b911ef710be851a2e08e1cfc61c2544db0f208faeade29ee98cc7506ccc2?debugLine=78&line=78)

Below is the full JSON payload the attacker used to call `expressExecute()`, including all parameters passed to the function:

```json
{
  "msg.sender": "0x632400f42e96a5deb547a179ca46b02c22cd25cd",
  "func": "expressExecute",
  "args": {
    "commandId": "0x5e77d6809707bb0c062a5c82270d7d939c4ad094dc683ccd4738131925cdeb01",
    "sourceChain": "berachain",
    "sourceAddress": "0x5eEdDcE72530e4fC96d43E3d70Fe09aD0D037175",
    "payload": "0x000000000000000000000000000000080000000000000f3792bae7f35dcde2916c6e6a72ccd3a5330d56500000000000000000000000000000138de105b391f32e7c1e4224ff1a86ab4c6ab742f5c68f39d485d04b149bda59a97c0000000000000000000000000000003e000000000000000000000000000000008000000000000000000000000000000034000000000000000000000000000000000000000000000f3792bae7f35dcde2916c6e6a72ccd3a5330d5650000000000000000000000000000002844dc9fb35105b391f32e7c1e4224ff1a86ab4c6ab742f5c68f39d485d04b149bda59a97c000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000012000000000000000000000000000000026000000000000000000000000000000001000000000000000000000000000000020000000000000000000000000000000024255000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000200000000000000000000000000000000e00000000000008cb8c4263eb26b2349d74ea2cb1b27bc40709e120000000000000000000033b1666d4acf7d79021f761000000000000cda36e1b514fcc52e4ca1238491e6e789a11a8bb00000000000063240f42e96a5deb547a179ca46b02c22cd25cd0000000000000000000000000000000000000000000000000000000000000138de00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000064259db2b4dc9fb350000000000000000000000000000000000000000f3792bae7f35dcde2916c6e6a72ccd3a5330d56500000000000000000000000000000138de00000000000000000000000000000"
  },
  "return": []
}
```

**Payload Breakdown**

1. `msg.sender`: The attacker’s address (`0x632400f42e96a5deb547a179ca46b02c22cd25cd`), which initiated the `expressExecute()` call.
2. `commandId`: A random, unused value (`0x5e77d680...`) — the native `expressExecute()` only checked for duplicate `commandId`s, so this trivial check was easily bypassed.
3. `sourceChain` **+** `sourceAddress`: Attacker-chosen values (`berachain` + `0x5eEdDcE72530e4fC96d43E3d70Fe09aD0D037175`) that matched the `peers` mapping in `_execute()`, satisfying the only validation step in the `_execute()` function.
4. `payload`: The hex-encoded malicious data, with two critical features:
    - Ends in a suffix that triggers the `receiveData` branch.
    - Contains an encoded call to the `unlock()` function on the Eywa CLP Portal.

---

## Prerequisite for the Attack

The exploit required minimal preparation, with no need for additional hacks or compromised parties:

- **Payload Construction**: The attacker only needed to:
    1. Choose a valid `sourceChain` / `sourceAddress` pair that matched the `peers` mapping.
    2. Craft a payload ending in `0x00` to trigger the `receiveData` branch.
    3. Encode a malicious command to call `unlock()` on the Eywa CLP Portal.
- **No Additional Access**: The attacker did not need to compromise the Axelar Gateway, validators, or any other third-party system—they only exploited the unvalidated inherited function.

---

## Intended Purpose & Proper Usage of `expressExecute()`

Axelar introduced an off-chain solver mechanism designed to optimize cross-chain message execution and reduce latency. This mechanism, known as Express Execution, intentionally allows third-party solvers to voluntarily front-run the standard Axelar validation process.

![image.png](CrossCurve%20Attack/image%205.png)

[https://docs.axelar.dev/dev/general-message-passing/express?f_link_type=f_linkinlinenote&flow_extra=eyJpbmxpbmVfZGlzcGxheV9wb3NpdGlvbiI6MCwiZG9jX3Bvc2l0aW9uIjowLCJkb2NfaWQiOiJkNWY0OGY3MWU2NTViOTVkLTdlOTRhZjhlMTdjMjMwMDkifQ%3D%3D](https://docs.axelar.dev/dev/general-message-passing/express?f_link_type=f_linkinlinenote&flow_extra=eyJpbmxpbmVfZGlzcGxheV9wb3NpdGlvbiI6MCwiZG9jX3Bvc2l0aW9uIjowLCJkb2NfaWQiOiJkNWY0OGY3MWU2NTViOTVkLTdlOTRhZjhlMTdjMjMwMDkifQ%3D%3D)

---

### How It's Supposed to Work:

In a typical cross-chain transaction:

1. **User Initiates Transfer**: A user wants to transfer tokens from Chain A to Chain B
2. **Standard Flow (Slow)**: The transaction must be validated by Axelar Network Gateway before execution on Chain B
3. **Express Flow (Fast)**: A third-party solver watches for pending cross-chain intents and can:
    - Call `expressExecute()` immediately upon seeing the intent
    - Use their own funds to send tokens to the user's address on Chain B
    - Register themselves as the express executor for this transaction
4. **Settlement**: When Axelar Network later validates the message from Chain A:
    - Instead of executing the transaction again on Chain B
    - The network reimburses the express executor who fronted the funds

---

### **Built-In Safety Through Economic Incentives**

The design assumes that malicious express executors can only harm themselves:

- If a solver provides incorrect execution, they lose their own funds
- The actual Axelar Gateway validation still occurs afterward as the authoritative check
- Legitimate transactions are eventually executed correctly by the gateway, regardless of express execution

Therefore, `expressExecute()` has minimal restrictions on who can call it — the economic incentive structure is meant to prevent abuse.

---

### **The Two Validation Layers**

Axelar's security model relies on two separate functions with distinct validation requirements:

**1. `expressExecute()` — Minimal Validation (Express Executor Entry Point)**

```solidity
function expressExecute(
    bytes32 commandId,
    string calldata sourceChain,
    string calldata sourceAddress,
    bytes calldata payload
) external payable virtual {
    if (gateway.isCommandExecuted(commandId)) revert AlreadyExecuted();

    address expressExecutor = msg.sender;
    bytes32 payloadHash = keccak256(payload);

    emit ExpressExecuted(commandId, sourceChain, sourceAddress, payloadHash, expressExecutor);
    _setExpressExecutor(commandId, sourceChain, sourceAddress, payloadHash, expressExecutor);

    _execute(sourceChain, sourceAddress, payload);  // ⚠️ Accepts unvalidated parameters
}

```

**Validation performed:**

- ✅ Checks if `commandId` has already been executed (prevents replay)
- ❌ **No gateway validation** — accepts attacker-controlled `sourceChain` and `sourceAddress`
- ❌ **No signature verification** — anyone can call this function

**2. `execute()` — Full Gateway Validation (Authoritative Execution)**

```solidity
function execute(
    bytes32 commandId,
    string calldata sourceChain,
    string calldata sourceAddress,
    bytes calldata payload
) external {
    bytes32 payloadHash = keccak256(payload);

    // ✅ CRITICAL: Validates message was approved by Axelar Gateway
    if (!gateway.validateContractCall(commandId, sourceChain, sourceAddress, payloadHash))
        revert NotApprovedByGateway();

    address expressExecutor = _popExpressExecutor(commandId, sourceChain, sourceAddress, payloadHash);

    if (expressExecutor != address(0)) {
        // Express executor already fulfilled this - reimburse them
        emit ExpressExecutionFulfilled(commandId, sourceChain, sourceAddress, payloadHash, expressExecutor);
    } else {
        // No express execution - execute normally
        _execute(sourceChain, sourceAddress, payload);
    }
}

```

**Validation performed:**

- ✅ **Full Axelar Gateway validation** via `validateContractCall()`
- ✅ Cryptographically verifies the cross-chain message is legitimate
- ✅ Confirms the message originated from the claimed source chain and address

---

### Where CrossCurve Failed

The security assumption breaks down when **`_execute()` performs privileged operations without additional validation**:

CrossCurve's Vulnerable Implementation

```solidity
function _execute(
    string calldata sourceChain,
    string calldata sourceAddress,
    bytes calldata payload_
) internal override {
    // ⚠️ WEAK VALIDATION: Only checks if sourceAddress matches peers mapping
    require(peers[sourceChain] == sourceAddress.toAddress(), "ReceiverAxelar: wrong peer");

    // ... payload processing ...

    if (payload_[payload_.length - 1] == 0x00) {
        bytes memory payload;
        (payload, sender, chainIdFrom, requestId) = abi.decode(data, (bytes, bytes32, uint256, bytes32));

        // ⚠️ CRITICAL: Executes privileged operation without verifying message authenticity
        IReceiver(receiver).receiveData(sender, uint64(chainIdFrom), payload, requestId);
    }
}

```

The issue is that CrossCurve ****allowed ****`_execute()` to **perform privileged operations (unlocking assets) that should ONLY happen after gateway validation**.

### The Correct Implementation

CrossCurve should have recognized that `_execute()` would be accessible via `expressExecute()` and either:

**Option 1: Disable express execution entirely (Recommended)**

```solidity
function expressExecute(...) external payable override {
    revert("Express execution not supported");
}

```

**Option 2: Separate express execution from privileged operations**

```solidity
function _execute(
    string calldata sourceChain,
    string calldata sourceAddress,
    bytes calldata payload_
) internal override {
    require(peers[sourceChain] == sourceAddress.toAddress(), "ReceiverAxelar: wrong peer");

    // ... payload processing ...

    if (payload_[payload_.length - 1] == 0x00) {
        bytes memory payload;
        (payload, sender, chainIdFrom, requestId) = abi.decode(data, (bytes, bytes32, uint256, bytes32));

        // ⚠️ For express execution: ONLY allow non-privileged operations
        // Example: Pre-fund user's account, but DON'T unlock protocol assets
        // The actual asset unlock should ONLY happen in execute() after gateway validation

        // ✅ For validated execution: Safe to perform privileged operations
        // This would require tracking which execution path triggered _execute()
    }
}

```

**Option 3: Only allow whitelisted express executors**

```solidity
mapping(address => bool) public approvedExpressExecutors;

function expressExecute(
    bytes32 commandId,
    string calldata sourceChain,
    string calldata sourceAddress,
    bytes calldata payload
) external payable override {
    require(approvedExpressExecutors[msg.sender], "Not approved express executor");
    super.expressExecute(commandId, sourceChain, sourceAddress, payload);
}

```