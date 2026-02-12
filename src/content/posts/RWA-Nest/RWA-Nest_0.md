---
title: Analysis Report of Nest (RWA Earn) - Draft
published: 2026-02-08
pinned: false
description: This is just a draft; please refer to the updated version instead.
tags: [Web3,Dev Handbook,RWA]
category: RWA
draft: false
image: ./cover.png
---
# 1. Scope

- NestVault.sol
- NestVaultCore.sol

Note that we did not include the `NestAccountant.sol` or `NestShareOFT.sol` in the current scope, and we will neglect their internal code logic for now. 

During auditing, we are currently **assuming** that all external dependencies are implemented correctly and behave as intended (i.e., external calls to `NestAccount.sol` and `NestShareOFT.sol` ).

---

# 2. System Overview & Architecture

## 2.1 Protocol Design

The `NestVaultCore` contract represents a hybrid vault architecture designed to bridge standard DeFi interfaces with the complexities of Real-World Assets (RWA). It extends the widely adopted **ERC-4626** standard (Tokenized Vaults) by integrating **ERC-7540** (Asynchronous Vaults) to manage the liquidity constraints often inherent in RWA backing.

### Core Concepts

- **Asynchronous Settlement (ERC-7540):** Unlike typical DeFi vaults where liquidity is assumed to be instantly available, this protocol assumes underlying assets may be illiquid or require manual processing time to unwind.
    - **Request Phase:** Users cannot simply swap Shares for Assets. They must first initiate a *Redemption Request*, locking their shares in a "Pending" state.
    - **Fulfillment Phase:** An authorized entity (Admin) processes these requests.
    - **Claim Phase:** Once fulfilled, the assets move to a "Claimable" state, specific to the user (Controller), ready for final withdrawal.
- **Decoupled Share Logic (ERC-7575):**
A critical architectural distinction is that `NestVaultCore` does **not** inherit ERC-20 logic directly to represent the Vault Shares. Instead, it delegates all token operations (minting, burning, transfer) to an external contract, `NestShareOFT`.
    - This design leverages **LayerZero's Omnichain Fungible Token (OFT)** standard, enabling the vault shares to be cross-chain native.
    - The Vault acts as the *primary controller* of this external token, holding the exclusive right to mint or burn shares via the `enter` and `exit` hooks during deposit and redemption flows. This allows share liquidity to move across chains while the backing assets remain secured on the host chain.
- **Separation of Pricing and Reserves:**
The vault does not calculate exchange rates based on the simplistic `totalAssets / totalSupply` formula common in AMMs. Instead, it queries an external `AccountantWithRateProviders`. This allows the protocol to inject complex pricing logic.

---

## 2.2 Roles & Security

The `NestVaultCore` security model is **heavily centralized**, employing a "Walled Garden" approach. The Vault Administrator holds absolute power to freeze operations, whitelist users, and control asset pricing. The safety of user funds relies entirely on the operational security (OpSec) of the protocol administrators and the integrity of the external Accountant oracle.

The protocol separates asset ownership from operational management through a granular access control model. This hierarchy is divided into two layers: a **Global Permission Layer** (The Authority) and a **Local Delegation Layer** (Owner/Controller/Operator).

### 1. The Authority (Global Admin)

The Authority is the protocol's supreme gatekeeper, managing the compliance firewall.

- **Mechanism:** Enforced via the `requiresAuth` modifier and the `isAuthorized` check in `_deposit`.
- **Capabilities:**
    - **Access Control:** Can whitelist or blacklist any address (User or Bot) from calling `deposit`, `withdraw`, `requestRedeem`, or `instantRedeem`.
    - **Financial Settlement:** Solely authorized to call `fulfillRedeem` to settle pending requests.
    - **Protocol Configuration:** Can modify fee percentages (`setFee`), change the pricing oracle (`setAccountantWithRateProviders`), and set the `minRate` safety floor.
