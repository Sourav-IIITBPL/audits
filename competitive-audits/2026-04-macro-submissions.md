# S1 - Permit2 deadline not enforced in `ClearMacroForwarderV1WithPermit2:runPermit2AndMacro` no-upgrade path thus expired authorizations remain permanently executable.

## Summary

When `runPermit2AndMacro` is called with `upgradeSuperToken == address(0)`, the contract verifies the Permit2 signature manually via `_verifyPermit2Signature` but never checks `p.permit.deadline` against `block.timestamp`. An expired Permit2 permit remains valid indefinitely from the contract's perspective.

## Root Cause

In `ClearMacroForwarderV1WithPermit2.sol : 41`, when `runPermit2AndMacro` function is called with `p.upgradeSuperToken == address(0)`, no tokens need to be pulled, so the code takes the manual signature-verification branch instead of calling Permit2's `permitWitnessTransferFrom`:

https://github.com/sherlock-audit/2026-04-macro/blob/main/protocol-monorepo/packages/ethereum-contracts/contracts/utils/ClearMacroForwarderV1WithPermit2.sol#L41-L69

```solidity
function _validatePermitAndMaybePull(  
    Permit2MacroParams calldata p,  
    IClearMacro m,  
    bytes calldata params  
) internal {  
   , , ,  

    if (p.upgradeSuperToken != address(0)) {  
        _pullAndUpgrade(p); // <- calls Permit2 directly. deadline enforced internally.  
    } else {  
        if (
            !_verifyPermit2Signature(
                p.permit,
                p.owner,
                p.spender,
                p.witness,
                p.witnessTypeString,
                p.signature
            )
        ) {  
            revert InvalidSignature();  
        } // <- No deadline check here.  
    }  
}
```

Inside `_verifyPermit2Signature`, the code only verifies the cryptographic signature. It never reads `p.permit.deadline`:

https://github.com/sherlock-audit/2026-04-macro/blob/main/protocol-monorepo/packages/ethereum-contracts/contracts/utils/ClearMacroForwarderV1WithPermit2.sol#L104-L115

```solidity
function _verifyPermit2Signature( , , , ) internal view returns (bool) {  
    return SignatureChecker.isValidSignatureNow(  
        owner,
        _permit2Digest(
            permit,
            spender,
            witness,
            witnessTypeString
        ),
        signature  
    );  
}
```

The `_permit2Digest` function includes `permit.deadline` inside `structHash`, meaning the deadline is part of the signed message, but the deadline is never compared to `block.timestamp`.

An expired permit will continue to verify as valid.

## Internal Pre-conditions

- User signs a Permit2 `PermitTransferFrom` with a finite deadline.
- The call path must enter the no-upgrade branch: `p.upgradeSuperToken == address(0)`.
- The signed witness (`macro + params`) remains valid (i.e., macro nonce unused and witness unchanged).

## External Pre-conditions

- A relayer or third party has access to the user’s signed Permit2 message (e.g., shared off-chain execution flow etc).
- No external system enforces expiration of the signed message.

## Attack Path

1. User signs a Permit2 permit authorizing ClearMacro action `X`, with:

```text
permit.deadline = T (e.g. now + 1 hour)
ClearMacro validBefore = 0 (no expiry set)
```

2. Time `T` passes.

User believes the signed authorization has expired.

3. Any relayer submits `runPermit2AndMacro` with the old signature:

```solidity
upgradeSuperToken = address(0)
```

4. `_verifyPermit2Signature` succeeds:

```text
signature is valid
deadline not checked
```

5. `_validatePayload` succeeds:

```text
validBefore == 0
=> no ClearMacro-level expiry
```

6. The macro executes on behalf of the user:

```text
user super tokens are moved
despite the Permit2 permit having expired
```

## Impact

- Users cannot rely on `permit.deadline` as an expiry mechanism, as expired signatures remain executable indefinitely.
- This creates a false sense of security, where time-bound authorizations can still be used long after expiry.
- Any party with access to the signature can execute it at any future time, leading to unauthorized macro execution on behalf of the user.
- The optional `validBefore` field does not mitigate this reliably, as users and integrators are not required or expected to mirror the Permit2 deadline.
- This breaks the core security guarantee of Permit2, where signatures are expected to be strictly time-limited.

## PoC

No response

## Mitigation

Add an explicit deadline check in `_validatePermitAndMaybePull` before proceeding:

```solidity
function _validatePermitAndMaybePull(  
    Permit2MacroParams calldata p,  
    IClearMacro m,  
    bytes calldata params  
) internal {  
   ,,,  

    if (p.upgradeSuperToken != address(0)) {  
        _pullAndUpgrade(p);                         
    } else {  
        if (block.timestamp > p.permit.deadline) {
            revert DeadlineExpired();
        } // <- Add this

        if (
            !_verifyPermit2Signature(
                p.permit,
                p.owner,
                p.spender,
                p.witness,
                p.witnessTypeString,
                p.signature
            )
        ) {  
            revert InvalidSignature();  
        }                             
    }  
}
```