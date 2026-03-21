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
