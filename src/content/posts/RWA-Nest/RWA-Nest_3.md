---
title: Nest (RWA Earn) - Part 3
published: 2026-02-11
pinned: false
description: Key Execution Flows
tags: [Smart Contract,RWA]
category: RWA
draft: false
---
## Scenario A: Permissioned Deposit

**Context:** Users cannot deposit directly into the Vault. They must pass through the `NestVaultPredicateProxy` which validates an off-chain signature (proof of KYC).

**Key Steps:**

1. **Verification:** Proxy verifies the signature against the set Policy.
2. **Asset Movement:** Assets are pulled User → Proxy → Vault → Share Contract.
3. **Pricing:** The Vault queries the `Accountant` for the current rate.
4. **Minting:** The Vault instructs the Share contract to mint tokens to the user.

![diagram.jpg](Nest(RWA%20Earn)-3-Key%20Execution%20Flows/diagram.jpg)

**Function Signature:**

```solidity
function deposit(
    ERC20 _depositAsset,      // Must be whitelisted in vault
    uint256 _depositAmount,    // Must be > 0
    address _recipient,        // Shares mint destination
    NestVault _vault,         // Target vault address
    PredicateMessage calldata _predicateMessage  // From off-chain service
) external requiresAuth nonReentrant whenNotPaused 
returns (uint256 _shares)

// or
function mint(
		ERC20 _depositAsset,
    uint256 _shares,
    address _recipient,
    NestVault _vault,
    PredicateMessage calldata _predicateMessage
) external nonReentrant whenNotPaused 
returns (uint256 _depositAmount)
```

**Event Monitoring:**

```solidity
event Deposit(
    address _recipient,
    address _depositAsset,
    uint256 _depositAmount,
    uint256 _shares,
    uint256 block.timestamp,
    address _vault
);
```

### Notes

**Authorization Flow:**

- Users cannot directly call the Vault - they must go through the `NestVaultPredicateProxy`
- The proxy validates off-chain KYC/compliance proofs via Predicate's policy engine
- Invalid signatures or failed policy checks will revert the entire transaction

**Asset Approval Requirements:**

```solidity
// Users must approve the PROXY, not the Vault
await asset.approve(proxyAddress, depositAmount);

// Common mistake - this won't work:
await asset.approve(vaultAddress, depositAmount); // ❌ Wrong
```

**Asset Custody:**

Final asset location is the **NestShareOFT contract**, not the Vault:

```
User → Proxy → Vault → Share Contract
```

**Cross-Chain Deposits:**

For bridged or cross-chain deposits, use the overloaded `deposit()` with a `_depositor` parameter:

```solidity
// Allows verification of non-EVM addresses
proxy.deposit(
    asset,
    amount,
    receiver,
    vault,
    bytes32(depositorsAddress), // Other address in bytes32
    predicateMessage
);
```

---

## Scenario B: Asynchronous Redemption

**Context:** RWAs are often illiquid. The protocol uses the **ERC-7540** standard: Request → Pending → Claim.

**Key Steps:**

1. **Request:** User calls `requestRedeem`. Shares are transferred to the Vault (Escrow) and tracked in `pendingRedeem`.
2. **Liquidation (Off-chain):** The admin liquidates real-world assets to ensure the Share contract has enough ERC20s.
3. **Fulfillment:** Admin calls `fulfillRedeem`. The Vault burns the locked shares and moves assets to a `claimable` state.
4. **Claim:** User calls `withdraw` to retrieve assets.

![diagram.jpg](Nest(RWA%20Earn)-3-Key%20Execution%20Flows/diagram%201.jpg)

**Function Signature：**

```solidity
function requestRedeem(
    uint256 _shares,      // Amount to redeem
    address _controller,  // Beneficiary of redemption
    address _owner        // Share owner (msg.sender or approved operator)
) external returns (uint256 _requestId)
```

```solidity
function fulfillRedeem(
		address _controller, 
		uint256 _shares
) public requiresAuth nonReentrant returns (uint256 _assets)
```

```solidity
function withdraw(
    uint256 _assets,      // Amount of assets to claim
    address _receiver,    // Asset recipient
    address _controller   // Redemption controller
) external returns (uint256 _shares)

// or using
function redeem(
		uint256 _shares, // Amount of shares to redeem
		address _receiver, 
		address _controller
) public override requiresAuth returns (uint256 _assets)
```

**Event Monitoring:**

