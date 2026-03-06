---
title: LayerZero(2) - Security Advisory
published: 2026-02-26
pinned: false
description: Considerations for developers when building ontop of LayerZero
tags: [Web3, Dev Handbook, Cross-Chain]
category: Cross-Chain
draft: false
image: ./cover.png
---

# Part I — Developer Security Considerations

## **Owner Privileges**

### 1. `setPeer(...)`

**What it is:** Called on the OApp, `setPeer(uint32 _eid, bytes32 _peer)` establishes the 1-to-1 trusted relationship between the local contract and a remote contract.

**The Attack:** If the `owner` is compromised, the attacker can overwrite a legitimate peer with a malicious contract they deployed on the remote chain. Any message sent from this malicious contract will pass the peer check, allowing the attacker to inject arbitrary payloads (e.g., minting infinite tokens).

**Mitigation:**
For pathways that will not change, developers could disable `setPeer` after the initial setup, or using a timelock.

### 2. `setDelegate(...)`

**What it is:** The `delegate` is the address authorized by the `EndpointV2` to manage an OApp's operational and security configurations. While the OApp **Owner** manages internal logic (like `setPeer`), the **Delegate** acts on the Endpoint.

The Delegate is authorized to:

- **Configure the Security Stack:** Call `setConfig` to set DVNs, block confirmations, and Executor preferences.
- **Manage Libraries:** Call `setSendLibrary` or `setReceiveLibrary` to upgrade or change the Message Library (e.g., ULN 302).
- **Resolve Stuck Pathways:** For OApps using ordered execution, the Delegate can call `skip` or `nilify` on the Endpoint to manually advance the nonce if a message is blocked or reverting.

**The Attack:** If an attacker takes over the Delegate role—either through an unprotected `setDelegate` function in the OApp or by compromising the Delegate's private key—they gain "root access" to the communication channel.

**Mitigation:**

- **Strict Access Control:** The `setDelegate` function in your OApp must be protected by `onlyOwner`.
- **Separation of Concerns:** While the Owner and Delegate can be different, both must be high-threshold Multisigs or Timelocks.
- **Verification:** Periodically verify the active delegate by querying the Endpoint directly: `ILayerZeroEndpointV2(endpoint).delegates(address(yourOApp))`.

### 3. `setConfig(...)`

**What it is:** Called by the Delegate on the `EndpointV2`, `setConfig(...)` defines the DVN configuration and block confirmations for message validation.

**The Attack:** A compromised Delegate calls `setConfig` to remove your premium DVNs and replaces them with a **1-of-1 malicious DVN** (a wallet the attacker controls). The attacker then submits fake cross-chain messages directly to the Endpoint. Because the Endpoint now trusts the attacker's wallet as the sole DVN, the forged message is validated and delivered to the OApp.

**Mitigation:**
The `delegate` must **never** be an Externally Owned Account (EOA). It must be a Timelock or a high-threshold Multisig. Implement off-chain monitoring to alert instantly if `setConfig` is called.

### 4. `setSendLibrary(...)` & `setReceiveLibrary(...)`

**What it is:** These functions (called on the EndpointV2) allow the Delegate to select which Message Library (e.g., ULN302) is responsible for verifying the authenticity of messages for a given pathway.

This one is fine

## Access Control on `_lzReceive` and `lzCompose`

**For OApp inheritors:** The base `OApp.sol` (via `OAppReceiver`) already handles both endpoint-only access and peer validation in the public `lzReceive()` entry point. The actual source code from LayerZero’s repo is:

```solidity
// From OAppReceiver.sol (actual LayerZero V2 source)
// https://github.com/LayerZero-Labs/LayerZero-v2/blob/main/packages/layerzero-v2/evm/oapp/contracts/oapp/OAppReceiver.sol

function lzReceive(
    Origin calldata _origin,
    bytes32 _guid,
    bytes calldata _message,
    address _executor,
    bytes calldata _extraData
) public payable virtual {
    // Check 1: Only the Endpoint can call this
    if (address(endpoint) != msg.sender) revert OnlyEndpoint(msg.sender);
    // Check 2: sender must match the registered peer for that srcEid
    if (_getPeerOrRevert(_origin.srcEid) != _origin.sender) revert OnlyPeer(_origin.srcEid, _origin.sender);
    // Delegates to your internal hook
    _lzReceive(_origin, _guid, _message, _executor, _extraData);
}
```

