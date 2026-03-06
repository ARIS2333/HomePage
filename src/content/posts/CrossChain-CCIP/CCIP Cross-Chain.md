---
title: CCIP(1) - Quick Walkthrough
published: 2026-03-05
pinned: false
description: Breakdown CCIP using a real cross-chain transaction.
tags: [Web3, Dev Handbook, Cross-Chain]
category: Cross-Chain
draft: false
image: ./cover.png
---

# CCIP Overview

![image.jpg](Analysis%20of%20CCIP%20CrossChain%20Bridge/diagram.jpg)
![image.png](Analysis%20of%20CCIP%20CrossChain%20Bridge/image.png)

# Token Transfer Example

To understand how the **CCIP Cross-Chain protocol** works, we will walk through a real example. In this example, we send **1 USD1** from **Ethereum Mainnet** to **Mantle Mainnet** using the `ccip-sdk` provided by Chainlink.

Below is the code used to perform the transfer:

```jsx
import { EVMChain, networkInfo } from '@chainlink/ccip-sdk'
import { Wallet } from 'ethers'

const source = await EVMChain.fromUrl('YOUR_PRC_PROVIDER')
const wallet = new Wallet('Your_PRIVATE_KEY', source.provider)

// Router Contract Address on ETH Mainnet
const router = '0x80226fc0Ee2b096224EeAc085Bb9a8cba1146f7D'
// Mantle Mainnet ID in CCIP Context
const destChainSelector = "1556008542357238666"
// USD1 Address on ETH
const USD1_TOKEN = '0x8d0d000ee44948fc98c9b98a4fa4921476f08b0d'

const message = {
  // Receiver Address on the Mantle Mainnet
  receiver: '0x5b41892ac4d8a1a2b0121e5fec62788379b210cd',
  tokenAmounts: [
    {
      token: USD1_TOKEN,
      amount: 1000000000000000000n, // 1 USD1 (18 decimals)
    },
  ],
}

const fee = await source.getFee({ router, destChainSelector, message })
console.log('Fee:', fee, 'wei')

const request = await source.sendMessage({
  router,
  destChainSelector: destChainSelector,
  message: { ...message, fee },
  wallet,
})
```

After executing this script, you should see an output like this:

```
Fee: 241,402,197,362,714 wei
approve => 0x21f39f4e755473062f6c0a212ae625ff5c83a6526f58cc528af58247cb5c3c13
ccipSend => 0xc3f8f46890511e34c1ecab92e2d992d28c5117790bed4d32876546eb7bc1a91e
```

