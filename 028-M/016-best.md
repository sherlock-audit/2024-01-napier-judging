Dancing Steel Coyote

high

# The last user can't quit ````Tranche```` and loss fund permanently

## Summary
After maturity of ````Tranche````, all users are intended to quit and get back their fund and profit. But actually the last user might fail to quit and permanently loss fund due to rounding error in the ````Tranche.withdraw()````. With the consideration there would be many ````Tranche```` instances based on different ````adapters```` and different ````maturity````, many users would suffer this risk, and this vulnerability would be easily triggered to cause users fund loss.

## Vulnerability Detail
The issue arises on L338 of ````withdraw()```` function and L676 \~ L680 of ````_computePrincipalTokenRedeemed()````, ````principalAmount```` is rounded down in favor of users rather than the protocol, which makes no enough principal token burned . This would cause the protocol have no enough ````target```` token to burn for last user.

```solidity
File: src\Tranche.sol
328:     function withdraw(
329:         uint256 underlyingAmount,
330:         address to,
331:         address from
332:     ) external override nonReentrant expired returns (uint256) {

337:         uint256 sharesRedeem = underlyingAmount.divWadDown(cscale);
338:         uint256 principalAmount = _computePrincipalTokenRedeemed(_gscales, sharesRedeem);
...
343:         _burnFrom(from, principalAmount);
...
350:     }

File: src\Tranche.sol
668:     function _computePrincipalTokenRedeemed(
669:         GlobalScales memory _gscales,
670:         uint256 _shares
671:     ) internal view returns (uint256) {
...
674:         if ((_gscales.mscale * MAX_BPS) / _gscales.maxscale >= oneSubTilt) {
...
676:             return (_shares.mulWadDown(_gscales.mscale) * MAX_BPS) / oneSubTilt;
677:         }
...
680:         return _shares.mulWadDown(_gscales.maxscale);
681:     }

```

The coded PoC shows more specific procedures how this would occur:
```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {TestTranche} from "./Tranche.t.sol";
import "forge-std/console2.sol";

contract TrancheRoundIssue is TestTranche {
    address alice = address(0x011);
    address bob = address(0x22);
    function setUp() public virtual override {
        super.setUp();
    }

    function testTrancheRoundIssue() public {
        // 1. issue some PT and YT
        deal(address(underlying), address(this), 1_000e6, true);
        tranche.issue(address(this), 1_000e6);

        // 2. alice and bob hold some YT
        yt.transfer(alice, yt.balanceOf(address(this)) / 2);
        yt.transfer(bob, yt.balanceOf(address(this)));

        // 3. after maturity
        vm.warp(_maturity);
        _simulateScaleIncrease();

        // 4. the withdraw() is called some times, typically causing "1" round error each time
        for (uint256 i; i < 10; ++i) {
            tranche.withdraw(2, address(this), address(this));
        }

        // 5. other operations such as collects, redeems and fee collection
        tranche.redeem(tranche.balanceOf(address(this)), address(this), address(this));
        vm.prank(management);
        tranche.claimIssuanceFees();
        vm.prank(alice);
        tranche.collect();

        // 6. bob becames the last unlucky guy, quit would fail
        vm.startPrank(bob);
        vm.expectRevert("ERC20: transfer amount exceeds balance");
        tranche.collect();
        vm.stopPrank();
    }
}
```

And the logs:
```solidity
2024-01-napier\napier-v1> forge test --match-test testTrancheRoundIssue
[⠔] Compiling...
[⠔] Compiling 40 files with 0.8.19
[⠊] Solc 0.8.19 finished in 74.33sCompiler run successful!
[⠒] Solc 0.8.19 finished in 74.33s

Running 1 test for test/unit/TrancheRoundIssue.t.sol:TrancheRoundIssue
[PASS] testTrancheRoundIssue() (gas: 1266016)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.58ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```


## Impact
The last user would fail to quit and permanently loss fund

## Code Snippet
https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L668
## Tool used

Manual Review

## Recommendation
```diff
diff --git a/napier-v1/src/Tranche.sol b/napier-v1/src/Tranche.sol
index 62d9562..457b6c5 100644
--- a/napier-v1/src/Tranche.sol
+++ b/napier-v1/src/Tranche.sol
@@ -673,11 +673,11 @@ contract Tranche is BaseToken, ReentrancyGuard, Pausable, ITranche {
         // If it's a sunny day, PT holders lose `tilt` % of the principal amount.
         if ((_gscales.mscale * MAX_BPS) / _gscales.maxscale >= oneSubTilt) {
             // Formula: principalAmount = (shares * mscale * MAX_BPS) / oneSubTilt
-            return (_shares.mulWadDown(_gscales.mscale) * MAX_BPS) / oneSubTilt;
+            return (_shares.mulWadUp(_gscales.mscale) * MAX_BPS) / oneSubTilt;
         }
         // If it's not a sunny day,
         // Formula: principalAmount = shares * maxscale
-        return _shares.mulWadDown(_gscales.maxscale);
+        return _shares.mulWadUp(_gscales.maxscale);
     }
```
