# L1 - `performUpkeep` reverts entirely when any eligible asset fails the minimum auction size check, blocking live auction endings and locking auctioned funds in the contract
## Summary

`performUpkeep` processes new auction starts before processing endings of expired auctions. If any eligible-to-start asset fails the `AmountBelowMinAuctionSize` validation (due to price volatility between `checkUpkeep` and `performUpkeep`), the entire transaction reverts. Expired auctions that need to be ended are also blocked. Their balances remain locked in the auction contract, bids are impossible (expired by `elapsedTime` > `auctionDuration`), CowSwap settlement is also blocked (same check in `isValidSignature`), and the CowSwap approval set at auction start remains active — until the next successful `performUpkeep` cycle.

## Description

The root ordering issue is in `BaseAuction : performUpkeep`:

```solidity
function performUpkeep(
    bytes calldata performData
) external whenNotPaused whenAssetOutConfigured onlyRole(Roles.AUCTION_WORKER_ROLE) {
   ,,,,

    if (hasFeeAggregator && eligibleAssets.length > 0) {
        s_feeAggregator.transferForSwap(address(this), eligibleAssets);
    }

    ,,,,

    for (uint256 i; i < eligibleAssets.length; ++i) {
        // ...
        (assetPrice,,) = _getAssetPrice(asset, true); // strict validation

        uint256 availableAssetUsdValue =
            (eligibleAssets[i].amount * assetPrice) / (10 ** assetDecimals);

        if (availableAssetUsdValue < assetParams.minAuctionSizeUsd) {
            revert AmountBelowMinAuctionSize(
                availableAssetUsdValue,
                assetParams.minAuctionSizeUsd
            );
        }

        // ...
    }

    for (uint256 i; i < endedAuctions.length; ++i) {
        _onAuctionEnd(endedAuctions[i], hasFeeAggregator);
        delete s_auctionStarts[asset];
        emit AuctionEnded(asset);
    }

    ,,,,
}
```

Once the auction duration elapses for an asset, BOTH `bid()` and `isValidSignature()` reject it:

```solidity
// In bid()
if (auctionStart == 0 || elapsedTime > assetParams.auctionDuration) {
    revert InvalidAuction(asset);
}

// In isValidSignature()
if (elapsedTime > assetParams.auctionDuration) {
    revert InvalidAuction(address(order.sellToken));
}
```
# Root Cause

performUpkeep unconditionally processes all eligibleAssets (auction starts) before any endedAuctions (auction endings). A revert in the starts loop prevents the endings loop from executing. There is no try/catch mechanism or independent execution boundary between the two phases.

# Attack Path

1. A USDC auction is started via `performUpkeep`:

   - USDC is transferred to the auction contract.
   - CowSwap vault relayer is approved.
   - WETH has sufficient balance in feeAggregator to be eligible for a new auction (e.g., 1 WETH at  
     `4,000 = $4,000 > $1,000 minimum`).

2. The USDC auction duration elapses:

   - Calls to `bid(USDC, ...)` revert with `InvalidAuction`.
   - CowSwap's `isValidSignature` also reverts with `InvalidAuction`.

3. `checkUpkeep` is called and returns:

```text
eligibleAssets: [{WETH, 1e18}]
endedAuctions: [USDC]
```

4. Between `checkUpkeep` and `performUpkeep`, WETH price drops to `$0.50` (via Chainlink feed update or transmit call).

5. `performUpkeep` is called with the stale `performData`:

   - `transferForSwap([{WETH, 1e18}])` succeeds (`feeAggregator` has WETH).
   - Processing WETH:

```text
assetPrice = $0.50
availableAssetUsdValue = $0.50
$0.50 < $1,000
REVERT AmountBelowMinAuctionSize
```

6. Everything reverts including the USDC ending.

7. State After Revert

- `s_auctionStarts[USDC]` remains set (auction still "live")
- `100k USDC` remains locked in auction contract
- USDC cannot be bid on (expired)
- USDC cannot be CowSwap-settled (expired)
- CowSwap approval for USDC is still set (but unusable)
- USDC not returned to `feeAggregator`

8. Persistence Condition

This persists until the next `checkUpkeep` cycle returns `performData` that excludes WETH from eligibleAssets (e.g., WETH price is still low making it ineligible at `checkUpkeep` time).

# Impact

- Auctioned asset funds are temporarily locked in the auction contract — unable to be returned to `feeAggregator` or auctioned again.
- Auction proceeds (LINK) accumulated in the contract are not forwarded to `s_assetOutReceiver` (Reserves).
- CowSwap vault relayer approval remains set after the auction has expired, even though it cannot be exploited (since `isValidSignature` blocks expired auctions).
- Duration: Until the next CRE workflow cycle where `checkUpkeep` produces `performData` without the problematic eligible asset — could be multiple cycles if the price oscillates around the minimum threshold.
- The USDC could remain stuck for an indeterminate time if the CRE workflow always bundles an ineligible asset with expired auctions.

# Mitigation