Developers who **correctly inherit OApp and only override `_lzReceive`** get these protections for free. The real vulnerabilities are:

**Vulnerability A — Implementing `ILayerZeroReceiver` directly (bypasses all built-in checks):**

```solidity
// ❌ VULNERABLE — implementing the raw interface with no access control
contract MyReceiver is ILayerZeroReceiver {
    function lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) external payable override {
        // No msg.sender check, no peer check — anyone can spoof a message!
        _processMessage(_message);
    }
}

// ✅ CORRECT — when using ILayerZeroReceiver directly, add both checks manually
function lzReceive(...) external payable override {
    require(msg.sender == address(endpoint), "not endpoint");
    require(_origin.sender == trustedPeers[_origin.srcEid], "not peer");
    _processMessage(_message);
}
```

**Vulnerability B — `lzCompose` has NO built-in checks:**

Unlike `lzReceive`, the `ILayerZeroComposer.lzCompose` interface does **not** have built-in endpoint or peer validation. Every OApp that implements `lzCompose` must add these checks manually:

```solidity
//https://github.com/LayerZero-Labs/LayerZero-v2/blob/401e23dd928d3d27e80fda65e053ee311ed6a08e/packages/layerzero-v2/evm/protocol/contracts/interfaces/ILayerZeroComposer.sol

// ❌ VULNERABLE — lzCompose with no access control
function lzCompose(
    address _from,
    bytes32 _guid,
    bytes calldata _message,
    address _executor,
    bytes calldata _extraData
) external payable override {
    // Anyone can call this and inject arbitrary compose messages!
    _execute(_message);
}

// ✅ CORRECT — per LayerZero's official integration checklist
function lzCompose(
    address _from,
    bytes32 _guid,
    bytes calldata _message,
    address _executor,
    bytes calldata _extraData
) external payable override {
    require(msg.sender == address(endpoint), "lzCompose: not endpoint");
    require(_from == trustedOApp, "lzCompose: not trusted OApp");
    _execute(_message);
}
```

---

## Peer Validation — `setPeer` Configuration

**Severity: HIGH**

In LayerZero V2, `setPeer(uint32 _eid, bytes32 _peer)` registers the only trusted counterpart contract on each remote chain. A misconfigured peer address means your OApp will silently reject all incoming messages — `_getPeerOrRevert` will revert when the peer is `bytes32(0)` or mismatched.

```solidity
// From OApp.sol base contract:
// https://github.com/LayerZero-Labs/LayerZero-v2/blob/401e23dd928d3d27e80fda65e053ee311ed6a08e/packages/layerzero-v2/evm/oapp/contracts/oapp/OAppCore.sol

function setPeer(uint32 _eid, bytes32 _peer) public virtual onlyOwner {
    peers[_eid] = _peer;
    emit PeerSet(_eid, _peer);
}

// ✅ CORRECT way to encode an EVM address as bytes32 (right-padded zeroes)
// This is the LayerZero-documented addressToBytes32 pattern:
function addressToBytes32(address _addr) internal pure returns (bytes32) {
    return bytes32(uint256(uint160(_addr)));
}
// Result: 0x0000000000000000000000005b41892ac4d8a1a2b0121e5fec62788379b210cd
setPeer(EndpointId.SEPOLIA_V2_TESTNET, addressToBytes32(sepoliaContractAddress));

// ❌ WRONG — common mistake: using abi.encode which left-pads address and produces wrong bytes32
bytes32 wrongPeer = bytes32(abi.encode(remoteAddress)); // length mismatch / wrong encoding
// Also wrong: using keccak or any other hash
```

