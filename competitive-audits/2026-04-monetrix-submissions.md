# S1 - Stale Surplus Validation Between Settlement and Distribution Enables Yield Distribution Based on Outdated Accounting

## Description

The protocol separates yield processing into two distinct phases:

- `settle(proposedYield)`
- `distributeYield()`

https://github.com/code-423n4/2026-04-monetrix/blob/3d94be1361ca01d959f9165a78f0d75c5657fe3e/src/core/MonetrixVault.sol#L364

https://github.com/code-423n4/2026-04-monetrix/blob/3d94be1361ca01d959f9165a78f0d75c5657fe3e/src/core/MonetrixVault.sol#L377

During `settle`, the protocol validates that the proposed yield is bounded by the current surplus via:

```solidity
IMonetrixAccountant(accountant).settleDailyPnL(proposedYield);
```

This ensures:

```text
proposedYield ≤ distributableSurplus (at time T1)
```

However, during `distributeYield`, the protocol does not re-validate surplus or backing conditions, and instead proceeds based on previously settled values and escrowed funds.

This introduces a time-of-check to time-of-use inconsistency:

```text
surplus(T1) is validated
surplus(T2) is assumed unchanged
```

If the system’s backing changes between `settle` and `distributeYield`, the protocol may distribute yield that is no longer supported by current backing.

## Root Cause

The issue arises from:

1. Split-phase accounting design

- `settle()` performs validation
- `distributeYield()` performs execution

2. Missing revalidation

`distributeYield()` does not call:

```solidity
accountant.distributableSurplus()
```

or any equivalent safety check.

3. External state dependency

`totalBackingSigned()` depends on:

- HyperCore precompile reads
- market conditions
- oracle values

These can change between transactions.

4. Lack of atomicity

No guarantee that:

```text
surplus(T1) == surplus(T2)
```

## Impact

This issue allows the protocol to violate its core economic invariant:

```text
Distributed yield ≤ current backing surplus
```

Specifically:

Yield may be distributed even when:

```text
totalBackingSigned < totalSupply
```

This can result in:

- undercollateralized USDM supply
- mispriced sUSDM exchange rate
- incorrect yield attribution

## Attack Path

T1

- surplus is positive
- `settle(proposedYield)` succeeds

T2 (later)

- backing decreases (e.g., L1 losses, oracle update)
- surplus becomes smaller or negative

T3

`distributeYield()` executes WITHOUT revalidation:

```text
-> yield is distributed based on stale state
-> accounting invariant is violated
```

## Recommended Mitigation

1. Revalidate surplus during distribution

Add check inside `distributeYield()`:

```solidity
require(
    accountant.distributableSurplus() >= pendingYield,
    "stale or insufficient surplus"
);
```

2. Atomic settlement-distribution

Combine into single operation:

```solidity
function settleAndDistribute(uint256 proposedYield)
```

3. Snapshot-based accounting

Store:

```solidity
uint256 settlementSnapshotBacking;
```

and validate against it during distribution.

4. Expiry window

Add time bound:

```solidity
require(block.timestamp - lastSettlementTime < MAX_DELAY);
```

## Proof of Concept

```solidity
function test_submissionValidity() public {
    address actor = user1;

    _deposit(actor, 1_000_000e6);

    _mockVaultL1SpotUsdc(1_000_000e8);

    vm.warp(block.timestamp + 1 days);

    uint256 safeYield = 200e6;

    vm.prank(operator);
    vault.settle(safeYield);

    _mockVaultL1SpotUsdc(0);

    vm.prank(operator);
    vault.distributeYield();

    int256 backing = accountant.totalBackingSigned();
    uint256 supply = usdm.totalSupply();

    assertLt(backing, int256(supply));
}
```

---

# S2 - Unbounded Per-User Request Arrays Lead to Permanent Fund Lock Due to Non-Scalable Claim Execution

## Description

The protocol maintains per-user request tracking using dynamically growing arrays:

- `_userUnstakeIds` in `sUSDM.sol`
- `_userRedeemIds` in `MonetrixVault.sol`

Each new user action (e.g., `cooldownShares`, `cooldownAssets`, `requestRedeem`) appends a new request ID to these arrays.

When a user later attempts to finalize their request (e.g., `claimUnstake` or `claimRedeem`), the protocol removes the corresponding request ID using a linear scan.

In `MonetrixVault.sol : 559`