Process auction endings before starts, or decouple them into separate loops with independent failure handling:
```solidity
// Option A: Reorder - end expired auctions first
function performUpkeep(
    bytes calldata performData
  ) external whenNotPaused whenAssetOutConfigured onlyRole(Roles.AUCTION_WORKER_ROLE) {

for (uint256 i; i < endedAuctions.length; ++i) {
    if (s_auctionStarts[endedAuctions[i]] == 0) revert InvalidAuction(endedAuctions[i]);
    _onAuctionEnd(endedAuctions[i], hasFeeAggregator);
    delete s_auctionStarts[endedAuctions[i]];
    emit AuctionEnded(endedAuctions[i]);
}

// Then attempt starts (failures here don't affect endings)
if (hasFeeAggregator && eligibleAssets.length > 0) {
    s_feeAggregator.transferForSwap(address(this), eligibleAssets);
}
for (uint256 i; i < eligibleAssets.length; ++i) { ... }

}

// Option B: Skip-on-fail for eligible assets instead of reverting
```

# Proof of Concept

```solidity
function testSubmissionValidity() public {
    uint256 usdcAmount = 100_000e6;
    _startAuction(address(mockUSDC), usdcAmount);
    console2.log("[setup] USDC auction started. Auction holds:",
        IERC20(address(mockUSDC)).balanceOf(address(auction)));

    deal(address(mockWETH), address(feeAggregator), 1e18);
    console2.log("[setup] feeAggregator WETH balance:", IERC20(address(mockWETH)).balanceOf(address(feeAggregator)));

    uint24 usdcDuration = auction.getAssetParams(address(mockUSDC)).auctionDuration;
    skip(usdcDuration + 1);
    _refreshPrices();

    _fundBidder(100_000e18);
    vm.expectRevert(
        abi.encodeWithSelector(BaseAuction.InvalidAuction.selector, address(mockUSDC))
    );
    _bid(address(mockUSDC), 1e6);
    console2.log("[pre-attack] USDC bid correctly reverts - auction expired");

    (, bytes memory performData) = auction.checkUpkeep("");
    (Common.AssetAmount[] memory eligibleAssets, address[] memory endedAuctions) =
        abi.decode(performData, (Common.AssetAmount[], address[]));

    console2.log("[checkUpkeep] eligibleAssets count:", eligibleAssets.length);
    console2.log("[checkUpkeep] endedAuctions count:", endedAuctions.length);

    assertEq(eligibleAssets.length, 1, "WETH must be eligible");
    assertEq(endedAuctions.length, 1, "USDC must be in ended list");
    assertEq(endedAuctions[0], address(mockUSDC), "USDC must be the ended auction");
    assertEq(eligibleAssets[0].asset, address(mockWETH), "WETH must be the eligible asset");

    _transmitPrices(0.5e18, 1e18, 20e18);

    _changePrank(auctionAdmin);
    vm.expectRevert(
        abi.encodeWithSelector(
            BaseAuction.AmountBelowMinAuctionSize.selector,
            uint256(5e17),
            uint256(1_000e18)
        )
    );
    auction.performUpkeep(performData);
    console2.log("[EXPLOIT] performUpkeep reverted - USDC auction cannot be ended");

    uint256 usdcAuctionStart = auction.getAuctionStart(address(mockUSDC));
    uint256 usdcInAuction    = IERC20(address(mockUSDC)).balanceOf(address(auction));
    uint256 usdcInAggregator = IERC20(address(mockUSDC)).balanceOf(address(feeAggregator));
    uint256 wethInAggregator = IERC20(address(mockWETH)).balanceOf(address(feeAggregator));

    assertTrue(usdcAuctionStart != 0, "BUG: USDC auctionStart still set");
    assertTrue(usdcInAuction > 0, "BUG: USDC locked in auction contract");
    assertEq(usdcInAggregator, 0, "BUG: USDC not in feeAggregator");
    assertEq(wethInAggregator, 1e18, "WETH correctly rolled back to feeAggregator");

    console2.log(" -STUCK STATE- ");
    console2.log("USDC auction start (should be 0):", usdcAuctionStart);
    console2.log("USDC locked in auction (should be 0):", usdcInAuction);
    console2.log("USDC in feeAggregator (should be non-zero):", usdcInAggregator);

    vm.expectRevert(
        abi.encodeWithSelector(BaseAuction.InvalidAuction.selector, address(mockUSDC))
    );
    _bid(address(mockUSDC), 1e6);

    console2.log("USDC bids also blocked. Funds remain locked until next checkUpkeep cycle.");
}
```
- EXpected Output :

```solidity
[PASS] testSubmissionValidity() (gas: 870246)
Logs:
  [setup] USDC auction started. Auction holds: 100000000000
  [setup] feeAggregator WETH balance: 1000000000000000000
  [pre-attack] USDC bid correctly reverts - auction expired
  [checkUpkeep] eligibleAssets count: 1
  [checkUpkeep] endedAuctions count: 1
  [EXPLOIT] performUpkeep reverted - USDC auction cannot be ended
   -STUCK STATE- 
  USDC auction start (should be 0): 604801
  USDC locked in auction (should be 0): 100000000000
  USDC in feeAggregator (should be non-zero): 0
  USDC bids also blocked. Funds remain locked until next checkUpkeep cycle.
  ```