```solidity
event RedeemRequest(
    address _controller,
    address _owner,
    uint256 REQUEST_ID,  // Always 0 in current implementation
    address msg.sender,
    uint256 _shares
);
```

```solidity
event RedeemFulfilled(
    address _controller,
    uint256 _shares,
    uint256 _assets,        // Expected assets
    uint256 actualTransfer   // Actually received
);
```

```solidity
event Withdraw(
		address msg.sender, 
		address _receiver, 
		address _controller,
		uint256 _assets, 
		uint256 _shares
);
```

### **Notes**

**Share Approval:**

Shares must be approved to the **Vault**, not the proxy:

```solidity
await share.approve(vaultAddress, shareAmount); // ✅ Correct
await share.approve(proxyAddress, shareAmount); // ❌ Wrong
```

**Controller vs Owner:**

- `_controller`: Who receives the assets (can differ from owner)
- `_owner`: Who owns the shares being redeemed
- If `msg.sender != _owner`, then msg.sender must be an approved operator of `_owner`

```solidity
// Scenario 1: Self-redemption (most common)
vault.requestRedeem(shares, userAddress, userAddress);

// Scenario 2: Operator-managed redemption
vault.requestRedeem(shares, beneficiaryAddress, userAddress);

// Scenario 3: Protocol integration
vault.requestRedeem(shares, defiProtocolAddress, userAddress);
```

**Request Aggregation:**

Multiple requests to same controller are **aggregated**

```jsx
// First request for owner_1
await vault.requestRedeem(100e6, controller, owner_1);
// pendingRedeem[controller] = 100e6

// Second request for owner_2
await vault.requestRedeem(50e6, controller, owner_2);
// pendingRedeem[controller] = 150e6 (cumulative!)
```

Therefore, if the user uses a DeFi protocol as the controller to receive the asset, the controller should implement a counting mechanism to record the amount owed to each user. For example:

```solidity
contract YieldAggregator {
    mapping(address => uint256) public userAssetClaims;

    function onRedeemFulfilled(uint256 assets) external {
        // Protocol needs to track which users are owed what
        userAssetClaims[user1] += portion1;
        userAssetClaims[user2] += portion2;
    }
}
```

**No Deadline:**

- Requests never expire - they remain pending indefinitely
- No built-in timeout mechanism
- Users can call `updateRedeem()` to reduce pending amount, **but not increase**:

```solidity
// Current pending: 100e6 shares

// ✅ Can reduce
vault.updateRedeem(50e6, controller, owner);
// Excess 50e6 shares returned to owner

// ❌ Cannot increase
vault.updateRedeem(150e6, controller, owner);
// Will revert with INSUFFICIENT_BALANCE
```

**Floating Exchange Rate:**

The exchange rate is determined **at fulfillment time**, not request time:

```solidity
// Day 1: Request 100 shares @ rate 1.05 = 105 USDC expected
vault.requestRedeem(100e6, controller, owner);

// Day 30: Rate drops to 1.00
// Day 30: Admin fulfills @ rate 1.00 = 100 USDC actual
admin.fulfillRedeem(controller, 100e6);

// User receives 100 USDC, not 105 USDC
```

**Partial Fulfillment:**

Admins can fulfill requests partially:

```solidity
// User requested 1000 shares
vault.requestRedeem(1000e6, controller, owner);

// Admin fulfills only 300 shares
admin.fulfillRedeem(controller, 300e6);
// Remaining 700 shares still pending

// Later, fulfill the rest
admin.fulfillRedeem(controller, 700e6);
```

---

## Scenario C: Instant Redemption

**Context:** If the Share contract holds a liquid buffer, users can bypass the wait time by paying a fee.

**Key Steps:**

1. **Calculation:** Vault calculates Gross Asset Value via Accountant, then subtracts the `InstantRedemption` fee.
2. **Execution:** Vault instructs Share contract to burn user shares and send *Net Assets* to the user.
3. **Fee:** The fee portion remains in the Share contract (accreting value to remaining holders).

![diagram.jpg](Nest(RWA%20Earn)-3-Key%20Execution%20Flows/diagram%202.jpg)

**Function Signature:**

```solidity
function instantRedeem(
    uint256 _shares,      // Shares to redeem
    address _receiver,    // Asset recipient
    address _owner        // Share owner
) external returns (
    uint256 _postFeeAmount,  // Net assets received
    uint256 _feeAmount       // Fee charged
)
```