- **Security Implication:** This role represents a total trust assumption. If the Authority is compromised or the contract is misconfigured, all user funds can be effectively frozen, as no entry or exit is possible without Global Admin approval.

### 2. The Owner (Capital Provider)

The Owner is the legal holder of the vault shares (`NestShareOFT`).

- **Capabilities:**
    - **Asset Control:** Must grant an ERC-20 allowance to the Vault to move shares.
    - **Delegation:** Can appoint one or more **Operators** to manage their shares via `setOperator` or by signing an EIP-712 `authorizeOperator` message.
    - **Source of Funds:** Acts as the only address from which shares can be pulled during a `requestRedeem` or `instantRedeem` transaction.

### 3. The Controller (Redemption Manager)

The Controller is the "Account Identity" used for the vault’s internal ledger.

- **Capabilities:**
    - **Position Ownership:** Owns the record in the `pendingRedeem` (pending assets) and `claimableRedeem` (ready-to-withdraw assets) mappings.
    - **Delegation:** Like the Owner, the Controller can authorize an **Operator** to manage their claimable assets.
- **Context:** This role enables institutional flexibility. An Owner can provide the capital, while a separate Controller (often a smart contract) manages the investment strategy and exit timing.

### 4. The Operator (Execution Delegate)

The Operator is the functional "Hand" of the system, typically an automation bot or a specialized management address.

- **Capabilities:**
    - **Execution:** Can call `requestRedeem`, `updateRedeem`, `withdraw`, and `redeem` on behalf of an Owner or Controller.
    - **Gas Management:** Can pay for transaction fees so the Owner/Controller does not need to maintain an active hot-wallet balance.
- **The "Dual-Lock" Constraint:** An Operator is the most restricted role. To execute any state-changing action, they must pass two simultaneous checks:
    1. **Global Lock (`requiresAuth`):** The Protocol Admin must have whitelisted the Operator's address in the `Authority` contract.
    2. **Local Lock (`_isAuthorizedCaller`):** The Owner or Controller must have granted the Operator permission within the Vault's internal `isOperator` mapping.

| Action | Required Global Auth | Required Local Auth |
| --- | --- | --- |
| **Deposit** | Authority Whitelist | Caller is Receiver (or authorized) |
| **Request Redeem** | Authority Whitelist | Caller is Operator for **Owner** |
| **Fulfill Redeem** | Authority Whitelist | N/A (Admin only) |
| **Withdraw/Redeem** | Authority Whitelist | Caller is Operator for **Controller** |
| **Update Redeem** | Authority Whitelist | Caller is Operator for **Owner AND Controller** |

---

## 2.3 Life Cycle & Money Flow

### 2.3.1 Deposit: User(Assets) → Vault → Share (Mint) → User

The deposit workflow begins when a user invokes the `deposit` function. This triggers a standard ERC-4626 flow that is intercepted by `NestVaultCore` to enforce access control and execute custom minting logic via an external contract.

**Step 1: Entry Point**
The user calls the public `deposit` function. 

```solidity
// NestVault.sol
function deposit(uint256 _assets, address _receiver) public virtual override returns (uint256) {
    return super.deposit(_assets, _receiver);
}

```

**Step 2: Calculation**
The call is passed to the parent `ERC4626Upgradeable` contract. This standard implementation calculates the exchange rate via `previewDeposit` and ensures deposit limits are respected.

```solidity
// ERC4626Upgradeable.sol
function deposit(uint256 assets, address receiver) public virtual returns (uint256) {
    uint256 maxAssets = maxDeposit(receiver);
    if (assets > maxAssets) {
        revert ERC4626ExceededMaxDeposit(receiver, assets, maxAssets);
    }

    // Critical: Calculates share amount based on custom exchange rate (see Step 3)
    uint256 shares = previewDeposit(assets);
    _deposit(_msgSender(), receiver, assets, shares);

    return shares;
}

```

