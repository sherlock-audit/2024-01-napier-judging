Noisy Coal Python

high

# Users who claimed more frequently received lesser yield

## Summary

Users who claimed more frequently received lesser yield, leading to a loss of assets for the affected users

## Vulnerability Detail

The formula for computing the accrued interest in Target Token is as follows if `sharesDeposited` > `sharesNow` :

$$
y(\frac{1}{lscale} - \frac{1}{maxScale})
$$

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L704

```solidity
File: Tranche.sol
704:     function _computeAccruedInterestInTarget(
705:         uint256 _maxscale,
706:         uint256 _lscale,
707:         uint256 _yBal
708:     ) internal pure returns (uint256) {
..SNIP..
722:         uint256 sharesNow = _yBal.divWadUp(_maxscale); // rounded up to prevent a user from withdrawing more shares than this contract has.
723:         uint256 sharesDeposited = _yBal.divWadDown(_lscale);
724:         if (sharesDeposited <= sharesNow) {
725:             return 0;
726:         }
727:         return sharesDeposited - sharesNow; // rounded down
728:     }
```

Assume that from $T1$ to $T3$, the current scale progressively increases from 1.0 to 1.5 at a constant rate.

Both Alice and Bob issued 100 PY and 100 YT at $T1$ when the current scale is 1.0.

One of the important invariants is that multiple small operations should have the same effect as 1 large operation.

#### First Scenario (Alice)

Alice collects her yield at $T2$ when the current scale is 1.25, and she accrued an interest of 20 Target Tokens.

$$
y(\frac{1}{lscale} - \frac{1}{maxScale}) = \frac{100}{1.0} - \frac{100}{1.25} = 20 T
$$

The collect function will then redeem 20 Target Tokens from the adaptor, she will receive 25 underlying tokens. `lscales[Alice]` will be updated to 1.25

$$
20 T \times currentRate = 20T * 1.25 = 25U
$$

Alice collects her yield again at T2 when the current scale is 1.5, and she accrued an interest of 13.334 Target Tokens.

$$
y(\frac{1}{lscale} - \frac{1}{maxScale}) = \frac{100}{1.25} - \frac{100}{1.5} = 13.334T
$$

The collect function will then redeem 20 Target Tokens from the adaptor, she will receive 20.001 underlying tokens. 

$$
13.334 T \times currentRate = 13.334 T * 1.5 = 20U
$$

Alice received a total of 45 underlying tokens (25 U + 20U) from $T1$ to $T3$.

#### Second Scenario (Bob)

Bob collects his yield at $T3$​ when the current scale is 1.5, and she accrued an interest of 33.3333 Target Tokens.

$$
y(\frac{1}{lscale} - \frac{1}{maxScale}) = \frac{100}{1.0} - \frac{100}{1.5} = 33.3333 T
$$

The collect function will then redeem 33.3333 Target Tokens from the adaptor, he will receive 20.001 underlying tokens. 

$$
33.3333 T \times currentRate = 33.3333 T * 1.5 = 50U
$$

Bob received a total of 50 underlying tokens from $T1$ to $T3$.

#### Conclusion

The above shows that if one collects their yield more frequently, they will receive fewer underlying assets. In the above scenarios, Alice received 5 underlying assets less than Bob.

Even if the users do not explicitly collect their yield via the `collect` function, the protocol will automatically collect their yield when new PT + YT tokens are issued to them, which also leads to a similar problem.

## Impact

Loss of assets for the affected users.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L704

## Tool used

Manual Review

## Recommendation

The total yield a user receives from $T1$ to $T3$ should be the same regardless of the number of times the collect yield function is executed.

Since Alice and Bob hold the same amount of YT tokens for the same period, they should receive the same yield.