```solidity
function previewInstantRedeem(uint256 _shares)
    public view
    returns (uint256 _postFeeAmount, uint256 _feeAmount)
```

**Event Monitoring:**

```solidity
event InstantRedeem(
    uint256 shares,
    uint256 assets,         // Gross assets (before fee)
    uint256 postFeeAmount,  // Net assets (after fee)
    address receiver
);
```

### Notes

**No Slippage Protection:**

The function has **no built-in slippage protection** (`minAssetsOut` parameter):

```solidity
// Current implementation - no min amount
vault.instantRedeem(shares, receiver, owner);

// Recommended pattern - check first
(uint256 expectedAssets, uint256 fee) = vault.previewInstantRedeem(shares);
require(expectedAssets >= minAcceptable, "Slippage too high");
vault.instantRedeem(shares, receiver, owner);
```

**Fee Mechanics:**

```solidity
// Fee stays in Share contract, accreting value to remaining holders
GrossAssets = shares * exchangeRate
FeeAmount = GrossAssets * InstantRedemptionFee
NetAmount = GrossAssets - FeeAmount

// Example: 1000 shares @ rate 1.05, 0.5% fee
// Gross = 1050 USDC
// Fee = 5.25 USDC (stays in Share contract)
// Net = 1044.75 USDC (sent to user)
```

**Fee Destination:**

Unlike management fees that are claimed by the protocol, instant redemption fees remain in the Share contract permanently, increasing the value for remaining shareholders (similar to a buyback).

**Authorization:**

- Must be owner or approved operator
- Shares must be approved to the Vault

```solidity
// As owner
share.approve(vault, shares);
vault.instantRedeem(shares, receiver, msg.sender);

// As operator
share.approve(vault, shares); // Owner approves
vault.setOperator(operator, true); // Owner authorizes operator
vault.instantRedeem(shares, receiver, owner); // Operator calls
```

**Liquidity Requirements:**

Since the manager of `NestShareOFT` is able to transfer all the funds, user should check the balance of the the share contract when using instant redemption:

```solidity
// Check before attempting
uint256 shareBalance = asset.balanceOf(address(share));
(uint256 netAssets, ) = vault.previewInstantRedeem(shares);

if (shareBa[lance < netAssets) {
    // Use async redemption instead
    vault.requestRedeem(shares, controller, owner);
}
```

**Fee Configuration:**

```solidity
// View current instant redemption fee
uint32 currentFee = vault.fees(DataTypes.Fees.InstantRedemption);
// e.g., 5000 = 0.5% (5000 / 1_000_000)

// Admin can update fee (up to maxFees cap)
vault.setFee(DataTypes.Fees.InstantRedemption, 10000); // 1%
```

---

## Scenario D: Exchange Rate Updates & Circuit Breakers

**Context:** The `NestAccountant` is responsible for updating the share price and accruing management fees. It includes a safety mechanism that pauses the system if certain undesirable conditions are met.

**Key Steps:**

1. **Validation:** New rate is checked against `Upper` and `Lower` bounds relative to the old rate.
2. **Fee Accrual:** Management fees are calculated based on time elapsed and added to `feesOwedInBase`.
3. **Circuit Breaker:** If the rate is invalid, the Accountant **Pauses**. This halts all deposits and redemptions until an Admin unpauses.

![diagram.jpg](Nest(RWA%20Earn)-3-Key%20Execution%20Flows/diagram%203.jpg)

**Function Signature:**

```solidity
function updateExchangeRate(uint96 _newExchangeRate) external requiresAuth
```

**Event Monitoring:**

```solidity
// Success case
event ExchangeRateUpdated(
    uint96 _currentExchangeRate,
    uint96 _newExchangeRate,
    uint64 currentTime
);

// Failure case (pause)
event ExchangeRateUpdatePaused(
    uint96 _currentExchangeRate,
    uint96 _newExchangeRate,
    uint64 currentTime
);

// Related events
event Paused();
event Unpaused();
```

### Notes

**Validation Parameters:**

```solidity
struct AccountantState {
    address payoutAddress;
    uint128 feesOwedInBase;
    uint128 totalSharesLastUpdate;
    uint96 exchangeRate;
    uint32 allowedExchangeRateChangeUpper;  // e.g., 1.05e6 = 5% increase max
    uint32 allowedExchangeRateChangeLower;  // e.g., 0.95e6 = 5% decrease max
    uint64 lastUpdateTimestamp;
    bool isPaused;
    uint32 minimumUpdateDelayInSeconds;     // e.g., 3600 = 1 hour min
    uint32 managementFee;                   // e.g., 0.02e6 = 2% annual
}
```