**Step 3: Custom Exchange Rate Logic**
Unlike standard vaults that use the ratio of `TotalAssets / TotalSupply`, `NestVaultCore` overrides the conversion logic to rely on an external oracle/accountant. 

```solidity
// NestVaultCore.sol
function _convertToShares(uint256 _assets, Math.Rounding _rounding) internal view override returns (uint256) {
    // Exchange rate is determined by the external Accountant, not the vault's current balance
    return Math.mulDiv(_assets, ONE_SHARE, _getValidatedRate(), _rounding);
}

```

**Step 4: Authorization & Execution**
The internal `_deposit` function performs a secondary authorization check (`isAuthorized`) before delegating the asset movement to `_enter`. Therefore, the user has to do things like KYC in order to be able to call the deposit function; otherwise, it will revert.

```solidity
// NestVaultCore.sol
function _deposit(address _caller, address _receiver, uint256 _assets, uint256 _shares) internal virtual override {
    // Security Check: Ensures the caller has permission to interact with the vault
    if (!isAuthorized(_msgSender(), msg.sig)) {
        revert Errors.UNAUTHORIZED();
    }

    _enter(_caller, _receiver, _assets, _shares);
    emit Deposit(_caller, _receiver, _assets, _shares);
}

```

**Step 5: Settlement**
Finally, the `_enter` function executes the physical transfer of funds. It pulls assets from the user into the Vault, approves the external Share contract, and triggers the minting of `NestShareOFT` tokens to the receiver.

```solidity
// NestVaultCore.sol
function _enter(address _caller, address _receiver, uint256 _assets, uint256 _shares) internal virtual {
    ERC20 assetToken = ERC20(asset());

    // 1. Pull Assets: User -> Vault
    SafeTransferLib.safeTransferFrom(assetToken, _caller, address(this), _assets);

    // 2. Approve & Mint: Vault delegates minting to the external SHARE contract
    SafeERC20.forceApprove(IERC20(address(assetToken)), address(SHARE), _assets);
    SHARE.enter(address(this), assetToken, _assets, _receiver, _shares);
    SafeERC20.forceApprove(IERC20(address(assetToken)), address(SHARE), 0);
}

```

**Result:** 

The User successfully exchanges underlying assets for `NestShareOFT` tokens at a rate determined by the `Accountant`.

**Note:**

There is also a `mint` function which allows the user to deposit, the difference between `mint` and `deposit` is the how they decide the input, the rest will be the same.

---

### 2.3.2 Request Redeem: Shares → Vault (Locked)

> NestVaultCore.sol
> 

The Request Redeem phase initiates the asynchronous withdrawal process. In this stage, shares are moved from the user's wallet into the Vault's custody, and a record is created in the internal ledger to track the pending obligation.

**Step 1: Entry Point & Asset Escrow**

The operation begins when an authorized caller (either the **Owner** or an **Operator**) invokes `requestRedeem`. At this stage, the Vault pulls the shares from the Owner’s balance and holds them in escrow.

```solidity
// NestVaultCore.sol
function requestRedeem(uint256 _shares, address _controller, address _owner)
    external
    override
    requiresAuth // Check 1: Global Authorization
    returns (uint256 _requestId)
{
    if (_controller == address(0)) revert Errors.ZERO_ADDRESS();

    // Check 2: Local Authorization
    _isAuthorizedCaller(_owner);

    if (SHARE.balanceOf(_owner) < _shares) revert Errors.INSUFFICIENT_BALANCE();
    if (_shares == 0) revert Errors.ZERO_SHARES();

    // MONEY FLOW: Shares move from the Owner's wallet into the Vault's custody
    SafeTransferLib.safeTransferFrom(ERC20(address(SHARE)), _owner, address(this), _shares);

    _requestId = _processRedeemRequest(_shares, _controller, _owner);
}

```

