---
title: Nest (RWA Earn) - Part 4
published: 2026-02-12
pinned: false
description: Auditing
tags: [Smart Contract,RWA]
category: RWA
draft: false
---

## Table of Contents

1. [Invariant Violations](#invariant-violations)
2. [Issues](#issues)
    - [[INFO-1] Instant Redemption Fee Not Explicitly Accounted For](#info-1-instant-redemption-fee-not-explicitly-accounted-for)
    - [[INFO-2] Misleading Comment in `updateRedeem` Function](#info-2-misleading-comment-in-updateredeem-function)
    - [[INFO-3] Fee-on-Transfer Tokens Incompatibility](#info-3-fee-on-transfer-tokens-incompatibility)
    - [[INFO-4] Missing Vault Authorization Check in `NestShareOFT.enter()`](#info-4-missing-vault-authorization-check-in-nestshareoftenter)

---

## Invariant Violations

1. **Share Conservation:**
    
    ```
    totalSupply() == Σ(userBalances) + pendingRedemptions + burnedSharesAcrosschains
    ```
    
    **Violation Risk:** Cross-chain minting without burning on source
    **Detection:** Monitor OFT message delivery failures
    
2. **Asset Backing:**
    
    ```
    shareContract.balanceOf(asset) >= (totalSupply - totalPendingShares) * rate
    ```
    
    **Violation Risk:** Assets withdrawn via `manage()` without burning shares
    **Detection:** Monitor `manage()` calls, track asset balance
    
3. **Rate Bounds:**
    
    ```
    minRate <= exchangeRate <= UPPER_BOUND_RATE_CAP
    ```
    
    **Violation Risk:** Oracle returning extreme values
    **Detection:** `updateExchangeRate` will revert or pause
    
4. **Fee Accumulation:**
    
    ```
    feesOwedInBase <= totalAssets * MAX_MANAGEMENT_FEE * (block.timestamp - inception) / ONE_YEAR
    ```
    
    **Violation Risk:** Fee calculation overflow
    **Detection:** Audit fee accrual logic
    

---

## Issues

### [INFO-1] Instant Redemption Fee Not Explicitly Accounted For

**Severity:** Informational

**Contract:** `NestVaultCore.sol`

**Location:** [Line 547](https://github.com/plumenetwork/nest-protocol/blob/main/contracts/NestVaultCore.sol#L547)

**Description:**

In the `instantRedeem` function, the redemption fee is calculated as follows:

```solidity
(_postFeeAmount, _feeAmount) = _convertToAssetsForInstantRedeem(_shares);
```

Where:

- `_feeAmount`: The fee charged (in asset tokens)
- `_postFeeAmount`: Net assets after fee deduction (sent to user)

Subsequently, `_postFeeAmount` is transferred to `_receiver` and `_shares` are burned:

```solidity
_exit(_receiver, _asset, _postFeeAmount, address(this), _shares);
```

**Issue:**

The `_feeAmount` remains in the Share contract but is not:

- Explicitly transferred to a treasury address
- Tracked in a dedicated protocol revenue variable
- Clearly documented as being retained to increase share value for remaining holders

While the fee effectively benefits remaining shareholders by increasing the exchange rate (similar to a token buyback), this mechanism lacks explicit accounting and documentation.

**Current Behavior:**

```solidity
// Example: User redeems 1000 shares
GrossAssets = 1050 USDC (1000 shares * 1.05 rate)
FeeAmount = 5.25 USDC (0.5% fee)
NetAmount = 1044.75 USDC (sent to user)

// After redemption:
// - 1044.75 USDC sent to user ✅
// - 1000 shares burned ✅
// - 5.25 USDC stays in Share contract (implicit benefit to holders)
```

**Recommendation:**

Add explicit documentation and/or accounting:

```solidity
/// @notice Instant redemption fee remains in Share contract,
///         accreting value to remaining shareholders
/// @dev    The fee is NOT sent to treasury but increases the exchange rate
///         for all remaining share holders (similar to a buyback)
function instantRedeem(...) external returns (...) {
    // ... existing code

    // Option 1: Add event for transparency
    emit InstantRedemptionFeeRetained(_feeAmount);

    // Option 2: Track cumulative fees (for analytics)
    _getNestVaultCoreStorage().cumulativeInstantRedemptionFees += _feeAmount;
}
```

**Impact:** Low - The current implementation is economically sound but lacks transparency in fee accounting.

---

### [INFO-2] Misleading Comment in `updateRedeem` Function

**Severity:** Informational

**Contract:** `NestVaultCore.sol`

**Location:** [Line 582](https://github.com/plumenetwork/nest-protocol/blob/main/contracts/NestVaultCore.sol#L582)

**Description:**

The function documentation for `updateRedeem` contains a misleading statement:

```solidity
/// @notice Update the number of shares in an existing redeem request
/// @dev    Allows the owner or an authorized operator to decrease the pending redeem amount
///         - If `_newShares` is lower, excess shares are returned to the owner
///         - If `_newShares` is higher, additional shares are pulled from the owner  // ❌ Incorrect
```

**Issue:**

The comment states that if `_newShares` is higher, additional shares will be pulled from the owner. However, the actual implementation **reverts** if a user attempts to increase their redemption amount:

```solidity
if (_oldShares < _newShares) {
    revert Errors.INSUFFICIENT_BALANCE();  // ❌ Cannot increase
}
```

**Correct Behavior:**

Users can **only decrease** their pending redemption amount, not increase it.

**Recommendation:**

Update the documentation to reflect the actual implementation:

```solidity
/// @notice Update the number of shares in an existing redeem request
/// @dev    Allows the owner or an authorized operator to decrease the pending redeem amount
///         - If `_newShares` is lower, excess shares are returned to the owner
///         - If `_newShares` is higher, the function will revert
///         - Users cannot increase their redemption requests, only decrease
///         - To increase, users must submit a new requestRedeem() transaction
```

**Impact:** Low - Documentation issue only, does not affect contract security or functionality.

---

### [INFO-3] Fee-on-Transfer Tokens Incompatibility

**Severity:** Informational

**Contract:** `NestVaultCore.sol`

**Location:** [Lines 490-492](https://github.com/plumenetwork/nest-protocol/blob/main/contracts/NestVaultCore.sol#L490-L492)

**Description:**

The contract implements a balance validation check to protect users from receiving fewer assets than expected:

```solidity
// Check if the amount transferred matches the predicted amount
uint256 actualTransfer = finalBalance - initialBalance;

if (actualTransfer < _assets) {
    revert Errors.TRANSFER_INSUFFICIENT();
}
```

**Issue:**

If the underlying asset is a **fee-on-transfer** token (tokens that automatically deduct a fee during transfers, e.g., certain reflection tokens or deflationary tokens), the `actualTransfer` will always be less than `_assets`, causing all deposits and withdrawals to revert.

**Example Scenario:**

```solidity
// Using a fee-on-transfer token with 1% transfer fee
_assets = 1000e6  // Expected transfer

// Actual transfer:
initialBalance = 0
finalBalance = 990e6  // Only 99% received due to 1% fee
actualTransfer = 990e6

// Validation check:
if (990e6 < 1000e6) {  // ✅ True
    revert Errors.TRANSFER_INSUFFICIENT();  // ❌ Reverts
}
```

**Current Protection Targets:**

This check is designed to protect against:

- Malicious tokens that don't transfer the full amount
- Reentrancy attacks that manipulate balances
- Accounting errors

**Affected Functions:**

Similar checks exist in:

- `fulfillRedeem()` - Line 490
- `_processInstantRedeem()` - Balance validation

**Recommendation:**

1. **Document the limitation explicitly:**

```solidity
/// @notice This vault is INCOMPATIBLE with fee-on-transfer tokens
/// @dev    Balance validation checks will cause reverts for tokens that
///         deduct fees during transfer. Only use standard ERC20 tokens.
contract NestVaultCore {
    // ...
}
```

1. **Add deployment-time validation:**

```solidity
function __NestVaultCore_init(...) internal onlyInitializing {
    // ... existing checks

    // Validate asset is not fee-on-transfer
    _validateAssetTransfer(_asset);
}

function _validateAssetTransfer(address _asset) internal {
    uint256 balanceBefore = ERC20(_asset).balanceOf(address(this));

    // Transfer 1 wei to self
    SafeTransferLib.safeTransferFrom(
        ERC20(_asset),
        msg.sender,
        address(this),
        1
    );

    uint256 balanceAfter = ERC20(_asset).balanceOf(address(this));

    if (balanceAfter - balanceBefore != 1) {
        revert Errors.FEE_ON_TRANSFER_NOT_SUPPORTED();
    }
}
```

1. **Maintain a whitelist of approved assets:**

```solidity
mapping(address => bool) public approvedAssets;

function setApprovedAsset(address _asset, bool _approved) external requiresAuth {
    approvedAssets[_asset] = _approved;
}

// Check in initialization
require(approvedAssets[_asset], "Asset not approved");
```

**Incompatible Token Types:**

- ❌ Fee-on-transfer tokens (e.g., SAFEMOON, REFLECT)
- ❌ Rebasing tokens (e.g., AMPL, stETH with balance changes)
- ❌ Tokens with transfer limits or blacklists
- ✅ Standard ERC20 tokens (USDC, USDT, DAI, WETH)

**Impact:** Medium - If fee-on-transfer tokens are used, the vault becomes completely non-functional. However, this can be easily mitigated through proper asset selection during deployment.

---

### [INFO-4] Missing Vault Authorization Check in `NestShareOFT.enter()`

**Severity:** Informational

**Contract:** `NestShareOFT.sol`

**Location:** [Line 148](https://github.com/plumenetwork/nest-protocol/blob/main/contracts/NestShareOFT.sol#L148)

**Description:**

The `enter()` function in `NestShareOFT` is responsible for minting shares when users deposit assets. Currently, it only checks `requiresAuth` which verifies if the caller has the `MINTER_ROLE`, but does not verify that the caller is the **authorized vault for the specific asset being deposited**.

**Current Implementation:**

```solidity
function enter(
    address from,
    ERC20 asset,
    uint256 assetAmount,
    address to,
    uint256 shareAmount
) external requiresAuth {  // ✅ Checks role, ❌ doesn't check vault-asset mapping
    // Transfer assets in
    if (assetAmount > 0) asset.safeTransferFrom(from, address(this), assetAmount);

    // Mint shares
    _mint(to, shareAmount);

    emit Enter(from, address(asset), assetAmount, to, shareAmount);
}
```

**Issue:**

The function does not validate that `msg.sender` (the vault calling `enter`) is the authorized vault for the specific `asset` being deposited. This could potentially allow:

1. A vault authorized for USDC to mint shares when depositing USDT (if both vaults have MINTER_ROLE)
2. Cross-contamination between different asset pools
3. Incorrect share minting if multiple vaults share the same Share token

**Example Scenario:**

```solidity
// Setup
vault_USDC.setVault(USDC, vault_USDC);  // USDC vault
vault_USDT.setVault(USDT, vault_USDT);  // USDT vault

// Both vaults have MINTER_ROLE on the same Share contract

// ❌ Problem: USDC vault can mint shares for USDT deposits
vault_USDC.enter(user, USDT, 1000e6, user, shares);
// This passes requiresAuth ✅
// But USDT should only be accepted by vault_USDT ❌
```

**Recommendation:**

Add explicit vault-asset authorization check:

```solidity
function enter(
    address from,
    ERC20 asset,
    uint256 assetAmount,
    address to,
    uint256 shareAmount
) external requiresAuth {
    // ✅ Add vault authorization check
    address authorizedVault = _getNestShareOFTStorage().vault[address(asset)];
    if (msg.sender != authorizedVault) {
        revert Errors.UnauthorizedVaultForAsset(msg.sender, address(asset));
    }

    // Transfer assets in
    if (assetAmount > 0) asset.safeTransferFrom(from, address(this), assetAmount);

    // Mint shares
    _mint(to, shareAmount);

    emit Enter(from, address(asset), assetAmount, to, shareAmount);
}
```

**Alternative Implementation (Gas Optimized):**

```solidity
function enter(
    address from,
    ERC20 asset,
    uint256 assetAmount,
    address to,
    uint256 shareAmount
) external requiresAuth {
    NestShareOFTStorage storage $ = _getNestShareOFTStorage();

    // Combined check: caller has auth AND is the registered vault for this asset
    if ($.vault[address(asset)] != msg.sender) {
        revert Errors.CallerNotAuthorizedVault();
    }

    // ... rest of the function
}
```

**Add Corresponding Error:**

```solidity
// In Errors.sol
error UnauthorizedVaultForAsset(address caller, address asset);
error CallerNotAuthorizedVault();
```

**Impact:** Medium (if multi-vault setup is used) / Low (if single vault per Share token). Adds important defense-in-depth validation that aligns with the vault-asset mapping design pattern already implemented in the contract.