**Fee Accrual Timing:**

Management fees are **only** calculated and accumulated when `updateExchangeRate()` is called - they do not accrue automatically or continuously:

```solidity
// Fees accrue based on time elapsed since LAST update
timeDelta = currentTime - lastUpdateTimestamp;

// Annual management fee applied proportionally
managementFeesAnnual = assetValue * managementFee / DENOMINATOR;
newFeesOwed = managementFeesAnnual * timeDelta / ONE_YEAR;

// Added to accumulated fees
feesOwedInBase += newFeesOwed;
```

- If `updateExchangeRate()` is not called for a long period, no fees accrue during that period until the next update.
- During a pause, fees continue to accrue for the time period up to the pause event:
    
    ```solidity
    if (_invalid) {
           // Calculate fees for time before pause
           timeDelta = currentTime - lastUpdateTimestamp;
           newFeesOwed = calculateFees(timeDelta);
           feesOwedInBase += newFeesOwed;
           
           isPaused = true;  // Then pause
       }
    ```
    

**Circuit Breaker Conditions:**

The system pauses if **any** of these three conditions are met:

```solidity
bool _invalid =
    // Condition 1: Update too soon (unless already paused)
    (!_state.isPaused &&
     _currentTime < _state.lastUpdateTimestamp + _state.minimumUpdateDelayInSeconds)

    // Condition 2: Rate increased too much
    || _newExchangeRate >
       _currentExchangeRate.mulDivDown(_state.allowedExchangeRateChangeUpper, DENOMINATOR)

    // Condition 3: Rate decreased too much
    || _newExchangeRate <
       _currentExchangeRate.mulDivDown(_state.allowedExchangeRateChangeLower, DENOMINATOR);
```

**Recovery from Pause:**

```solidity
// 1. Admin investigates the cause
// 2. Admin unpauses the system
accountant.unpause();

// 3. When paused, next update can happen immediately
// (minimumDelay check is skipped if isPaused == true)
accountant.updateExchangeRate(correctedRate);
```

**Impact on User Operations:**

When Accountant is paused, the following operations **fail**:

```solidity
// ❌ All rate queries revert
vault.deposit(...);           // Calls getRateInQuoteSafe()
vault.instantRedeem(...);     // Calls getRateInQuoteSafe()
vault.fulfillRedeem(...);     // Calls getRateInQuoteSafe()

// ✅ These still work (no rate needed)
vault.requestRedeem(...);     // Only locks shares
vault.withdraw(...);          // Uses pre-calculated claimable
vault.redeem(...);            // Uses pre-calculated claimable
```

**Admin Paramater Updates:**

All circuit breaker parameters can be updated:

```solidity
// Adjust bounds
accountant.updateUpper(1.10e6);  // Allow 10% increase
accountant.updateLower(0.90e6);  // Allow 10% decrease

// Adjust timing
accountant.updateDelay(7200);    // 2 hour minimum

// Adjust fees
accountant.updateManagementFee(0.015e6);  // 1.5% annual
```

---

## Scenario E: Cross-Chain Transfer for OFT

**Context:** Moving shares from ain A to Chain B using LayerZero.

**Key Steps:**

1. **Source:** `NestVaultOFT` burns shares on the source chain. **Assets do not move.**
2. **Message:** LayerZero relays the message.
3. **Destination:** `NestVaultOFT` mints shares on the destination chain.

![diagram.jpg](Nest(RWA%20Earn)-3-Key%20Execution%20Flows/diagram%204.jpg)

**Function Signature:**

```solidity
function send(
    SendParam calldata _sendParam,
    MessagingFee calldata _fee,
    address _refundAddress
) external payable returns (
    MessagingReceipt memory msgReceipt,
    OFTReceipt memory oftReceipt
)
```

**Event Monitoring:**

```solidity
// Source chain
event OFTSent(
    bytes32 indexed guid,    // Unique message ID
    uint32 dstEid,           // Track destination
    address indexed from,
    uint256 amountSentLD,
    uint256 amountReceivedLD
);

// Destination chain
event OFTReceived(
    bytes32 indexed guid,    // Same ID as OFTSent
    uint32 srcEid,           // Track source
    address indexed to,
    uint256 amountReceivedLD
);

// Use guid to match send/receive pairs
// Track via LayerZero Scan: layerzeroscan.com
```