**Note on Money Flow:** The line `SafeTransferLib.safeTransferFrom(...)` is the critical "Locking" mechanism. The shares are no longer in the Owner's wallet, but they are not yet burned; they are held by the Vault contract itself.

**Step 2: The "Double-Gate" Authorization Check**

Security in this phase is maintained by two distinct permission checks, ensuring that only KYC'd entities and authorized managers can initiate a redemption.

1. **`requiresAuth` (Global Level):** Validates that the `msg.sender` (the caller) is whitelisted in the protocol's `Authority` contract. This ensures that the caller (typically a Relayer or a KYC'd user) is allowed to interact with the protocol.
2. **`_isAuthorizedCaller(_owner)` (Account Level):** Validates that the caller has the specific right to move the `_owner`'s shares. This is true if the caller is the `_owner` themselves or an `Operator` previously approved by the `_owner`.

**Step 3: State Transition & Accounting**

Once the shares are locked, the Vault updates its internal state to reflect the new pending obligation. The `_processRedeemRequest` function updates two critical storage variables:

1. **`pendingRedeem[_controller]`**: An internal ledger that tracks how many shares are currently owed to a specific **Controller**.
2. **`totalPendingShares`**: A global counter tracking all locked shares across the vault. This is used in `totalAssets()` calculations to ensure the vault's exchange rate remains accurate even while redemptions are in progress.

```solidity
// NestVaultCore.sol
function _processRedeemRequest(uint256 _shares, address _controller, address _owner)
    internal
    returns (uint256 _requestId)
{
    NestVaultCoreStorage storage $ = _getNestVaultCoreStorage();

    // Incremental accounting: Adds new shares to the Controller's existing pending bucket
    uint256 _currentPendingShares = $.pendingRedeem[_controller].shares;
    $.pendingRedeem[_controller] = DataTypes.PendingRedeem(_shares + _currentPendingShares);

    // Global accounting: Increases total shares currently awaiting settlement
    $.totalPendingShares += _shares;

    emit RedeemRequest(_controller, _owner, REQUEST_ID, msg.sender, _shares);
    return REQUEST_ID; // Note: REQUEST_ID is initialized to 0 per ERC-7540 standard
}

```

**Result:** The Owner's shares are now locked in the Vault. The protocol has committed to a future redemption, recorded under the **Controller's** identity, awaiting administrative fulfillment.

---

### 2.3.3 Fulfillment by Admin

The Fulfillment phase is the administrative step where "Pending" obligations are settled. An authorized keeper or administrator (verified via `requiresAuth`) processes the requests by burning the locked shares and earmarking the equivalent underlying assets for the user to claim.

**Step 1: Request Validation**

The function first verifies that the `Controller` has an active redemption request and that the amount being fulfilled does not exceed the pending balance.

```solidity
// NestVaultCore.sol
NestVaultCoreStorage storage $ = _getNestVaultCoreStorage();
DataTypes.PendingRedeem storage _request = $.pendingRedeem[_controller];

// Ensure there is a pending request to fulfill
if (_request.shares == 0 || _shares > _request.shares) {
    revert Errors.ZERO_SHARES();
}

```

**Step 2: Valuation (Share to Asset Conversion)**

The Vault calculates the amount of underlying assets owed based on the current exchange rate provided by the `Accountant`.

```solidity
// NestVaultCore.sol
// _assets is calculated using the Accountant's validated rate
_assets = convertToAssets(_shares);

if (_assets == 0) {
    revert Errors.ZERO_ASSETS();
}

```

**Note on Valuation:** Unlike standard vaults where the rate might be locked at the time of *request*, here it is calculated at the time of *fulfillment* via `convertToAssets(_shares)`. This means the user maintains price exposure to the underlying asset until the Admin executes this transaction.

**Step 3: Execution & Share Burning (`_exit`)**