**Critical operational risks:**

```solidity
// ❌ DANGER — setting peer to zero disables the channel silently
// _getPeerOrRevert will revert on all incoming messages from that srcEid
setPeer(40161, bytes32(0)); // Do NOT do this on a live channel

// ❌ DANGER — wiring to wrong contract address (e.g., staging address in prod)
// All messages will be rejected; tokens burned on source with no destination credit
```

**Audit checklist:**
- Confirm `peers[eid]` is non-zero for every active pathway before mainnet deployment
- Ensure `setPeer` is protected by `onlyOwner` — a compromised owner can re-point peers to a malicious contract, effectively stealing all incoming cross-chain funds
- Consider using a timelock for `setPeer` calls on high-TVL OApps
- Pathways are **directional** — you must call `setPeer` on **both** chains; missing one side breaks the channel in that direction

---

## Reentrancy — What the Protocol Protects and What It Does Not

The actual `EndpointV2.lzReceive` implementation clears the stored payload **before** calling your OApp:

```solidity
// LayerZero/V2/protocol/contracts/EndpointV2.sol

function lzReceive(
    Origin calldata _origin,
    address _receiver,
    bytes32 _guid,
    bytes calldata _message,
    bytes calldata _extraData
) external payable {
    // Step 1: _clearPayload runs FIRST — deletes inboundPayloadHash[...][nonce]
    //         This means re-entering THIS same lzReceive with the same nonce
    //         will revert because the hash no longer exists.
    _clearPayload(_receiver, _origin.srcEid, _origin.sender, _origin.nonce, abi.encodePacked(_guid, _message));

    // Step 2: Only AFTER clearing does it call your OApp
    ILayerZeroReceiver(_receiver).lzReceive{ value: msg.value }(_origin, _guid, _message, msg.sender, _extraData);

    emit PacketDelivered(_origin, _receiver);
}
```

**What this prevents:** A malicious Executor or a reentering contract **cannot replay the same cross-chain message** (same nonce, same guid) a second time via `EndpointV2.lzReceive`, because `_clearPayload` already deleted the hash before your logic runs.

**What this does NOT prevent:** Reentrancy into **other functions** of your OApp. If your `_lzReceive` makes an external call to a contract that then calls back into a *different* public function of your OApp (e.g., `withdraw()`, `swap()`, `claim()`), the protocol-level protection offers no defence.

```solidity
// ❌ VULNERABLE — _lzReceive credits balance, then makes an external call.
//    A malicious token with a transfer hook can reenter OApp.withdraw()
//    before the balance update is visible to it.
contract VulnerableOApp is OApp {
    mapping(address => uint256) public pendingWithdrawal;

    function _lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) internal override {
        (address recipient, uint256 amount) = abi.decode(_message, (address, uint256));

        pendingWithdrawal[recipient] += amount;

        // External call — if 'token' has an ERC-777 tokensReceived hook or
        // the token is upgradeable and malicious, it can call back into
        // withdraw() while pendingWithdrawal[recipient] is already inflated
        IERC20(token).transfer(recipient, amount);
    }

    // Separate entry point — NOT protected by EndpointV2's payload-clearing
    function withdraw() external {
        uint256 amount = pendingWithdrawal[msg.sender];
        pendingWithdrawal[msg.sender] = 0; // Does NOT help if reentrant from _lzReceive
        payable(msg.sender).transfer(amount);
    }
}

// ✅ CORRECT — Checks-Effects-Interactions inside _lzReceive
//    + ReentrancyGuard on any function that shares state with _lzReceive
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract SafeOApp is OApp, ReentrancyGuard {
    mapping(address => uint256) public pendingWithdrawal;

    function _lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) internal override nonReentrant {
        (address recipient, uint256 amount) = abi.decode(_message, (address, uint256));

        // Effects first — state settled before any external call
        pendingWithdrawal[recipient] += amount;

        // Interaction last
        IERC20(token).safeTransfer(recipient, amount);
    }

    function withdraw() external nonReentrant {
        uint256 amount = pendingWithdrawal[msg.sender];
        require(amount > 0, "nothing to withdraw");
        pendingWithdrawal[msg.sender] = 0;
        payable(msg.sender).transfer(amount);
    }
}
```