### Notes

**SendParam Structure:**

```solidity
struct SendParam {
    uint32 dstEid;              // Destination endpoint ID (e.g., 30101 for Ethereum)
    bytes32 to;                 // Recipient address (bytes32 format)
    uint256 amountLD;           // Amount in local decimals
    uint256 minAmountLD;        // Minimum amount after slippage
    bytes extraOptions;         // Gas settings
    bytes composeMsg;           // Additional message data
    bytes oftCmd;               // OFT-specific commands
}
```

**MessagingFee Structure:**

```solidity
struct MessagingFee {
    uint256 nativeFee;          // Gas payment in native token (ETH, MATIC, etc.)
    uint256 lzTokenFee;         // Optional ZRO token payment
}
```

**Critical Differences from ERC20:**

```solidity
// ❌ Traditional ERC20 - assets move
bridge.transfer(recipient, amount);
// Assets locked on source, minted on destination

// ✅ OFT - shares move, assets stay
vault.send(sendParam, fee, refundAddress);
// Shares burned on source, minted on destination
// ASSETS REMAIN WHERE THEY ARE
```

For example:

```solidity
// Before transfer
Source Chain:
  - Share Contract: 1000 USDC (backing all shares)
  - User: 100 shares
Destination Chain:
  - Share Contract: 0 USDC
  - User: 0 shares

// After sending 100 shares from Source → Destination
Source Chain:
  - Share Contract: 1000 USDC (unchanged!)
  - User: 0 shares (burned)
Destination Chain:
  - Share Contract: 0 USDC (unchanged!)
  - User: 100 shares (minted)

// Total shares across all chains: same as before
// Total assets: remain on original chain
```

**Exchange Rate Synchronization:**

All chains share the **same exchange rate** from the Accountant:

```solidity
// Ethereum: exchangeRate = 1.05
// User holds 100 shares = 105 USDC value

// After bridging to Arbitrum
// Arbitrum: exchangeRate = 1.05 (same)
// User holds 100 shares = 105 USDC value (same)

// Rate updates affect all chains simultaneously
```

**Slippage Handling:**

OFT has no slippage for share transfers:

```solidity
sendParam.amountLD = 100e6;       // Send 100 shares
sendParam.minAmountLD = 100e6;    // Receive 100 shares

// Amount burned on source = Amount minted on destination
// Always 1:1 (except tiny dust from DVN fees)
```

**Complete Send Example:**

```solidity
// 1. Prepare send parameters
SendParam memory sendParam = SendParam({
    dstEid: 30110,                    // Arbitrum
    to: addressToBytes32(recipient),  // Convert address to bytes32
    amountLD: 100e6,                  // 100 shares
    minAmountLD: 100e6,               // Expect 100 shares
    extraOptions: optionsBytes,       // Gas settings
    composeMsg: "",                   // No compose
    oftCmd: ""                        // No OFT commands
});

// 2. Quote the fee
MessagingFee memory fee = vault.quoteSend(sendParam, false);

// 3. Send with native token
vault.send{value: fee.nativeFee}(
    sendParam,
    fee,
    msg.sender  // Refund address for excess fees
);
```

---

## Scenario F: Management Fee Claiming

**Context:** Management fees accrue in `feesOwedInBase` during each exchange rate update. The protocol can claim these fees in any supported asset through the `NestAccountant.claimFees()` function, which must be called by the Share contract.

**Key Steps:**

1. **Trigger:** Share contract calls `claimFees()` with desired fee asset.
2. **Conversion:** Accountant converts `feesOwedInBase` to the target asset using rate providers.
3. **Transfer:** Fees are transferred from Share contract to `payoutAddress`.
4. **Remainder:** Any precision loss is carried forward as remainder dust.

![diagram.jpg](Nest(RWA%20Earn)-3-Key%20Execution%20Flows/diagram%205.jpg)

**Function Signature:**

```solidity
function claimFees(ERC20 _feeAsset) external
```

**Event Monitoring:**

```solidity
event FeesClaimed(
    address indexed feeAsset,  // Asset in which fees were claimed
    uint256 amount              // Amount of fees transferred
);
```

### Notes

**Caller Restriction:**

Only `NestShareOFT` can call `claimFees()`:

```solidity
function claimFees(ERC20 _feeAsset) external {
    if (msg.sender != SHARE) {
        revert OnlyCallableByNestShare();
    }
    // ...
}

// Triggered via Share's manage() function
share.manage(
    address(accountant),
    abi.encodeWithSignature("claimFees(address)", feeAsset),
    0  // no ETH value
);
```

**Fee Asset Requirements:**

The fee asset must have a configured rate provider:

```solidity
// Before claiming in non-base assets, configure rate provider
accountant.setRateProviderData(
    WETH,                // Fee asset
    false,               // Not pegged to base
    wethRateProvider     // Oracle address
);

// For pegged assets (USDT, USDC, DAI if base is USDC)
accountant.setRateProviderData(
    USDT,
    true,                // Pegged to base
    address(0)           // No oracle needed
);
```

**Fee Conversion Logic:**

```solidity
// Example 1: Claim in base asset (USDC) - no conversion
accountant.claimFees(USDC);
// feesOwedInBase = 1000000000 (1000 USDC, 6 decimals)
// feesInFeeAsset = 1000000000 (no conversion)
// Gas cost: ~50k

// Example 2: Claim in pegged asset (USDT) - simple conversion
accountant.claimFees(USDT);
// feesOwedInBase = 1000000000 (1000 USDC, 6 decimals)
// USDT also 6 decimals, pegged 1:1
// feesInFeeAsset = 1000000000
// Gas cost: ~55k

// Example 3: Claim in different asset (WETH) - oracle conversion
accountant.claimFees(WETH);
// feesOwedInBase = 1000000000 (1000 USDC, 6 decimals)
// Step 1: Convert decimals 6 → 18: 1000000000000000000000
// Step 2: Get rate: 3000e18 (1 WETH = 3000 USDC)
// Step 3: Calculate: 1000e18 * 1e18 / 3000e18 = 333333333333333333
// feesInFeeAsset = 0.333... WETH
// Gas cost: ~75k (includes oracle call)
```

**Precision Handling:**

```solidity
// Precision loss example
feesOwedInBase = 1000000001 (1000.000001 USDC, 6 decimals)
feeAsset = WETH (18 decimals)
rate = 3000e18

// Step 1: Decimal conversion
1000000001 → 1000000001000000000000 (18 decimals)

// Step 2: Rate conversion
1000000001000000000000 * 1e18 / 3000e18
= 333333333666666666 (18 decimals)

// Step 3: Remainder tracking
remainder = 1 (in base 6 decimals)
// This 1 wei USDC is carried forward to next claim

// State update
feesOwedInBase = 1 (carried forward)
```

**Remainder Accumulation:**

```solidity
// Claim 1
feesOwedInBase = 1000000001
Claimed = 1000000000 in WETH
Remainder = 1

// Claim 2 (later)
feesOwedInBase = 500000000 + 1 (previous remainder)
Claimed = 500000000 in WETH
Remainder = 1 (again)

// Remainder keeps accumulating until significant
```

**Payout Address:**

```solidity
// Set during initialization
initialize(
    totalShares,
    payoutAddress,  // Treasury multisig
    startingRate,
    // ... other params
);

// Update later if needed
accountant.updatePayoutAddress(newTreasuryAddress);

// Event emitted
event PayoutAddressUpdated(
    address oldPayout,  // Old treasury
    address newPayout   // New treasury
);
```

**Fee Claiming Strategies:**

```solidity
// Strategy 1: Claim frequently in base asset
share.manage(
    address(accountant),
    abi.encodeWithSignature("claimFees(address)", baseAsset),
    0
);

// Strategy 2: Accumulate and claim in appreciated asset
share.manage(
    address(accountant),
    abi.encodeWithSignature("claimFees(address)", WETH),
    0
);
```

**Prerequisites Checklist:**

```solidity
// ✅ 1. Rate Provider Configured
accountant.setRateProviderData(feeAsset, isPegged, rateProvider);

// ✅ 2. System Not Paused
require(!accountant.isPaused(), "System paused");

// ✅ 3. Fees Accumulated
require(accountant.feesOwedInBase() > 0, "No fees to claim");

// ✅ 4. Share Contract Has Assets
uint256 balance = feeAsset.balanceOf(address(share));
(uint256 feesInFeeAsset,) = accountant.previewClaimFees(feeAsset);
require(balance >= feesInFeeAsset, "Insufficient balance");

// ✅ 5. Caller is Authorized
// Only Share contract or authorized manager
```