The Vault reduces the pending state and invokes the `_exit` hook. This logic triggers the `NestShareOFT` contract to burn the shares previously held in escrow. As a result, the equivalent underlying assets are released and become available within the Vault contract.

```solidity
// NestVaultCore.sol
_request.shares -= _shares;
$.totalPendingShares -= _shares;

// The Vault tracks its balance before and after the burn/exit to ensure success
uint256 initialBalance = _asset.balanceOf(address(this));

// MONEY FLOW: Shares are burned; Underlying Assets are released to the Vault
_exit(address(this), _asset, _assets, address(this), _shares);

uint256 finalBalance = _asset.balanceOf(address(this));

```

**Step 4: Integrity Verification**

To protect against external failures or fee-on-transfer issues in the underlying asset, the Vault performs a strict balance check. If the actual amount of assets released is less than the calculated `_assets`, the transaction reverts.

```solidity
// NestVaultCore.sol
uint256 actualTransfer = finalBalance - initialBalance;

if (actualTransfer < _assets) {
    revert Errors.TRANSFER_INSUFFICIENT();
}

```

**Step 5: Transition to "Claimable" State**

Finally, the assets are moved from a global pool to the Controller’s specific `Claimable` bucket. The user (or their Operator) can now see these assets as ready for withdrawal.

```solidity
// NestVaultCore.sol
// The assets (and the record of shares they represent) are now in the Claimable ledger
$.claimableRedeem[_controller] = DataTypes.ClaimableRedeem(
    $.claimableRedeem[_controller].assets + actualTransfer,
    $.claimableRedeem[_controller].shares + _shares
);

```

**Result:** The "Pending" shares have been destroyed. The Vault now holds the physical underlying assets in a "Claimable" state, exclusively reserved for the `Controller`.

---

### 2.3.4 Instant Redeem: The Liquidity Bypass

The **Instant Redeem** feature allows users to bypass the standard asynchronous settlement process (ERC-7540) in exchange for a fee. Unlike the three-step request-fulfillment-withdraw cycle, this function provides immediate settlement in a single transaction.

**Step 1: Entry & Authorization**

The process begins with the same rigorous authorization checks as the asynchronous flow. The caller must be whitelisted globally (`requiresAuth`) and must be either the Owner or an authorized Operator (`_isAuthorizedCaller`).

```solidity
// NestVaultCore.sol
function instantRedeem(uint256 _shares, address _receiver, address _owner)
    public
    requiresAuth
    nonReentrant
    returns (uint256 _postFeeAmount, uint256 _feeAmount)
{
    _isAuthorizedCaller(_owner); // Security: Local permission check

    if (SHARE.balanceOf(_owner) < _shares) revert Errors.INSUFFICIENT_BALANCE();
    if (_shares == 0) revert Errors.ZERO_SHARES();

    // MONEY FLOW: Shares move from Owner to Vault for immediate processing
    SafeTransferLib.safeTransferFrom(ERC20(address(SHARE)), _owner, address(this), _shares);

    (_postFeeAmount, _feeAmount) = _processInstantRedeem(_shares, _receiver);
}

```

**Step 2: Valuation and Fee Application**

The Vault calculates the asset value of the shares using the `Accountant`. Crucially, it then applies a fee percentage (defined in `fees[DataTypes.Fees.InstantRedemption]`) which is deducted from the total asset amount.

```solidity
// NestVaultCore.sol
function _processInstantRedeem(uint256 _shares, address _receiver)
    internal
    returns (uint256 _postFeeAmount, uint256 _feeAmount)
{
    // Calculation: Total Assets - Fee = Post-Fee Amount
    (_postFeeAmount, _feeAmount) = _convertToAssetsForInstantRedeem(_shares);

    ERC20 _asset = ERC20(asset());
    uint256 _initialReceiverBalance = _asset.balanceOf(_receiver);

```

**Step 3: Direct Settlement (`_exit`)**