---

## Gas Limit Misconfiguration in Message Options

Message options passed to `_lzSend` via `OptionsBuilder` dictate the gas forwarded to the destination’s `lzReceive()` call. Setting this too low causes a **silent reverted delivery** on the destination — tokens are already burned on the source but never minted on the destination.

```solidity
import { OptionsBuilder } from "@layerzerolabs/lz-evm-oapp-v2/contracts/oapp/libs/OptionsBuilder.sol";

// ❌ DANGEROUS — hardcoded low gas limit; complex _lzReceive logic will OOG
bytes memory options = OptionsBuilder.newOptions()
    .addExecutorLzReceiveOption(50_000, 0); // 50k gas — likely insufficient

// ✅ BETTER — estimate gas with a local fork test, then add buffer
bytes memory options = OptionsBuilder.newOptions()
    .addExecutorLzReceiveOption(200_000, 0); // Empirically determined + 30% buffer

// ✅ EVEN BETTER — expose gas limit as a configurable parameter
uint128 public dstGasLimit = 200_000;

function setDstGasLimit(uint128 _limit) external onlyOwner {
    dstGasLimit = _limit;
}

function _buildOptions() internal view returns (bytes memory) {
    return OptionsBuilder.newOptions()
        .addExecutorLzReceiveOption(dstGasLimit, 0);
}
```

**Recovery path for failed delivery:** If the Executor’s `execute()` call fails due to OOG on destination, the message nonce is blocked. The user (or operator) must call `lzReceive` manually with sufficient gas after the `payloadHash` is committed by DVNs. Design your OApp with a manual retry mechanism:

```solidity
// The Endpoint exposes this for manual retry — document this for users
ILayerZeroEndpointV2(endpoint).lzReceive(
    origin,
    address(this),
    guid,
    message,
    ""
);
```

---

## DVN Security Stack Configuration

The DVN configuration is the backbone of your OApp’s security model. LayerZero V2 uses an X-of-Y-of-N model: `X` required DVNs must all sign, and at least `N` of `Y` optional DVNs must also sign. A weak configuration (e.g., 1-of-1 with a single DVN) means a single compromised or malicious verifier can forge message delivery.

```tsx
// layerzero.config.ts — DVN configuration example

import { DVNOptions } from "@layerzerolabs/lz-evm-sdk-v2";

const securityConfig = {
    requiredDVNs: [
        "0xLayerZeroLabsDVN",   // LayerZero Labs DVN
        "0xGoogleCloudDVN",     // Google Cloud DVN
    ],
    optionalDVNs: [
        "0xChainlinkDVN",       // Chainlink DVN
        "0xPolyhedraDVN",       // Polyhedra DVN
    ],
    optionalDVNThreshold: 1,    // At least 1 of the optional DVNs must sign
    // Effective security: 2-of-2 required + 1-of-2 optional (strong)
};

// ❌ WEAK — single required DVN, no optional DVNs
// If this DVN is compromised, any message can be forged
const weakConfig = {
    requiredDVNs: ["0xSomeSingleDVN"],
    optionalDVNs: [],
    optionalDVNThreshold: 0,
};
```

---

## Nonce & Message Ordering

LayerZero V2 delivers messages with an associated nonce per `(srcEid, sender, dstEid)` pathway. By default, execution is **unordered** — a higher-nonce message can be executed before a lower-nonce one if the DVN threshold is met first. For applications with state dependencies between sequential messages (e.g., a multi-step DeFi operation), this is a logic vulnerability.

If your OApp uses ordered delivery and also calls `skip()`, `burn()`, or `clear()` (which advance the protocol nonce without processing), you must **manually sync your local nonce tracking** to avoid a dangerous mismatch.

