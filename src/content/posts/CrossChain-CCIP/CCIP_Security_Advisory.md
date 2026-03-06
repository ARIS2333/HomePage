---
title: CCIP(2) - Security Advisory
published: 2026-03-06
pinned: false
description: Considerations for developers when building ontop of CCIP
tags: [Web3, Dev Handbook, Cross-Chain]
category: Cross-Chain
draft: false
image: ./cover.png
---

# CCIP Overview

![image.jpg](Analysis%20of%20CCIP%20CrossChain%20Bridge/diagram.jpg)
![image.png](Analysis%20of%20CCIP%20CrossChain%20Bridge/image.png)

# **Token Pool**

Source: [https://github.com/smartcontractkit/chainlink-ccip/blob/contracts-ccip-v1.6.1/chains/evm/contracts/pools/TokenPool.sol](https://github.com/smartcontractkit/chainlink-ccip/blob/contracts-ccip-v1.6.1/chains/evm/contracts/pools/TokenPool.sol)

## Owner Privileges

### 1. `setRouter`

```solidity
function setRouter(address newRouter) public onlyOwner {
    if (newRouter == address(0)) revert ZeroAddressNotAllowed();
    address oldRouter = address(s_router);
    s_router = IRouter(newRouter);
    emit RouterUpdated(oldRouter, newRouter);
}
```

**Impact:** `s_router` is the single source of truth for ramp validation. Every lock/burn and release/mint ultimately checks against it:

```solidity
function _onlyOnRamp(uint64 remoteChainSelector) internal view {
    if (!isSupportedChain(remoteChainSelector)) revert ChainNotAllowed(remoteChainSelector);
    if (!(msg.sender == s_router.getOnRamp(remoteChainSelector)))
        revert CallerIsNotARampOnRouter(msg.sender);  // ← validated against s_router
}

function _onlyOffRamp(uint64 remoteChainSelector) internal view {
    if (!isSupportedChain(remoteChainSelector)) revert ChainNotAllowed(remoteChainSelector);
    if (!s_router.isOffRamp(remoteChainSelector, msg.sender))
        revert CallerIsNotARampOnRouter(msg.sender);  // ← validated against s_router
}
```

Swapping `s_router` to a malicious contract that unconditionally return `true` from `isOffRamp` collapses the inbound security model.

An attacker can then directly call `releaseOrMint` on a **LockRelease pool**, bypassing the `_onlyOffRamp` check and draining the pool's real ERC20 token to an arbitrary receiver — with no cross-chain message, no DON involvement, and no OffRamp execution required:

```solidity
/// @notice Mint tokens from the pool to the recipient
  /// @dev The _validateReleaseOrMint check is an essential security check
  function releaseOrMint(
    Pool.ReleaseOrMintInV1 calldata releaseOrMintIn
  ) public virtual override returns (Pool.ReleaseOrMintOutV1 memory) {
    // Calculate the local amount
    uint256 localAmount = _calculateLocalAmount(
      releaseOrMintIn.sourceDenominatedAmount, _parseRemoteDecimals(releaseOrMintIn.sourcePoolData)
    );

    _validateReleaseOrMint(releaseOrMintIn, localAmount);

    // Mint to the receiver
    _releaseOrMint(releaseOrMintIn.receiver, localAmount);

    emit ReleasedOrMinted({
      remoteChainSelector: releaseOrMintIn.remoteChainSelector,
      token: address(i_token),
      sender: msg.sender,
      recipient: releaseOrMintIn.receiver,
      amount: localAmount
    });

    return Pool.ReleaseOrMintOutV1({destinationAmount: localAmount});
  }
```

The attack is entirely local to the destination chain, making it invisible to the DON which only monitors source-chain `CCIPMessageSent` events. The only remaining on-chain defense is the RMN curse check against the immutable `i_rmnProxy`, but here it is unlikely to trigger since it is watching for the remoteChainSelector, and most of the bridged chains are not cursed at all:

```solidity
if (IRMN(i_rmnProxy).isCursed(bytes16(uint128(releaseOrMintIn.remoteChainSelector)))) revert CursedByRMN();
```

---

### 2. `applyChainUpdates`

```solidity
function applyChainUpdates(
    uint64[] calldata remoteChainSelectorsToRemove,
    ChainUpdate[] calldata chainsToAdd
) external virtual onlyOwner {
    // REMOVAL: immediately purges chain config
    for (uint256 i = 0; i < remoteChainSelectorsToRemove.length; ++i) {
        if (!s_remoteChainSelectors.remove(remoteChainSelectorToRemove))
            revert NonExistentChain(remoteChainSelectorToRemove);
        delete s_remoteChainConfigs[remoteChainSelectorToRemove]; // ← wipes rate limits + remote token
        emit ChainRemoved(remoteChainSelectorToRemove);
    }

    // ADDITION: initializes the token bucket with capacity as the starting tokens
    remoteChainConfig.outboundRateLimiterConfig = RateLimiter.TokenBucket({
        rate: newChain.outboundRateLimiterConfig.rate,
        capacity: newChain.outboundRateLimiterConfig.capacity,
        tokens: newChain.outboundRateLimiterConfig.capacity, // ← starts full
        lastUpdated: uint32(block.timestamp),
        isEnabled: newChain.outboundRateLimiterConfig.isEnabled
    });
    remoteChainConfig.remoteTokenAddress = newChain.remoteTokenAddress; // ← destination token binding
}
```

**Impact:** 

The owner could configure `remoteTokenAddress` for a remote chain, where it is used inside `lockOrBurn` as `destTokenAddress`:

```solidity
return Pool.LockOrBurnOutV1({
    destTokenAddress: getRemoteToken(lockOrBurnIn.remoteChainSelector), // ← set by owner here
    destPoolData: _encodeLocalDecimals()
});
```

A wrong `remoteTokenAddress` means the CCIP message instructs the destination to release the wrong token entirely. However this could not be used to drain a pool, since it is only used by the `offRamp` to find the corresponding pool contract.

Removing a chain triggers `delete s_remoteChainConfigs[...]`, which causes `isSupportedChain()` to return `false`, making all subsequent `_onlyOnRamp` / `_onlyOffRamp` calls revert with `ChainNotAllowed` .

---

### 3. `addRemotePool` / `removeRemotePool`

```solidity
function addRemotePool(uint64 remoteChainSelector, bytes calldata remotePoolAddress) external onlyOwner {
    if (!isSupportedChain(remoteChainSelector)) revert NonExistentChain(remoteChainSelector);
    _setRemotePool(remoteChainSelector, remotePoolAddress);
}

function removeRemotePool(uint64 remoteChainSelector, bytes calldata remotePoolAddress) external onlyOwner {
    if (!isSupportedChain(remoteChainSelector)) revert NonExistentChain(remoteChainSelector);
    if (!s_remoteChainConfigs[remoteChainSelector].remotePools.remove(keccak256(remotePoolAddress))) {
        revert InvalidRemotePoolForChain(remoteChainSelector, remotePoolAddress);
    }
    emit RemotePoolRemoved(remoteChainSelector, remotePoolAddress);
}
```

**Impact:** Both functions manage the whitelist of trusted source pools enforced in `_validateReleaseOrMint` via the `isRemotePool` check. 

Removing a remote pool address means any in-flight CCIP message claiming to originate from that pool will revert with `InvalidSourcePoolAddress` when the OffRamp attempts to execute `releaseOrMint` on the destination. However, no direct theft is possible through this function alone,  service is trivially restored by re-adding the pool address.

**Attack Scenario:**
Imagine two pools connected via a CCIP bridge: `ValuablePoolA` (handling `ValuableTokenA` on Chain A) and `ValuablePoolB` (on Chain B).

**Step 1: Compromise and Setup**

1. The attacker manages to steal the `owner` private key for `ValuablePoolA` on Chain A.
2. The attacker deploys a `FakeToken` contract and a corresponding `FakePool` on Chain B.
3. The attacker mints a massive amount of `FakeToken` to their own wallet on Chain B (at zero cost).

**Step 2: Configuration Hijack (Chain A)**
Using the compromised `owner` key, the attacker calls `applyChainUpdates` on `ValuablePoolA` (Chain A) to maliciously alter the bridge configuration for Chain B:

- They set the `remoteTokenAddress` to the address of `FakeToken`.
- They add `FakePool` to the `remotePoolAddresses` array (registering it as a trusted remote pool).

**Step 3: Execution (Chain B)**
The attacker initiates a standard cross-chain transfer on Chain B by interacting with the CCIP Router, locking or burning their `FakeToken` via their `FakePool`.

**Step 4: DON Attestation**
The CCIP DON monitors Chain B. The DON is agnostic to the economic value of tokens; its job is strictly to verify that a state transition occurred. Because the attacker successfully locked/burned the tokens inside `FakePool` and the local `OnRamp` processed it, the DON attests to the validity of the cross-chain message and commits it to Chain A.

**Step 5: Bypassing Validation and Draining Funds (Chain A)**
When the CCIP `OffRamp` on Chain A receives the attested message, it calls `releaseOrMint` on `ValuablePoolA`. The internal `_validateReleaseOrMint` function executes the following critical check:

```solidity
if (!isRemotePool(releaseOrMintIn.remoteChainSelector, releaseOrMintIn.sourcePoolAddress)) {
    revert UntrustedRemotePool(releaseOrMintIn.sourcePoolAddress);
}
```

Because the attacker previously registered `FakePool` as a trusted remote pool via `applyChainUpdates`, this check **passes successfully**. `ValuablePoolA` then processes the payload and releases or mints `ValuableTokenA` to the attacker, matching the massive amount of `FakeToken` burned on Chain B.

---

### 4. `setRateLimitAdmin`

```solidity
function setRateLimitAdmin(address rateLimitAdmin) external onlyOwner {
    s_rateLimitAdmin = rateLimitAdmin;
    emit RateLimitAdminSet(rateLimitAdmin);
}
```

**Impact:** 

`s_rateLimitAdmin` is checked alongside `owner()` in rate limit functions:

```solidity
function setChainRateLimiterConfigs(...) external {
    if (msg.sender != s_rateLimitAdmin && msg.sender != owner())
        revert Unauthorized(msg.sender); // ← either can call
    ...
}
```

---

### 5. `setChainRateLimiterConfig`

```solidity
function _setRateLimitConfig(
    uint64 remoteChainSelector,
    RateLimiter.Config memory outboundConfig,
    RateLimiter.Config memory inboundConfig
) internal {
    if (!isSupportedChain(remoteChainSelector)) revert NonExistentChain(remoteChainSelector);
    RateLimiter._validateTokenBucketConfig(outboundConfig);
    s_remoteChainConfigs[remoteChainSelector].outboundRateLimiterConfig._setTokenBucketConfig(outboundConfig);
    RateLimiter._validateTokenBucketConfig(inboundConfig);
    s_remoteChainConfigs[remoteChainSelector].inboundRateLimiterConfig._setTokenBucketConfig(inboundConfig);
    emit ChainConfigured(remoteChainSelector, outboundConfig, inboundConfig);
}
```

**Impact:** These token buckets are consumed on every transfer:

```solidity
function _consumeOutboundRateLimit(uint64 remoteChainSelector, uint256 amount) internal {
    s_remoteChainConfigs[remoteChainSelector].outboundRateLimiterConfig._consume(amount, address(i_token));
}

function _consumeInboundRateLimit(uint64 remoteChainSelector, uint256 amount) internal {
    s_remoteChainConfigs[remoteChainSelector].inboundRateLimiterConfig._consume(amount, address(i_token));
}
```

`_consume` will revert if the bucket is empty. Setting `capacity` and `rate` too low throttles legitimate users; setting `isEnabled = false` removes the cap entirely, meaning an exploit could drain the pool in a single transaction with no throughput ceiling.

---

### 6. `applyAllowListUpdates`

```solidity
function applyAllowListUpdates(address[] calldata removes, address[] calldata adds) external onlyOwner {
    _applyAllowListUpdates(removes, adds);
}

function _applyAllowListUpdates(address[] memory removes, address[] memory adds) internal {
    if (!i_allowlistEnabled) revert AllowListNotEnabled(); // ← immutable flag, set at deploy
    for (uint256 i = 0; i < adds.length; ++i) {
        if (toAdd == address(0)) continue;
        if (s_allowlist.add(toAdd)) emit AllowListAdd(toAdd);
    }
}
```

**Impact:** The allowlist check is enforced inside `_validateLockOrBurn`, which runs on every outbound transfer:

```solidity
function _validateLockOrBurn(Pool.LockOrBurnInV1 calldata lockOrBurnIn) internal {
    if (!isSupportedToken(lockOrBurnIn.localToken)) revert InvalidToken(lockOrBurnIn.localToken);
    if (IRMN(i_rmnProxy).isCursed(...)) revert CursedByRMN();
    _checkAllowList(lockOrBurnIn.originalSender); // ← allowlist checked here
    _onlyOnRamp(lockOrBurnIn.remoteChainSelector);
    _consumeOutboundRateLimit(...);
}

function _checkAllowList(address sender) internal view {
    if (i_allowlistEnabled) {
        if (!s_allowlist.contains(sender)) revert SenderNotAllowed(sender); // ← reverts if not listed
    }
}
```

Note `i_allowlistEnabled` is **immutable** — set once at constructor time and can never be toggled. So if a pool was deployed with an empty allowlist (`allowlist.length == 0`), `applyAllowListUpdates` is permanently disabled.

## Decimal Handling

Starting **from v1.5.1**, token pools support tokens with different decimal places across blockchains. 

When tokens move between blockchains with different decimal places, rounding can occur due to a loss of precision. This rounding can affect small amounts of tokens during cross-chain transfers.

**Impact**

Consider a token developer who deployed their token across three blockchains with different decimal configurations:

- Blockchain A: High precision (18 decimals)
- Blockchain B: Low precision (9 decimals)
- Blockchain C: High precision (18 decimals)

| Scenario | Transfer Path | Example | Impact |
| --- | --- | --- | --- |
| High to Low Precision ❌ | A → B |   • Send from A: 1.123456789123456789.
  • Receive on B: 1.123456789 | Lost: 0.000000000123456789
  • Burn/mint: Tokens permanently burned on Blockchain A
  • Lock/release: Tokens locked in pool on Blockchain A |
| Low to High Precision ✅ | B → A |   • Send from B: 1.123456789
  • Receive on A: 1.123456789000000000 |   • No precision loss |
| Equal Precision ✅ | A → C |   • Send from A: 1.123456789123456789
  • Receive on C: 1.123456789123456789 |   • No precision loss |

**Best Practices**

- Deploy tokens with the same number of decimals across all blockchains whenever possible
- This prevents any loss of precision during cross-chain transfers
- Different decimals should only be used when required by blockchain limitations (e.g., non-EVM chains with decimal constraints)
- Verify decimal configurations on both source and destination blockchains before transfers
- Consider implementing UI warnings for transfers that might be affected by rounding
- When using high-to-low precision transfers, be aware that:
    - In burn/mint pools: Lost precision results in permanently burned tokens
    - In lock/release pools: Lost precision results in tokens accumulating in the source pool

# **tokenAdminRegistry**

Source: [https://github.com/smartcontractkit/chainlink-ccip/tree/contracts-ccip-v1.6.1/chains/evm/contracts/tokenAdminRegistry](https://github.com/smartcontractkit/chainlink-ccip/tree/contracts-ccip-v1.6.1/chains/evm/contracts/tokenAdminRegistry)

## Who Can register a token pool

Registration is not performed by the Chainlink team; it is performed by the **Token Administrator**. To become the administrator, you must prove to the `TokenAdminRegistry` that you are the legitimate owner of the token.

There are two primary ways to initiate registration:

- **For New Tokens (Using `TokenPoolFactory`):**
If you are creating a new token and its pool simultaneously, the `TokenPoolFactory` handles the heavy lifting.
    1. The factory deploys your token and your pool.
    2. It uses the `RegistryModuleOwnerCustom` to register you as the admin.
    3. It automatically links the pool to the token in the `TokenAdminRegistry`.
    4. The factory then transfers ownership of both the token and the pool back to you (`IOwnable(token).transferOwnership(msg.sender)`).
- **For Existing Tokens (Using `RegistryModuleOwnerCustom`):**
If your token already exists, you must register it manually via `RegistryModuleOwnerCustom`. This contract checks for three standard ownership patterns:
    - **`registerAdminViaGetCCIPAdmin`**: Checks if your token has a custom `getCCIPAdmin()` function.
    - **`registerAdminViaOwner`**: Checks if you are the `owner()` of the token.
    - **`registerAccessControlDefaultAdmin`**: Checks if you hold the `DEFAULT_ADMIN_ROLE` (using OpenZeppelin AccessControl).
    - If you pass these checks, the module calls `TokenAdminRegistry.proposeAdministrator`, and you must then call `acceptAdminRole` to finalize your status.

---

## Administrator Privileges

### `setPool`

```solidity
function setPool(address localToken, address pool) external onlyTokenAdmin(localToken)
```

It tells the entire CCIP network which smart contract is responsible for handling your tokens on that specific chain.

**Impact:**

In a Lock/Release pool, the pool holds a reserve of tokens. When users bridge *in*, the pool releases tokens to them.

- **The Attack:** The attacker sets the pool to a malicious address (`MaliciousPool`).
- **The Exploit:**
    1. A legitimate user sends tokens from Chain A to Chain B.
    2. The CCIP `OffRamp` on Chain B receives the valid proof and calls `MaliciousPool.releaseOrMint`.
    3. Instead of sending the tokens to the `receiver` specified in the message, the `MaliciousPool` code is written to ignore the `receiver` argument and execute: `IERC20(token).transfer(AttackerAddress, amount)`.
- **Result:** The user's cross-chain transfer is "successful" according to the protocol (the transaction finalized), but the tokens are drained to the attacker’s wallet rather than the user's. The locked liquidity in the bridge is systematically siphoned out.

However, for Burn/Mint mechanism, in order to mint token, a pool have to be granted by the token owner with the minter role, so malicious pool cannot do infinite minting without this role.

### `transferAdminRole`

```solidity
  function transferAdminRole(address localToken, address newAdmin) external onlyTokenAdmin(localToken) {
    TokenConfig storage config = s_tokenConfig[localToken];
    config.pendingAdministrator = newAdmin;

    emit AdministratorTransferRequested(localToken, msg.sender, newAdmin);
  }
```

Transfers the administrator role for a token to a new address with a 2-step process.

The new admin must call `acceptAdminRole` to accept the role.