Unlike the `fulfillRedeem` function—which releases assets to the Vault's own balance—`instantRedeem` directs the `_exit` hook to send the assets **directly to the receiver**. The shares are burned, and the assets never enter the "Claimable" ledger, completing the trade instantly.

```solidity
// NestVaultCore.sol
// MONEY FLOW: Shares burned; Assets sent DIRECTLY to the Receiver
_exit(_receiver, _asset, _postFeeAmount, address(this), _shares);

```

**Step 4: Verification of Transfer**

To ensure the receiver actually received the funds (protecting against failed transfers or fee-on-transfer tokens), the Vault calculates the balance delta of the receiver. If the actual increase in the receiver's balance is less than the calculated `_postFeeAmount`, the transaction reverts.

```solidity
// NestVaultCore.sol   
    uint256 _finalReceiverBalance = _asset.balanceOf(_receiver);
    uint256 _receiverBalanceDelta = _finalReceiverBalance - _initialReceiverBalance;

    if (_receiverBalanceDelta < _postFeeAmount) {
        revert Errors.TRANSFER_INSUFFICIENT();
    }

    emit InstantRedeem(_shares, _postFeeAmount + _feeAmount, _postFeeAmount, _receiver);
}

```

**Result:** The Owner's shares are destroyed, a fee is captured by the protocol (implicitly remaining in the vault or accounted for by the accountant), and the Receiver receives the underlying assets immediately.

---

### 2.3.5 Withdrawal & Redeem: Final Asset Claim

This final phase completes the asynchronous lifecycle. After an administrator has fulfilled a request, the assets are no longer "Pending" but are held in a **Claimable** state. The **Controller** (or their authorized **Operator**) can then extract these assets.

The protocol provides two entry points for this, depending on whether the user wants to specify the amount of assets to receive or the number of claimable shares to burn.

**Step 1: Authorization & Permission**

Both functions require the "Double Lock" to be satisfied:

1. **Global:** The caller must be whitelisted by the protocol's `Authority` (`requiresAuth`).
2. **Local:** The caller must be the **Controller** or an approved **Operator** for that Controller (`_isAuthorizedCaller`).

**Step 2: Internal Ledger Accounting**

Unlike the initial deposit, these functions do not query an external oracle. Instead, they operate on the **internal snapshot** created during the Fulfillment phase. The vault calculates the proportional value of the claimable assets relative to the claimable shares stored in the `claimableRedeem` mapping.

**Option A: Withdrawal (Asset-Based)**
The user requests a specific amount of assets (`_assets`). The vault calculates the corresponding number of shares to "burn" from the claimable ledger.

```solidity
// NestVaultCore.sol -> withdraw()
DataTypes.ClaimableRedeem storage _claimable = $.claimableRedeem[_controller];

// Proportionally calculate shares: (assets * claimable_shares) / claimable_assets
_shares = _assets.mulDivDown(_claimable.shares, _claimable.assets);

// Update Ledger
_claimable.assets -= _assets;
_claimable.shares = _claimable.shares > _shares ? _claimable.shares - _shares : 0;

```

**Option B: Redemption (Share-Based)**
The user requests to burn a specific number of claimable shares (`_shares`). The vault calculates the corresponding amount of assets to release.

```solidity
// NestVaultCore.sol -> redeem()
DataTypes.ClaimableRedeem storage _claimable = $.claimableRedeem[_controller];

// Proportionally calculate assets: (shares * claimable_assets) / claimable_shares
_assets = _shares.mulDivDown(_claimable.assets, _claimable.shares);

// Update Ledger
_claimable.assets = _claimable.assets > _assets ? _claimable.assets - _assets : 0;
_claimable.shares -= _shares;

```

**Step 3: Precision & Rounding Logic**

As noted in the source comments, partial claims introduce precision loss. The protocol utilizes `Math.mulDivDown` for these calculations.

