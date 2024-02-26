Noisy Coal Python

medium

# AMM will revert if exchange rate is one

## Summary

The AMM will stop working unexpectedly when the `preTradeExchangeRate` is 1.0.

## Vulnerability Detail

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/libs/PoolMath.sol#L213

```solidity
File: PoolMath.sol
203:     function _getRateAnchor(
204:         uint256 totalBaseLptTimesN,
205:         uint256 lastLnImpliedRate,
206:         uint256 totalUnderlying18,
207:         int256 rateScalar,
208:         uint256 timeToExpiry
209:     ) internal pure returns (int256 rateAnchor) {
210:         // `extRate(t*) = e^(lastLnImpliedRate * yearsToExpiry(t))`
211:         // Get pre-trade exchange rate with zero-fee
212:         int256 preTradeExchangeRate = _getExchangeRateFromImpliedRate(lastLnImpliedRate, timeToExpiry);
..SNIP..
213:         // exchangeRate should not be below 1.
214:         // But it is mathematically almost impossible to happen because `exp(x) < 1` is satisfied for all `x < 0`.
215:         // Here x = lastLnImpliedRate * yearsToExpiry(t), which is very unlikely to be negative.(or
216:         // more accurately the natural log rounds down to zero). `lastLnImpliedRate` is guaranteed to be positive when it is set
217:         // and `yearsToExpiry(t)` is guaranteed to be positive because swap can only happen before maturity.
218:         // We still check for this case to be safe.
219:         require(preTradeExchangeRate > SignedMath.WAD);
220:         uint256 proportion = totalBaseLptTimesN.divWadDown(totalBaseLptTimesN + totalUnderlying18);
221:         int256 lnProportion = _logProportion(proportion);
222: 
223:         // Compute `rateAnchor(t) = extRate(t*) - ln(portion(t*)) / rateScalar(t)`
224:         rateAnchor = preTradeExchangeRate - lnProportion.divWadDown(rateScalar);
```

At Line 219, it requires that the `preTradeExchangeRate` be larger than 1.0. However, technically, the exchange rate can be 1.0 based on the comment in Line 213 that the exchange rate should not be below one, which means that the exchange rate should be 1.0 or above. Thus, when the `preTradeExchangeRate` is 1.0, the AMM will revert unexpectedly.

## Impact

The AMM might stop working unexpectedly. Breaking of core protocol/contract functionality.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/libs/PoolMath.sol#L213

## Tool used

Manual Review

## Recommendation

Consider making the following change:

```diff
function _getRateAnchor(
    uint256 totalBaseLptTimesN,
    uint256 lastLnImpliedRate,
    uint256 totalUnderlying18,
    int256 rateScalar,
    uint256 timeToExpiry
) internal pure returns (int256 rateAnchor) {
	..SNIP..
-    require(preTradeExchangeRate > SignedMath.WAD);
+    require(preTradeExchangeRate >= SignedMath.WAD);
```

Sidenote: This is also implemented in [Pendle's Math Library](https://github.com/pendle-finance/pendle-core-v2-public/blob/2de25376697d077629f28f5d2fc165582f565aac/contracts/libraries/math/MarketMathCore.sol#L251)