You can find all router contract addresses [here](https://ccip.chain.link/status), and the list of chain selectors [here](https://docs.chain.link/ccip/directory/mainnet/chain/mainnet).

Next, let’s break down each step of the transaction.

---

## Automatic Approval

Even though we did not explicitly write any code to approve the router contract to spend our **USD1** tokens, the `ccip-sdk` automatically handles this step in the background. This is indicated by the output:

```
approve => 0x21f39f4e755473062f6c0a212ae625ff5c83a6526f58cc528af58247cb5c3c13
```

From [Etherscan](https://etherscan.io/tx/0x21f39f4e755473062f6c0a212ae625ff5c83a6526f58cc528af58247cb5c3c13), we can see that transaction hash

`0x21f39f4e755473062f6c0a212ae625ff5c83a6526f58cc528af58247cb5c3c13`  approves the router contract `0x80226fc0ee2b096224eeac085bb9a8cba1146f7d`f to spend **1 USD1** on behalf of my address `0x5B41892ac4d8a1A2B0121e5FEc62788379b210cd`.

![image.png](Analysis%20of%20CCIP%20CrossChain%20Bridge/image%201.png)

---

For the second transaction hash:

```
0xc3f8f46890511e34c1ecab92e2d992d28c5117790bed4d32876546eb7bc1a91e
```

From [Etherscan](https://etherscan.io/tx/0xc3f8f46890511e34c1ecab92e2d992d28c5117790bed4d32876546eb7bc1a91e) we can see that this transaction calls the `ccipSend` function on the router contract and triggers two token transfer events.

![image.png](Analysis%20of%20CCIP%20CrossChain%20Bridge/image%202.png)

The first transfer wraps **ETH into WETH** and sends the WETH to a fee collection address to pay for the cross-chain execution cost.

The second transfer moves **USD1** from the user’s address into the **USD1 token pool**, which is used by the CCIP protocol to facilitate the cross-chain transfer.

## Cross-Chain Transaction Details (Src Chain)

Now let’s take a closer look at the actual cross-chain transaction:

`0xc3f8f46890511e34c1ecab92e2d992d28c5117790bed4d32876546eb7bc1a91e`

The full execution trace can be viewed on BlockSec Phalcon here:

[https://app.blocksec.com/phalcon/explorer/tx/eth/0xc3f8f46890511e34c1ecab92e2d992d28c5117790bed4d32876546eb7bc1a91e](https://app.blocksec.com/phalcon/explorer/tx/eth/0xc3f8f46890511e34c1ecab92e2d992d28c5117790bed4d32876546eb7bc1a91e)

![image.png](Analysis%20of%20CCIP%20CrossChain%20Bridge/image%203.png)

The transaction calls the `ccipSend` function on the **Chainlink CCIP Router**, which orchestrates the entire cross-chain transfer process. From the execution trace, we can break the transaction into several key stages.

---

### 1. RMN Security Check

At the beginning of the transaction, the router performs a security validation through the **RMN (Risk Management Network)**.

```
ARMProxy.isCursed() → false
RMNRemote.isCursed() → false
```

This check ensures that the protocol is not in a paused or “cursed” state. If RMN determines that the system is compromised or under attack, cross-chain transfers can be halted globally. Since both calls return `false`, the transaction proceeds normally.

---

### 2. Fee Calculation

Next, the router queries the **OnRamp** contract to compute the required cross-chain fee.

```
OnRamp.getFee(...)
FeeQuoter.getValidatedFee(...)
```

The fee is validated using the `FeeQuoter` contract, which calculates the amount required to pay for:

- Cross-chain message delivery
- Destination chain execution
- Network overhead

In this transaction, the required fee is:

```
241,402,197,362,714 wei
```

---

### 3. Wrapping ETH into WETH

The transaction then wraps the ETH sent with the transaction into WETH.

```
WETH.deposit()
```

This produces the following event:

```
WETH.Deposit
dst: Chainlink CCIP Router
amount: 241,402,197,362,714
```

This step converts the native ETH fee into **WETH**, which is used internally by the protocol for fee accounting.

---

### 4. Paying the Cross-Chain Fee

After wrapping, the router transfers the WETH fee to the **OnRamp contract**.

```
WETH.transfer → OnRamp
```

This payment funds the cross-chain message execution and delivery.

---

### 5. Locating the Token Pool

Next, the router determines which token pool manages the USD1 token.

```
OnRamp.getPoolBySourceToken(...)
TokenAdminRegistry.getPool(token)
```

The registry resolves the pool responsible for USD1:

```
HybridWithExternalMinterTokenPool
```

Token pools are responsible for locking or burning tokens on the source chain before the corresponding tokens are minted or unlocked on the destination chain.

---

### 6. Transferring USD1 to the Pool

Once the pool is identified, the router transfers the user’s tokens into the pool.

```
USD1.transferFrom(
  sender,
  HybridWithExternalMinterTokenPool,
  amount
)
```

This step moves **1 USD1** from the sender’s wallet into the CCIP token pool.

The earlier approval transaction allows the router to perform this `transferFrom`.

---

### 7. Lock/Burn Operation

The token pool then executes its cross-chain accounting logic:

```
HybridWithExternalMinterTokenPool.lockOrBurn(...)
```

Depending on the token configuration, the pool will either:

- **Lock** the tokens on the source chain, or
- **Burn** them before minting on the destination chain.

For USD1, this pool prepares the cross-chain transfer so the corresponding tokens can be minted or released on Mantle.

---

### 8. Forwarding the Message to the OnRamp

After collecting the fee and locking the tokens, the router forwards the cross-chain request to the **OnRamp contract**:

```
OnRamp.forwardFromRouter(...)
```

Inside `forwardFromRouter`, several internal operations take place.

First, the **FeeQuoter** processes the message arguments:

```
FeeQuoter.processMessageArgs(...)
```

This step validates the fee payment and prepares the message parameters for the cross-chain system.

Next, the OnRamp verifies the token pool again using the **TokenAdminRegistry**:

```
TokenAdminRegistry.getPool(token)
```

Then, the pool performs the **lock-or-burn operation**:

```
HybridWithExternalMinterTokenPool.lockOrBurn(...)
```

This step finalizes the token-side accounting on the source chain. Depending on the token’s configuration, the pool either locks the tokens or burns them before minting or unlocking equivalent tokens on the destination chain.

Once the token handling is completed, the OnRamp emits the event:

```
OnRamp.CCIPMessageSent
```

The Chainlink CCIP network monitors this event and uses it as the trigger to begin the **off-chain verification and relay process**. After verification by CCIP nodes and the Risk Management Network (RMN), the message will be delivered to the **OffRamp contract on Mantle**, where the final token minting or release will occur.

## Cross-Chain Transaction Details (Destination Chain)

After some time, we can observe on Mantle (the destination chain) that **1 USD1** has been successfully delivered to the recipient address.

Transaction on Mantle:

[https://mantlescan.xyz/tx/0xc67e1e20515ac1f5f21fede7b523a80af91c18078abfcd71d0d7e1ebf8c50cc0](https://mantlescan.xyz/tx/0xc67e1e20515ac1f5f21fede7b523a80af91c18078abfcd71d0d7e1ebf8c50cc0)

![image.png](Analysis%20of%20CCIP%20CrossChain%20Bridge/image%204.png)

The full execution trace can also be inspected on BlockSec Phalcon:

[https://app.blocksec.com/phalcon/explorer/tx/mantle/0xc67e1e20515ac1f5f21fede7b523a80af91c18078abfcd71d0d7e1ebf8c50cc0](https://app.blocksec.com/phalcon/explorer/tx/mantle/0xc67e1e20515ac1f5f21fede7b523a80af91c18078abfcd71d0d7e1ebf8c50cc0)

![image.png](Analysis%20of%20CCIP%20CrossChain%20Bridge/image%205.png)

From the trace we can see that the transaction is executed through the **CCIP OffRamp contract**, which is responsible for verifying cross-chain messages and releasing or minting tokens on the destination chain.

---

### 1. Executing the Cross-Chain Message

The transaction begins with a call to the OffRamp contract:

```
OffRamp.execute(reportContext)
```

This function is called by the **CCIP Executing DON** after the cross-chain message has been validated off-chain. The `reportContext` contains the verified message data that was originally emitted by the `CCIPMessageSent` event on Ethereum.

---

### 2. RMN Security Check

Similar to the source-chain execution, the first step is a **Risk Management Network (RMN)** validation:

```
ARMProxy.isCursed()
RMNRemote.isCursed()
```

Both calls return `false`, meaning the protocol is not in a paused or compromised state. 

---

### 3. Processing the Incoming Message

Next, the OffRamp processes the cross-chain message:

```
OffRamp.executeSingleMessage(...)
```

This function extracts the message payload, then the OffRamp determines which **token pool** should handle the incoming USD1 tokens.

---

### 4. Locating the Token Pool

The contract queries the **TokenAdminRegistry** to find the correct pool responsible for USD1:

```
TokenAdminRegistry.getPool(token = USD1)
```

The registry returns the following pool:

```
BurnMintWithExternalMinterTokenPool
```

---

### 5. Verifying Token Pool Capabilities

Before executing the transfer, the OffRamp checks whether the token pool supports the required interfaces:

```
supportsInterface(...)
```

---

### 6. Minting Tokens on the Destination Chain

Next, the token pool performs the core operation:

```
BurnMintWithExternalMinterTokenPool.releaseOrMint(...)
```

Inside this function, the protocol verifies that the message originated from a valid **OffRamp** and that the transfer complies with configured rate limits.

We can see the following event in the trace:

```
InboundRateLimitConsumed
```

This mechanism prevents excessive inflows of tokens from other chains and acts as a safety guard against potential exploits.

After passing all validations, the pool calls the token’s **TokenGovernor** contract:

```
TokenGovernor.mint(recipient, amount)
```

This step **mints 1 USD1 on Mantle** and sends it directly to the recipient address.

---

### 7. Verifying the Final Balance

The trace shows a balance check before and after the mint operation:

Before mint:

```
USD1.balanceOf(recipient) = 0
```

After mint:

```
USD1.balanceOf(recipient) = 1 USD1
```

This confirms that the cross-chain transfer was completed successfully.

---

### 8. Finalizing Message Execution

Finally, the OffRamp contract emits two events:

```
ExecutionStateChanged
Transmitted
```

`ExecutionStateChanged` indicates that the cross-chain message has been successfully executed, while `Transmitted` records metadata related to the CCIP oracle reporting process.

At this point, the **cross-chain transfer is fully completed.**

## Notes When Initiating a CCIP Cross-Chain Transfer

When initiating a CCIP cross-chain transfer, users should **correctly estimating the fee and constructing the message parameters**. These are primarily handled through two functions: `getFee` and `ccipSend`.

---

### 1. Estimating the Cross-Chain Fee

Before sending a cross-chain message, users should always call `getFee` to estimate the required execution fee.

```solidity
function getFee(
  uint64 destinationChainSelector,
  Client.EVM2AnyMessage memory message
) external view returns (uint256 fee);
```

**Parameters**

| Name | Type | Description |
| --- | --- | --- |
| destinationChainSelector | uint64 | The identifier of the destination chain |
| message | Client.EVM2AnyMessage | The cross-chain CCIP message including token transfers and/or data |

**Returns**

| Name | Type | Description |
| --- | --- | --- |
| fee | uint256 | Execution fee required for message delivery to the destination chain |

The returned fee is **denominated in the `feeToken` specified in the message**. If the `feeToken` is set to `address(0)`, the fee will be paid using the native token of the source chain (e.g., ETH).

Since fees depend on message size, token transfers, and destination chain conditions, it is recommended to **always compute the fee dynamically before calling `ccipSend`**.

---

### 2. Understanding CCIP Message Structures

CCIP messages use different structures depending on the **direction of the message flow**.

**EVM → Other Chains**

When sending a message from an EVM chain, the message format is `EVM2AnyMessage`.

```solidity
struct EVM2AnyMessage {
  bytes receiver;
  bytes data;
  EVMTokenAmount[] tokenAmounts;
  address feeToken;
  bytes extraArgs;
}
```

| Field | Type | Description |
| --- | --- | --- |
| receiver | bytes | Encoded receiver address on the destination chain |
| data | bytes | Custom payload sent with the message |
| tokenAmounts | EVMTokenAmount[] | Tokens and amounts to transfer |
| feeToken | address | Token used to pay the fee (`address(0)` for native token) |
| extraArgs | bytes | Additional arguments encoded using `_argsToBytes` |

The token transfer information is defined using the `EVMTokenAmount` structure:

```solidity
struct EVMTokenAmount {
  address token;
  uint256 amount;
}
```

| Field | Type | Description |
| --- | --- | --- |
| token | address | Token contract address |
| amount | uint256 | Amount to transfer |

---

**Other Chains → EVM**

For messages arriving on an EVM chain, CCIP uses the `Any2EVMMessage` structure.

```solidity
struct Any2EVMMessage {
  bytes32 messageId;
  uint64 sourceChainSelector;
  bytes sender;
  bytes data;
  EVMTokenAmount[] destTokenAmounts;
}
```

| Field | Type | Description |
| --- | --- | --- |
| messageId | bytes32 | Message ID corresponding to the original `ccipSend` call |
| sourceChainSelector | uint64 | Identifier of the source chain |
| sender | bytes | Sender address (needs `abi.decode` if originating from an EVM chain) |
| data | bytes | Custom payload from the original message |
| destTokenAmounts | EVMTokenAmount[] | Tokens and amounts received on the destination chain |

---

### 3. Sending the Cross-Chain Message

Once the correct message parameters are constructed and the required fee is known, the user can initiate the cross-chain transfer using `ccipSend`.

```solidity
function ccipSend(
  uint64 destinationChainSelector,
  Client.EVM2AnyMessage calldata message
) external payable returns (bytes32);
```

**Parameters**

| Name | Type | Description |
| --- | --- | --- |
| destinationChainSelector | uint64 | The destination chain identifier |
| message | Client.EVM2AnyMessage | The cross-chain CCIP message |

**Returns**

| Name | Type | Description |
| --- | --- | --- |
| messageId | bytes32 | Unique identifier of the cross-chain message |