---
title: Breakdown of LayerZero Cross-Chain Protocol
published: 2026-02-24
pinned: false
description: Explain the mechanism of LayerZero using a real cross-chain transaction.
tags: [Web3, Dev Handbook, Cross-Chain]
category: Cross-Chain
draft: false
image: ./cover.png
---

# Introduction of LayerZero

LayerZero is an omnichain interoperability protocol designed for arbitrary message passing across different blockchains. It acts as a foundational communication layer that enables developers to create Omnichain Applications (OApps) that can maintain state and transfer assets across different chains seamlessly, without the need for a central intermediary or a middle-chain consensus.

**Key Security Features:**

- **Modular Verification via DVNs**: In LayerZero V2, security is handled by Decentralized Verifier Networks (DVNs). Developers can choose which DVNs verify their messages (e.g., Google Cloud or Chainlink), allowing for a highly customizable and modular security stack tailored to specific application needs.
- **Decoupled Infrastructure**: The protocol separates message verification (DVNs) from message execution (Executors). By ensuring that the entity verifying the message and the entity delivering the message are independent, LayerZero prevents collusion and enhances the overall integrity of the cross-chain communication.
- **Immutable Endpoints**: LayerZero utilizes immutable smart contracts called Endpoints on every supported chain. Because these contracts cannot be upgraded or altered, the core transport logic remains censorship-resistant and predictable for developers and users.

---

# Protocol Overview