```solidity
function _removeUserRedeemId(address user, uint256 requestId) private {
    uint256[] storage ids = _userRedeemIds[user];
    uint256 len = ids.length;

    for (uint256 i = 0; i < len; i++) {
        if (ids[i] == requestId) {
            ids[i] = ids[len - 1];
            ids.pop();
            return;
        }
    }
}
```

In `sUSDM.sol : 317`

```solidity
function _removeUserUnstakeId(address user, uint256 requestId) private {
    uint256[] storage ids = _userUnstakeIds[user];
    uint256 len = ids.length;

    for (uint256 i = 0; i < len; i++) {
        if (ids[i] == requestId) {
            ids[i] = ids[len - 1];
            ids.pop();
            return;
        }
    }
}
```

This results in `O(n)` complexity for every claim operation, where `n` is the number of requests created by the user.

Because the protocol does not enforce any upper bound on the number of requests a user can create, a user can accumulate a sufficiently large number of entries such that any subsequent claim operation exceeds the block gas limit and reverts.

This leads to a scenario where the protocol can no longer execute the required state transition to release user funds, rendering them permanently inaccessible.

## Root Cause

The issue arises from the interaction of the following design decisions:

1. Unbounded state growth

The protocol allows unlimited request creation per user.

Each request appends to an ever-growing array.

2. Non-scalable removal mechanism

Request removal is implemented via `O(n)` linear iteration.

Gas cost grows proportionally with array size.

3. Mandatory execution dependency

Claim operations require successful removal of request IDs.

There is no alternative or fallback execution path.

4. Lack of safeguards

- No maximum request limit
- No batching
- No pagination

## Attack Path

1. User deposits and stakes funds.

2. User creates multiple valid requests.

```solidity
for (uint256 i = 0; i < N; i++) {
    susdm.cooldownShares(shares / N);
}
```

Each call appends a request ID to `_userUnstakeIds[user]`. The array grows to size `N`.

3. Cooldown period elapses.

4. User attempts to claim.

```solidity
susdm.claimUnstake(requestId);
```

5. Claim fails due to gas exhaustion.

- `_removeUserUnstakeId()` performs a full array scan
- Gas cost becomes `O(N)`
- For sufficiently large `N`, transaction exceeds block gas limit and reverts

6. Permanent lock condition

- Removal is required for claim completion
- No alternative execution path exists
- Funds remain indefinitely locked in escrow

## Impact

This issue causes a permanent loss of access to funds for affected users.

Specifically:

- Users can enter a state where all claim operations revert due to gas constraints
- Funds held in escrow (pending redeem or unstake) become permanently unclaimable
- There is no protocol-level recovery mechanism once this state is reached

This represents a failure of protocol liveness, where valid state transitions (claiming funds) become unexecutable.

## Recommended Mitigation

- Use O(1) removal via index mapping

Replace linear search with indexed storage:

```solidity
mapping(address => mapping(uint256 => uint256)) userRequestIndex;
mapping(address => uint256[]) userRequestIds;
```

On insertion:

```solidity
userRequestIndex[user][requestId] = index;
```

On removal:

```solidity
uint256 index = userRequestIndex[user][requestId];
uint256 lastId = ids[ids.length - 1];

ids[index] = lastId;
userRequestIndex[user][lastId] = index;

ids.pop();
delete userRequestIndex[user][requestId];
```

- Enforce upper bounds on request count

```solidity
require(
    userRequestIds[user].length < MAX_REQUESTS,
    "too many requests"
);
```

## Proof of Concept

```solidity
function test_submissionValidity() public {
    address attacker = user1;

    _deposit(attacker, 1_000_000e6);
    _stake(attacker, 1_000_000e6);

    uint256 shares = susdm.balanceOf(attacker);

    vm.startPrank(attacker);

    uint256 iterations = 2000;
    uint256 sharePerRequest = shares / iterations;

    for (uint256 i = 0; i < iterations; i++) {
        susdm.cooldownShares(sharePerRequest);
    }

    vm.stopPrank();

    uint256[] memory ids = susdm.getUserUnstakeIds(attacker);
    assertGt(ids.length, 1500);

    vm.warp(block.timestamp + 7 days);

    vm.startPrank(attacker);

    uint256 lastId = ids[ids.length - 1];

    uint256 gasBefore = gasleft();
    susdm.claimUnstake(lastId);
    uint256 gasUsed = gasBefore - gasleft();

    vm.stopPrank();

    assertGt(gasUsed, 500_000);

    assertGt(susdm.getUserUnstakeIds(attacker).length, 1000);
}
```
