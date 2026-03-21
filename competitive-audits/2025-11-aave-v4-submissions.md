# S-1 Withdrawal Revert Due to Inconsistent Rounding Between Preview and Execution

## Summary
Withdrawals can revert for users even when attempting to withdraw the maximum amount reported by the protocol. This is caused by inconsistent rounding directions between asset-to-share and share-to-asset conversions across the supply and withdraw paths.

Specifically, `previewRemoveByShares()` rounds the withdrawable asset amount down, while the actual `remove()` operation rounds the required shares up. Under realistic states (interest accrual, virtual assets/shares, non-integer ratios), this mismatch can cause the withdraw operation to require more shares than the user owns, resulting in a revert.

This issue can lead to stuck user funds and violates the expected reversibility between supply and withdraw operations.

## Affected Functions

```solidity
Spoke.supply(uint256 reserveId, uint256 amount, address onBehalfOf)
Spoke.withdraw(uint256 reserveId, uint256 amount, address onBehalfOf)

Hub.add(uint256 assetId, uint256 amount)
Hub.previewRemoveByShares(uint256 assetId, uint256 shares)
Hub.remove(uint256 assetId, uint256 amount, address to)
```

Link - https://github.com/sherlock-audit/2025-11-aave-v4/blob/main/aave__aave-v4/src/spoke/Spoke.sol#L248C1-L277C1

## Root Cause

The root cause is a rounding asymmetry between preview and execution paths when converting between shares and assets.

The protocol uses:

- rounding down when converting shares → assets in `previewRemoveByShares`
- rounding up when converting assets → shares in `remove`

These two operations are not mathematical inverses.

The protocol implicitly assumes that:

```solidity
ceil( floor(x * A / S) * S / A ) ≤ x
```

Where:

- x = user shares
- A = total assets (including virtual assets)
- S = total shares (including virtual shares)

This assumption is false when:

- A / S is non-integer (common after interest accrual)
- virtual assets and shares are present
- rounding down is followed by rounding up

As a result, the asset amount considered safe during preview may later require more shares than the user owns during execution.

## Internal Pre-conditions

- Non-zero interest accrued (asset/share ratio drifted)
- Virtual assets/shares enabled (always true)
- User attempts to withdraw their full position or the maximum previewed amount

## External Pre-conditions

none

## Attack Path

#### Supply Path (rounds down)

In `Spoke.supply`:

```solidity
uint256 suppliedShares = reserve.hub.add(reserve.assetId, amount);
userPosition.suppliedShares += suppliedShares.toUint120();
```

In `Hub.add`:

```solidity
suppliedShares = asset.toAddedSharesDown(amount);
```

Which expands to:

```solidity
suppliedShares =
  floor(
    amount * (totalAddedShares + VIRTUAL_SHARES)
    / (totalAddedAssets + VIRTUAL_ASSETS)
  )
```

Effect: users receive ≤ exact fair shares (rounding down).

#### Withdraw Path (rounds down, then up)

In `Spoke.withdraw`:

```solidity
uint256 withdrawnAmount = MathUtils.min(
  amount,
  hub.previewRemoveByShares(assetId, userPosition.suppliedShares)
);

uint256 withdrawnShares = hub.remove(assetId, withdrawnAmount, msg.sender);

userPosition.suppliedShares -= withdrawnShares.toUint120();
```

In `Hub.previewRemoveByShares`:

```solidity
return shares.toAssetsDown(asset.totalAddedAssets(), asset.addedShares);
```

Which expands to:

```solidity
withdrawnAmount =
  floor(
    userShares * (totalAddedAssets + VIRTUAL_ASSETS)
    / (totalAddedShares + VIRTUAL_SHARES)
  )
```

In `Hub.remove`:

```solidity
uint120 shares = asset.toAddedSharesUp(amount);
```

Which expands to:

```solidity
withdrawnShares =
  ceil(
    withdrawnAmount * (totalAddedShares + VIRTUAL_SHARES)
    / (totalAddedAssets + VIRTUAL_ASSETS)
  )
```

### Steps

1. User supplies assets → receives shares rounded down  
2. Time passes → interest accrues, ratios drift  
3. User calls `withdraw()`  
4. `previewRemoveByShares()` returns a conservative asset amount (rounded down)  
5. `remove()` converts that amount back to shares (rounded up)  
6. Required shares exceed `userPosition.suppliedShares`  
7. Subtraction underflows → transaction reverts  

### Example

Assume a 6-decimal asset (e.g., USDC):

```solidity
totalAddedAssets  = 1_000_000_000  (1,000 USDC)
totalAddedShares  =   998_001_000
VIRTUAL_ASSETS    = 1_000_000
VIRTUAL_SHARES    = 1_000_000

A = 1_001_000_000
S =   999_001_000

userShares = 1_000_000
```

##### Preview

```solidity
withdrawnAmount =
  floor(1_000_000 * 1_001_000_000 / 999_001_000)
= 1_002_000
```

#### Execution

```solidity
withdrawnShares =
  ceil(1_002_000 * 999_001_000 / 1_001_000_000)
= 1_001_000
```

#### Result

```solidity
userShares      = 1_000_000
withdrawnShares = 1_001_000

→ subtraction underflows
→ withdraw reverts
```

## Impact

Withdrawals can revert deterministically even when users attempt to withdraw the maximum amount returned by the protocol’s preview function.

users may be unable to fully withdraw their position, violating the expected guarantee that previewed withdrawals are always executable. This creates a user-facing denial of withdrawal under normal protocol conditions (interest accrual and virtual balances).

## PoC

No response

## Mitigation

### Mitigation 1 — Use Preview-Consistent Share Calculation (Recommended)

Ensure the same rounding direction is used for preview and execution:

```solidity
uint256 withdrawnShares =
  hub.previewRemoveByAssets(assetId, withdrawnAmount);
```

This guarantees:

```solidity
withdrawnShares ≤ userPosition.suppliedShares
```

### Mitigation 2 — Clamp Shares to User Balance

Defensively cap the shares burned:

```solidity
withdrawnShares = MathUtils.min(
  withdrawnShares,
  userPosition.suppliedShares
);
```

This preserves safety even if rounding mismatches remain.