- In **Withdrawal**: Rounding down `_shares` means the user may burn slightly fewer shares than a perfectly precise calculation would require.
- In **Redemption**: Rounding down `_assets` means the user may receive slightly fewer assets than a perfectly precise calculation would require.

The vault includes a safety check in `redeem` to prevent "zero-payout" transactions unless the user is fully clearing their remaining claimable balance.

**Step 4: Asset Transfer**

Once the internal ledger is updated, the vault performs the physical transfer of the underlying assets from the vault contract to the designated `_receiver`.

```solidity
// NestVaultCore.sol
// Physical transfer of the underlying asset (e.g., USDC/WETH)
SafeTransferLib.safeTransfer(ERC20(asset()), _receiver, _assets);

```

**Result:** The Controller’s claimable balance is reduced or cleared, and the physical assets are moved to the receiver’s wallet, successfully concluding the vault lifecycle.

---

### 2.3.6 Update Redeem: Modifying Pending Requests

The **Update Redeem** function provides a mechanism for users to modify their existing asynchronous redemption requests. This is primarily used to decrease the amount of shares currently locked in the "Pending" state, effectively allowing an Owner to "cancel" part of their request and reclaim their shares before administrative fulfillment occurs.

**Step 1: Authorization**

This function enforces the most rigorous authorization checks in the contract. To prevent unauthorized manipulation of a pending position, the caller must satisfy three conditions:

1. **Global:** Caller must be whitelisted by the `Authority` (`requiresAuth`).
2. **Owner-Level:** Caller must be authorized by the `_owner` to manage their shares.
3. **Controller-Level:** Caller must be authorized by the `_controller` to manage the redemption record.

```solidity
// NestVaultCore.sol
function updateRedeem(uint256 _newShares, address _controller, address _owner) external requiresAuth {
    _isAuthorizedCaller(_owner);      // Permission to move the shares
    _isAuthorizedCaller(_controller); // Permission to modify the record
```

**Step 2: Validation of Existing State**

The vault verifies that a redemption request is actually in progress for the specified Controller. Importantly, the current implementation **only allows for the reduction** of a pending request. If a user attempts to increase the number of shares (`_newShares > _oldShares`), the transaction reverts.

```solidity
// // NestVaultCore.sol
uint256 _oldShares = $.pendingRedeem[_controller].shares;

if (_oldShares == 0) revert Errors.NO_PENDING_REDEEM();

// Limitation: Cannot increase the pending amount via this function
if (_oldShares < _newShares) {
    revert Errors.INSUFFICIENT_BALANCE();
}
```

**Step 3: Accounting Adjustment**

The Vault calculates the delta between the old request and the new request. It then updates both the Controller’s specific ledger and the global `totalPendingShares` counter.

```solidity
// NestVaultCore.sol
// Calculate the amount to be returned to the Owner
uint256 _returnAmount = _oldShares - _newShares;

// Global Accounting: Reduce the vault's total pending obligations
$.totalPendingShares -= _returnAmount;

// Ledger Accounting: Update the Controller's specific pending bucket
$.pendingRedeem[_controller].shares = _newShares;
```

**Step 4: Refund**

The "Money Flow" in this step is the inverse of the initial request. The Vault releases the excess shares from its own custody and transfers them back to the Owner’s wallet.

```solidity
// NestVaultCore.sol
// MONEY FLOW: Vault (Escrow) -> Owner (Wallet)
SafeTransferLib.safeTransfer(ERC20(address(SHARE)), _owner, _returnAmount);

emit RedeemUpdated(_controller, _owner, msg.sender, _oldShares, _newShares);
```

**Result:** The pending redemption obligation is reduced, and the Owner regains immediate liquid control over their reclaimed `NestShareOFT` tokens.

---

# 3. Threat Model & Trust Assumptions

## 3.1 Trust Assumptions

- **The Accountant:** We trust the `AccountantWithRateProviders` will always provides the correct price.
- **The Authority:** We trust the Admin to properly whitelist Operators. If the Admin is malicious, they can freeze the entire protocol.