```solidity
// ❌ DANGEROUS for ordered workflows — message 2 could execute before message 1
// Default OApp has no ordering enforcement

// ✅ OPTION 1: Use addExecutorOrderedExecutionOption (enforces nonce order at the Executor level)
bytes memory options = OptionsBuilder.newOptions()
    .addExecutorLzReceiveOption(200_000, 0)
    .addExecutorOrderedExecutionOption(); // Forces sequential delivery

// ✅ OPTION 2: Track and enforce nonce order in application logic
contract OrderedOApp is OApp {
    mapping(uint32 => uint64) public nextExpectedNonce; // srcEid => nonce

    function _lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) internal override {
        uint64 expected = nextExpectedNonce[_origin.srcEid];
        require(_origin.nonce == expected, "OApp: unexpected nonce");
        nextExpectedNonce[_origin.srcEid]++;

        _processMessage(_message);
    }

    // ⚠️ If you also call skip()/clear()/nilify() you must keep nextExpectedNonce in sync:
    function skipMessage(uint32 srcEid, address srcSender, uint64 nonce) external onlyOwner {
        endpoint.skip(address(this), srcEid, bytes32(uint256(uint160(srcSender))), nonce);
        nextExpectedNonce[srcEid]++; // MUST manually advance to stay in sync
    }
}
```

**Note:** Using `addExecutorOrderedExecutionOption` means a stuck/failed message at nonce `N` will **block all subsequent messages** (nonce `N+1`, `N+2`, …) on that pathway until the blocked message is resolved via `skip()` or manual retry.

---

## Delegate & Ownership Misconfiguration

In LayerZero V2, the `OApp` constructor takes a `_delegate` parameter. The delegate is the address authorized to call `setConfig()` on the Endpoint — i.e., change DVN configuration, MessageLib version, and block confirmation requirements — **without going through your OApp contract**. If the delegate is set to a hot wallet or an uncontrolled address, an attacker who compromises that key can silently downgrade your DVN security stack.

```solidity
// ❌ RISKY — delegate set to a hot deployer wallet
constructor(address _lzEndpoint)
    OApp(_lzEndpoint, msg.sender) // msg.sender = hot wallet
{}

// ✅ SAFER — delegate set to a multisig or timelock
constructor(address _lzEndpoint, address _multisig)
    OApp(_lzEndpoint, _multisig) // Gnosis Safe or similar
{}

// ✅ Transfer delegate after deployment if initial setup required hot wallet
function transferDelegate(address _newDelegate) external onlyOwner {
    endpoint.setDelegate(_newDelegate);
}
```

**Also verify:** After calling `npx hardhat lz:oapp:wire`, confirm the delegate is correctly set by querying:

```solidity
ILayerZeroEndpointV2(endpoint).delegates(address(this)); // Should return your multisig
```

---

## Fee Handling & `msg.value` Forwarding

The `MessagingFee` struct has two components: `nativeFee` (paid in the chain’s native token) and `lzTokenFee` (paid in ZRO). The `EndpointV2` contract validates that the supplied `msg.value` is at least equal to the required `nativeFee` — if `msg.value < nativeFee`, the call **reverts on the source chain** (safe, no funds lost). Any **excess** `msg.value` beyond what is required is refunded to the `_refundAddress` you supply.

The primary risks are:

```solidity
// ❌ BUG 1 — quoting fee but not passing msg.value to the internal _lzSend
// This compiles fine but will revert at the Endpoint's fee check
function send(bytes calldata _message, bytes calldata _options) external payable {
    MessagingFee memory fee = _quote(dstEid, _message, _options, false);
    // msg.value received but contract has no payable fallback — ETH stuck here
    // _lzSend is never called with value forwarded
    _lzSend(dstEid, _message, _options, fee, payable(msg.sender));
    // NOTE: OApp._lzSend internally calls endpoint.send{value: _fee.nativeFee}(...)
    // It uses _fee.nativeFee from the struct, NOT msg.value directly —
    // so the contract must hold enough ETH at call time
}

// ❌ BUG 2 — setting _refundAddress to address(0) burns excess ETH permanently
_lzSend(dstEid, _message, _options, fee, payable(address(0))); // NEVER do this

// ✅ CORRECT — validate msg.value, use msg.value as the nativeFee directly
// to handle any fee increases between quote-time and submission-time
function send(bytes calldata _message, bytes calldata _options) external payable {
    MessagingFee memory fee = _quote(dstEid, _message, _options, false);
    require(msg.value >= fee.nativeFee, "OApp: insufficient msg.value for fee");

    // Pass msg.value as the nativeFee — excess is automatically refunded
    // by the Endpoint to payable(msg.sender)
    _lzSend(
        dstEid,
        _message,
        _options,
        MessagingFee(msg.value, 0), // Use actual msg.value to handle price slippage
        payable(msg.sender)         // Refund address for excess
    );
}
```

---

## `lzCompose` Atomicity Assumption

When using `lzCompose`, developers sometimes assume the compose call is **atomic with** `lzReceive`. It is not. `lzReceive` delivers the message and emits a `ComposeMsg` event. The Executor then calls `lzCompose` in a **separate transaction**. This creates a time window — and a failure scenario — where the initial receive succeeds but the compose step fails.

```solidity
// ❌ BROKEN ASSUMPTION — state from lzReceive may be stale by the time
//    lzCompose executes, especially under high chain load
contract ComposedOApp is OApp, IOAppComposer {
    uint256 public pendingSwapAmount;

    function _lzReceive(...) internal override {
        pendingSwapAmount = abi.decode(_message, (uint256));
        // Emits ComposeMsg — lzCompose will fire "later"
        emit ComposeMsg(...);
    }

    function lzCompose(
        address _from,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) external payable override {
        // pendingSwapAmount might have been modified by another tx in between!
        _executeSwap(pendingSwapAmount); // UNSAFE
    }
}

// ✅ CORRECT — pass all needed data through the compose message payload itself
function lzCompose(
    address _from,
    bytes32 _guid,
    bytes calldata _message,        // Payload forwarded from lzReceive compose data
    address _executor,
    bytes calldata _extraData
) external payable override {
    require(msg.sender == address(endpoint), "lzCompose: caller not endpoint");
    require(_from == peer, "lzCompose: invalid OApp source");

    // Decode from the message payload directly — no reliance on shared state
    (uint256 amount, address recipient) = abi.decode(_message, (uint256, address));
    _executeSwap(amount, recipient);
}
```

---

# Part II — User-Facing Security Considerations

### Irreversibility of Cross-Chain Transactions

Once a cross-chain message is dispatched via `_lzSend`, the source-chain tokens are **immediately burned or locked** by `_debit()`. There is no cancellation mechanism in the LayerZero protocol. If the destination-side `_lzReceive` fails (e.g., wrong recipient address, insufficient gas, paused contract), the tokens do not automatically return.

**What users should check before sending:**
- Double-check the destination address — especially when bridging to a chain with a different address format (e.g., non-EVM chains using `bytes32` addressing)
- Verify the destination contract is unpaused and accepting deposits
- Use LayerZero Scan (`layerzeroscan.com`) to monitor transaction status after sending

```
Transaction lifecycle on LayerZero Scan:
INFLIGHT → DELIVERED (success)
         → FAILED    (destination reverted — manual retry may be needed)
         → BLOCKING  (ordered channel: a prior nonce is stuck)
```

---

### DVN Trust Assumption

Every OApp has a DVN security configuration set by the **application developer**, not the protocol. Users interacting with a LayerZero-integrated bridge implicitly trust the DVNs are working fine.

**How to verify an OApp’s DVN configuration on-chain:**