![image.png](Analysis%20of%20LayerZero%20Cross-Chain%20Bridge%20(Arb-Sep%20/image.png)

## Key Components

The LayerZero architecture is divided into on-chain components (the Protocol) and off-chain components (the Infrastructure). This separation ensures that security and execution are decoupled.

### **1. On-Chain Components**

- **OApp (Omnichain Application):**
    - **Concept:** Smart contracts developed by users.
    - **Function:** They act as the entry and exit points for the cross-chain message. The **Sender OApp** initiates the message on the Source chain, and the **Receiver OApp** handles the logic once the message arrives on the Destination chain.
- **LZ Endpoint:**
    - **Concept:** A series of immutable smart contracts on every supported blockchain.
    - **Function:** It acts as the interface for OApps. It enforces how many verifiers are needed, it also ensures messages are ordered and delivered without data loss.
- **MessageLib:**
    - **Concept:** Specific logic versions (e.g., **ULN302** shown in the image) that define how messages should be framed and validated.
    - **Function:** It handles the low-level work of packing the message on the source and unpacking/verifying it on the destination.
- **MessageLib Registry:**
    - **Concept:** A directory of all available Message Libraries.
    - **Function:** It allows the Protocol to remain future-proof. If a better verification method is developed, a new MessageLib can be added to the registry without changing the core Endpoint.

### **2. Off-Chain Components**

- **DVNs:**
    - **Concept:** Independent entities that monitor the source chain.
    - **Function:** Their sole job is to **verify**. They read the message hash from the source chain and attest to the destination chain that the message is valid. By using multiple DVNs, LayerZero avoids a single point of failure.
- **Executor:**
    - **Concept:** An off-chain entity responsible for the physical delivery of the message.
    - **Function:** The Executor waits for the DVNs to finish their verification. Once the destination chain has confirmed the message is valid, the Executor calls the destination Endpoint to trigger the final transaction.

---

## Key Interactions & Message Flow

The diagram illustrates the lifecycle of a cross-chain message through four primary steps:

**Step 1: Initiation (`_lzSend`)**

The process begins when the **Sender OApp** calls the `_lzSend` function. The message is passed to the **LZ Endpoint**, which uses the **ULN302** library to emit a `PacketSent` event. This event contains the payload and the routing instructions.

**Step 2: Verification (`verify`)**

The **DVNs** hear the `PacketSent` event on the Source chain. They perform their independent validation, checking the state of the source chain. Once satisfied, they call the `verify` function on the Destination chain's **MessageLib**. This records a `PayloadHash` on the destination, confirming that this specific message was truly sent from the source.

**Step 3: Execution (`execute`)**

While the DVNs are verifying, the **Executor** also hears the initial event. It waits for the **MessageLib** on the destination chain to receive enough verifications from the DVNs to meet the application's security threshold. Once the security check is satisfied, the Executor calls the `execute` function.

**Step 4: Delivery (`_lzReceive`)**

The Destination **LZ Endpoint** receives the execution call, checks that the `PacketVerified` status is true, and then passes the message to the **Receiver OApp** via the `_lzReceive` function. The cross-chain interaction is now complete, and the application logic is triggered on the new chain.

---

# An workflow example: Cross-Chain OFT Transfer

To demonstrate the lifecycle of a cross-chain transaction, I performed a hands-on technical test. I deployed an **OFT (Omnichain Fungible Token)** contract on both the **Arbitrum Sepolia** and **Ethereum Sepolia** networks. 

In this example, I minted tokens on Arbitrum and initiated a cross-chain transfer of **1 OFT** to Sepolia to observe the underlying protocol interactions.

![image.png](Analysis%20of%20LayerZero%20Cross-Chain%20Bridge%20(Arb-Sep%20/image%201.png)

The transaction details can be viewed here:

- **LayerZero Scan:** [0x6d49...00baf582dbcfcba6c4670f6bb109](https://testnet.layerzeroscan.com/tx/0x6d494e393b25131f01bd64bf947299da4ee000baf582dbcfcba6c4670f6bb109)
- **Arbitrum Scan (Source):** [0x6d49...00baf582dbcfcba6c4670f6bb109](https://sepolia.arbiscan.io/tx/0x6d494e393b25131f01bd64bf947299da4ee000baf582dbcfcba6c4670f6bb109)
- **Sepolia Scan (Destination):** [0x2f19...86ce95430a056a](https://sepolia.etherscan.io/tx/0x2f1918b66a867cc59f09d5ac2bb45ec4a0b0bf2abface4cae086ce95430a056a)

The complete workflow can be represented by the image below:

![image.png](Analysis%20of%20LayerZero%20Cross-Chain%20Bridge%20(Arb-Sep%20/image%202.png)

**OFT Message Flow**:

1. **Token Debit**: **`_debit()`** burns or locks tokens locally
2. **Message Encoding**: **`OFTMsgCodec.encode()`** creates standardized token message
3. **LayerZero Send**: **`_lzSend()`** dispatches message via LayerZero protocol
4. **Verification & Delivery**: DVNs verify, Executors deliver (standard LayerZero flow)
5. **Message Decoding**: **`OFTMsgCodec.decode()`** extracts token data from message
6. **Token Credit**: **`_credit()`** mints or unlocks tokens on destination

---

## 1. Deploy and Pair Contracts

To enable cross-chain communication, the OFT contracts were first deployed to both target chains and "wired" together to establish a secure peer-to-peer connection.

The implementation follows the standardized framework provided in the [LayerZero V2 OFT Quickstart](https://docs.layerzero.network/v2/developers/evm/oft/quickstart). However, while the documentation provides a general template, I specifically customized the environment to bridge **Ethereum Sepolia** and **Arbitrum Sepolia**.

This required mapping the specific **Endpoint IDs** in the `hardhat.config.ts` and `layerzero.config.ts` files to ensure the LayerZero protocol correctly identified the communication paths:

- **Ethereum Sepolia:** `EndpointId.SEPOLIA_V2_TESTNET` (40161)
- **Arbitrum Sepolia:** `EndpointId.ARBSEP_V2_TESTNET` (40231)

After deploying the contracts using `npx hardhat lz:deploy`, I performed the "wiring" process. This step is essential as LayerZero contracts will not accept messages from any source unless explicitly paired. I executed the following command to call `setPeer` on both chains:

```bash
npx hardhat lz:oapp:wire --oapp-config layerzero.config.ts
```

This configuration effectively registered the Arbitrum contract address as a trusted "peer" on the Sepolia contract and vice versa, creating a secure, bidirectional channel for the OFT tokens to move.

Below is the code for MyOFT, which mints some test tokens to my wallet address during deployment.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.22;

import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";
import { OFT } from "@layerzerolabs/oft-evm/contracts/OFT.sol";

contract MyOFT is OFT {
    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint,
        address _delegate
    ) OFT(_name, _symbol, _lzEndpoint, _delegate) Ownable(_delegate) {
        _mint(msg.sender, 100000 * (10 ** 18));
    }
}

```

After this, we can see that my wallet address (`0x5B41…cd`) has successfully received 100,000 OFT tokens.

![image.png](Analysis%20of%20LayerZero%20Cross-Chain%20Bridge%20(Arb-Sep%20/image%203.png)

## 2. Origin Chain Invocation (Arbitrum)

I initiated the transfer from the Arbitrum network. This step involves burning the tokens on the source chain and paying the cross-chain gas fee.

I used the Hardhat CLI task to execute the transaction. This task calculates the required gas (via `Quote`) and calls the `send` function on the contract.

```bash
npx hardhat lz:oft:send --network arbitrum-sepolia --to-network sepolia --amount 1 --to 0x3d99...
```

![image.png](Analysis%20of%20LayerZero%20Cross-Chain%20Bridge%20(Arb-Sep%20/image%204.png)

Inside the `MyOFT.sol` contract, the `_lzSend` function is triggered. It performs two key actions:

1. **Debit:** It calls `_debit()`, which burns the token from my wallet.
2. **Packet Emission:** It interacts with the `EndpointV2` contract to emit a `PacketSent` event. This event contains the encoded message intended for Sepolia.

Inside the input data on Arbitrum Scan we can see the following code:

```solidity
Function: send(tuple _sendParam,tuple _fee,address _refundAddress) ***

MethodID: 0xc7c7f5b3
[0]:  0000000000000000000000000000000000000000000000000000000000000080
[1]:  000000000000000000000000000000000000000000000000000065ba9dcb15bd
[2]:  0000000000000000000000000000000000000000000000000000000000000000
[3]:  0000000000000000000000005b41892ac4d8a1a2b0121e5fec62788379b210cd
[4]:  0000000000000000000000000000000000000000000000000000000000009ce1
[5]:  0000000000000000000000003d9962756de0a11f504673e145ec206c70ed015d
[6]:  0000000000000000000000000000000000000000000000000de0b6b3a7640000
[7]:  0000000000000000000000000000000000000000000000000de0b6b3a7640000
[8]:  00000000000000000000000000000000000000000000000000000000000000e0
[9]:  0000000000000000000000000000000000000000000000000000000000000120
[10]: 0000000000000000000000000000000000000000000000000000000000000140
[11]: 0000000000000000000000000000000000000000000000000000000000000002
[12]: 0003000000000000000000000000000000000000000000000000000000000000
[13]: 0000000000000000000000000000000000000000000000000000000000000000
[14]: 0000000000000000000000000000000000000000000000000000000000000000
```

Inside, it shows: 

- The Recipient (line 3): 0x5b41892ac4d8a1a2b0121e5fec62788379b210cd
- The destination Chain (line 4): 9ce1 converts to 40161 in decimal, which is the endpoint for sepolia in layerzero V2

## 3. DVN Verification

Once the packet was emitted on Arbitrum, the off-chain **Decentralized Verifier Networks (DVNs)** detected the transaction.

This stage is automated by the LayerZero infrastructure configured during the wiring step.

1. The DVN (e.g., LayerZero Labs DVN) listens for the `PacketSent` event on Arbitrum.
2. It waits for block confirmations to ensure the transaction is final.
3. It signs a payload hash and submits this verification to the destination chain (Sepolia).

## 4. Destination Chain Invocation (Sepolia)

Upon successful verification, the **Executor** completed the transfer on Ethereum Sepolia.

![image.png](Analysis%20of%20LayerZero%20Cross-Chain%20Bridge%20(Arb-Sep%20/image%205.png)

The LayerZero Executor （`0xF5E8…7a`） called the `lzReceive` function on the destination Endpoint, which in turn called my OFT contract.

```solidity
// Abstract logic inside OFT.sol
function _lzReceive(Origin calldata _origin, bytes calldata _message) internal virtual override {
    // 1. Decodes the message (Amount: 1, To: 0x3d99...)
    (uint256 amount, bytes32 toAddress) = abi.decode(_message, (uint256, bytes32));

    // 2. Mints the token to the user
    _credit(toAddress, amount, ...);
}
```

After this, the MyOFT contract on sepolia minted 1 token for my wallet address.

# DVN

![image.png](Analysis%20of%20LayerZero%20Cross-Chain%20Bridge%20(Arb-Sep%20/image%206.png)

The stack of multiple DVNs allows each application to configure a unique security threshold for each source and destination, known as X-of-Y-of-N, where

- **X** represents the **Required DVNs**: These specific verifiers (e.g., Google Cloud or Chainlink) *must* all sign off on the message.
- **Y** represents the **Optional DVNs**: These are a pool of additional verifiers from which a specific number of approvals is needed.
- **N** represents the **Threshold of Optional DVNs**: This is the minimum number of 'Y' verifiers that must attest to the message.

![image.png](Analysis%20of%20LayerZero%20Cross-Chain%20Bridge%20(Arb-Sep%20/image%207.png)

In this stack, each DVN independently verifies the payloadHash of each message to ensure integrity. Once the designated DVN threshold has been reached, the message nonce can be marked as verified and inserted into the destination Endpoint for execution.

| Message Nonce | Description |
| --- | --- |
| 1 | The Security Stack has verified the `payloadHash` and the nonce has been committed to the Endpoint's messaging channel. |
| 2 | All configured DVNs have verified the `payloadHash`, but no caller has yet committed the nonce to the Endpoint's messaging channel. |
| 3 | Two required and one optional DVN have verified the `payloadHash`, meeting the security threshold, but the nonce has not yet been committed. |
| 4 | Even though the optional DVN threshold is met, the Security Stack requires that every **required** DVN (e.g. `DVN^A`) must verify the `payloadHash` before the nonce can be committed. |
| 5 | Only the required DVNs (e.g. `DVN^A`, `DVN^B`) have verified the `payloadHash`; none of the optional verifiers have submitted their proof. |
| 6 | Both the required DVNs and the optional threshold have verified the `payloadHash`, but no caller has committed the nonce to the Endpoint's messaging channel yet. |

# Executors

In the LayerZero protocol, execution is the final step where a message is officially delivered. Once a message is verified by the DVN stack, the Executor triggers the permissionless functions on the destination Endpoint:

- **lzReceive**: Delivers the verified message to the destination OApp, triggering its primary logic.
- **lzCompose**: Handles more complex, composed messages that occur after the initial receipt.

**Key Roles of the Executor**

- **Gas Abstraction**: The Executor allows users to pay for the entire cross-chain transaction on the **source chain** using the source native token or ZRO. The Executor then handles the conversion and provides the necessary gas to execute the transaction on the destination.
- **Automated Delivery**: The Executor monitors the verification status of messages. As soon as the DVN threshold is met, the Executor automatically invokes the destination contract, ensuring a seamless user experience without requiring the user to switch networks manually.

Developers can pass specific instructions to the Executor via **Message Options**. These include:

- [**`lzReceiveOption`**](https://docs.layerzero.network/v2/developers/evm/configuration/options#lzreceive-option): Specify **`gas`** and **`msg.value`** when calling **`lzReceive(...)`**.
- [**`lzComposeOption`**](https://docs.layerzero.network/v2/developers/evm/configuration/options#lzcompose-option): Specify **`gas`** and **`msg.value`** when calling **`lzCompose(...)`**.
- [**`lzNativeDropOption`**](https://docs.layerzero.network/v2/developers/evm/configuration/options#lznativedrop-option): Drop a specified **`amount`** of native tokens to a **`receiver`** on the destination.
- [**`lzOrderedExecutionOption`**](https://docs.layerzero.network/v2/developers/evm/configuration/options#orderedexecution-option): Enforce nonce-ordered execution of messages.

It is important to note that the Executor is **permissionless**. While applications could use a "Default Executor" for convenience, the protocol is designed so that:

1. **Anyone can be an Executor**: You can run your own Executor or use a third-party service.
2. **Manual Execution**: If an Executor fails or is censored, a user can manually call lzReceive themselves, provided the DVNs have verified the message.

# Transaction Pricing

One of the biggest hurdles in cross-chain development is that different blockchains have no knowledge of each other’s gas prices or token values. LayerZero solves this by providing a transparent, component-based pricing model that allows users to pay for everything on the source chain in a single transaction.

### The Four-Component Fee Structure

Every transaction you send through LayerZero consists of four distinct cost elements:

1. **Source Chain Transaction Fee**: The standard gas fee paid to the source network (e.g., Arbitrum) to process the initial transaction.
2. **Security Stack Fees**: Payment to your chosen **DVNs**. Since each DVN (Google Cloud, Chainlink, etc.) operates its own infrastructure to verify your message, they are compensated for this service.
3. **Executor Fees**: Payment to the **Executor** for the service of monitoring the message and physically delivering it to the destination.
4. **Destination Gas Purchase**: The "pre-payment" for the gas that will be consumed on the destination chain.

### The Conversion Formula

To allow a user to pay in ARB (on Arbitrum) for a transaction that ends in ETH (on Sepolia), the protocol uses a real-time conversion formula:

![image.png](Analysis%20of%20LayerZero%20Cross-Chain%20Bridge%20(Arb-Sep%20/image%208.png)

The protocol calculates the total gas needed on the destination, multiplies it by the current gas price on that chain, and then uses a price ratio (e.g., ETH/ARB) to determine the equivalent amount in the source chain's native token.

### Dynamic Pricing Factors

Fees are not static; they change based on three main variables:

- **Security Configuration**: Using a "1-of-1" DVN setup is cheaper than a "3-of-5" setup. High-security applications pay more for the extra verification.
- **Network Congestion**: If Ethereum is congested, the "Destination Gas Purchase" part of the fee will increase to reflect higher gas prices.
- **Message Complexity**: A simple token transfer (like my 1 OFT example) requires less gas than a complex DeFi interaction, such as a cross-chain swap or a multi-step contract call.

### Fee Estimation

Before a user sends a transaction, OApps should call the on-chain **`quote`** function. This function:

- Returns the exact amount of native tokens required to cover the DVNs, Executor, and destination gas.
- Ensures the transaction won't fail due to underfunding once it reaches the destination.