## 3.2 External Dependencies

- **NestShareOFT:** Reliance on LayerZero implementation.

---

# 4. Detailed Findings (The Bug List)

## 4.1 High

---

### **H-01: Rounding Direction in `withdraw` Allows Dust Theft**

[https://github.com/plumenetwork/nest-protocol/blob/main/contracts/NestVaultCore.sol#L630](https://github.com/plumenetwork/nest-protocol/blob/main/contracts/NestVaultCore.sol#L630)

**Description:** 

In the `withdraw` function (claiming fulfilled assets), the vault calculates the amount of "claimable shares" to burn based on the requested assets. The calculation uses `mulDivDown`:

```solidity
_shares = _assets.mulDivDown(_claimable.shares, _claimable.assets);
```

Later it burns the `_shares` using this value:

```solidity
_claimable.shares = _claimable.shares > _shares ? _claimable.shares - _shares : 0;
```

Standard ERC-4626 implementation patterns (and general vault accounting principles) dictate that when a user requests a specific amount of assets, the system should calculate the required shares and **round up**. This ensures the protocol never gives away assets for "free".

However, in the current implementation, by rounding down, a user can withdraw small amounts of assets such that the calculated `_shares` to burn is `0`.

**Recommendation:** 

Change rounding direction to `mulDivUp` for shares burned.

---

## 4.2 Low

### L-01: Code vs. Docs Contradiction in `updateRedeem`

[https://github.com/plumenetwork/nest-protocol/blob/main/contracts/NestVaultCore.sol#L582](https://github.com/plumenetwork/nest-protocol/blob/main/contracts/NestVaultCore.sol#L582)

**Description:** 

The comment for `updateRedeem` claims the function allows users to increase their redemption amount:

> If `_newShares` is higher, additional shares are pulled from the owner
> 

However, the actual code explicitly reverts if the new amount is higher:

```solidity
if (_oldShares < _newShares) {
    revert Errors.INSUFFICIENT_BALANCE();
}
```

**Recommendation:**

Either update the documentation to reflect that this is a "reduce-only" function or update the code to handle the logic of pulling additional shares (`safeTransferFrom`) and increasing `totalPendingShares`.

---

### L-02: `instantRedeem` Fees are not Collected Properly

[https://github.com/plumenetwork/nest-protocol/blob/main/contracts/NestVaultCore.sol#L547](https://github.com/plumenetwork/nest-protocol/blob/main/contracts/NestVaultCore.sol#L547)

**Description:** 

In `instantRedeem`, a fee is calculated and deducted from the payout:

```solidity
(_postFeeAmount, _feeAmount) = _convertToAssetsForInstantRedeem(_shares);
// ...
_exit(_receiver, _asset, _postFeeAmount, address(this), _shares);
```

The `_exit` function burns the full `_shares` amount and sends the `_postFeeAmount` amount of asset to `_receiver`, left `_feeAmount` remaining in the `NestVaultCore`. 

The contract has **no mechanism to sweep or collect these fees** into a treasury, potentially benefiting the remaining share holders by increasing the net asset value per share.

**Recommendation:** 

Clarify if "Fee-to-LP" is intended. If protocol revenue is desired, `instantRedeem` should explicitly transfer `_feeAmount` to a fee collector address.

---

# 5. Notable Factors

- **Centralized Authority Dependency:**
The protocol utilizes a strict `requiresAuth` modifier on all critical user actions (depositing, requesting redemptions, withdrawing). While this enforces KYC/AML compliance, it creates a single point of failure.
- **Accountant Reliance:**
The vault does not calculate share value based on internal reserves but relies on an external `AccountantWithRateProviders`.
- **Asynchronous Settlement Latency:**
Users retain price exposure to the underlying asset during the "Pending" period between a redemption request and its fulfillment.