```solidity
// Query the Endpoint for an OApp's SendLib config
ILayerZeroEndpointV2 endpoint = ILayerZeroEndpointV2(ENDPOINT_ADDRESS);

// Get the MessageLib (e.g., ULN302) address in use
(address sendLib,) = endpoint.getSendLibrary(oAppAddress, dstEid);

// Query the ULN config from the lib
IMessageLibManager.UlnConfig memory config = IUlnBase(sendLib).getUlnConfig(oAppAddress, dstEid);

// Inspect:
config.requiredDVNCount    // Should be >= 2 for meaningful security
config.requiredDVNs        // Array of required DVN addresses
config.optionalDVNThreshold // Should be > 0 for defense-in-depth
```

Users should treat any OApp with `requiredDVNCount == 1` as higher risk, similar to a 1-of-1 multisig.

---

### Executor Failure & Stuck Messages

The Executor is a **permissionless** off-chain entity — it is not guaranteed to run. If the default Executor goes offline, becomes underfunded, or is censored by a destination chain validator, your message will remain in a `INFLIGHT` state with a committed `payloadHash` but no final delivery.

**Recovery path for users:**

Once DVNs have submitted their `verify()` calls and the `PayloadHash` is committed on the destination chain, **anyone** can manually trigger delivery by calling `lzReceive` on the destination Endpoint directly. Users can do this via Etherscan’s Write Contract interface:

```solidity
// Call directly on the destination chain's LayerZero EndpointV2
ILayerZeroEndpointV2(DEST_ENDPOINT).lzReceive(
    Origin({
        srcEid: SOURCE_EID,           // e.g., 40231 for Arbitrum Sepolia
        sender: bytes32(SENDER_OApp), // Source OApp address as bytes32
        nonce: STUCK_NONCE            // The nonce of the stuck message
    }),
    address(RECEIVER_OApp),
    GUID,
    MESSAGE_PAYLOAD,
    ""
);
```

Monitor the status on LayerZero Scan and attempt manual execution only after the status shows `payloadHash` is committed (DVN verification complete).

---

### Fee Underpayment — Silent Message Failure

LayerZero fees are dynamic and priced at the moment of the `quote()` call. If a user submits a transaction with a `msg.value` below the actual `nativeFee`, the transaction reverts on the source chain — which is safe. However, if an application does not implement a fresh `quote()` call at submission time (caching a stale fee estimate), users may send the wrong amount.

**Fee components users pay in a single source-chain transaction:**
1. Source chain gas fee (standard EVM gas)
2. DVN verification fees (paid to each DVN for attestation work)
3. Executor delivery fee (paid for automated destination execution)
4. Destination gas pre-purchase (converted via the formula below)

```
Source Chain Cost = gasUnits × dstGasPrice × (dstTokenPrice / srcTokenPrice)
```

**User checklist:**
- Always use the application’s front-end quote at transaction submission time, not a cached value
- If bridging during high network congestion on the destination chain, fees will be higher
- Check that the quoted `nativeFee` in the transaction matches what you’re signing in your wallet

---

### Fake OApp / Phishing Contracts

Because LayerZero is a permissionless protocol, anyone can deploy a contract that mimics a legitimate OApp’s interface. A malicious contract can call `_lzSend` to the real Endpoint with valid parameters, but implement a fraudulent `_debit()` that steals additional tokens, or a `_lzReceive` that never properly credits users.

**How to verify an OApp’s legitimacy:**

```solidity
// 1. Check the contract is verified on Etherscan/Arbiscan
// 2. Confirm the registered peer addresses match the official deployment
bytes32 peer = IOApp(suspectContract).peers(DESTINATION_EID);
// Compare against the official deployment address from the project's documentation

// 3. Confirm the Endpoint address is the official LayerZero V2 Endpoint
address ep = IOApp(suspectContract).endpoint();
// Mainnet Ethereum: 0x1a44076050125825900e736c501f859c50fE728c
// Mainnet Arbitrum: 0x1a44076050125825900e736c501f859c50fE728c
// (same address across EVM chains — this is a good sanity check)
```

Any bridge UI that asks you to approve token allowance to an unverified contract or a contract whose `endpoint()` does not return the official LayerZero Endpoint address should be treated as a scam.