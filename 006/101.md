Noisy Coal Python

medium

# `swapUnderlyingForYt` revert due to rounding issues

## Summary

The core function (`swapUnderlyingForYt`) of the Router will revert due to rounding issues. Users who intend to swap underlying assets to YT tokens via the Router will be unable to do so.

## Vulnerability Detail

The `swapUnderlyingForYt` allows users to swap underlying assets to a specific number of YT tokens they desire.

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L353

```solidity
File: NapierRouter.sol
297:     function swapUnderlyingForYt(
298:         address pool,
299:         uint256 index,
300:         uint256 ytOutDesired,
301:         uint256 underlyingInMax,
302:         address recipient,
303:         uint256 deadline
304:     ) external payable override nonReentrant checkDeadline(deadline) returns (uint256) {
..SNIP..
330:             // Variable Definitions:
331:             // - `uDeposit`: The amount of underlying asset that needs to be deposited to issue PT and YT.
332:             // - `ytOutDesired`: The desired amount of PT and YT to be issued.
333:             // - `cscale`: Current scale of the Tranche.
334:             // - `maxscale`: Maximum scale of the Tranche (denoted as 'S' in the formula).
335:             // - `issuanceFee`: Issuance fee in basis points. (10000 =100%).
336: 
337:             // Formula for `Tranche.issue`:
338:             // ```
339:             // shares = uDeposit / s
340:             // fee = shares * issuanceFeeBps / 10000
341:             // pyIssue = (shares - fee) * S
342:             // ```
343: 
344:             // Solving for `uDeposit`:
345:             // ```
346:             // uDeposit = (pyIssue * s / S) / (1 - issuanceFeeBps / 10000)
347:             // ```
348:             // Hack:
349:             // Buffer is added to the denominator.
350:             // This ensures that at least `ytOutDesired` amount of PT and YT are issued.
351:             // If maximum scale and current scale are significantly different or `ytOutDesired` is small, the function might fail.
352:             // Without this buffer, any rounding errors that reduce the issued PT and YT could lead to an insufficient amount of PT to be repaid to the pool.
353:             uint256 uDepositNoFee = cscale * ytOutDesired / maxscale;
354:             uDeposit = uDepositNoFee * MAX_BPS / (MAX_BPS - (series.issuanceFee + 1)); // 0.01 bps buffer
```

Line 353-354 above compute the number of underlying deposits needed to send to the Tranche to issue the amount of YT token the users desired. It attempts to add a buffer of 0.01 bps buffer to prevent rounding errors that could lead to insufficient PT being repaid to the pool and result in a revert. During the audit, it was found that this buffer is ineffective in achieving its purpose.

The following example/POC demonstrates a revert could still occur due to insufficient PT being repaid despite having a buffer:

Let the state be the following:

- cscale = 1.2e18
- maxScale = 1.25e18
- ytOutDesired = 123
- issuanceFee = 0% (For simplicity's sake, the fee is set to zero. Having fee or not does not affect the validity of this issue as this is a math problem)

The following computes the number of underlying assets to be transferred to the Tranche to mint/issue PY + YT

```solidity
uDepositNoFee = cscale * ytOutDesired / maxscale;
uDepositNoFee = 1.2e18 * 123 / 1.25e18 = 118.08 = 118 (Round down)

uDeposit = uDepositNoFee * MAX_BPS / (MAX_BPS - (series.issuanceFee + 1))
uDeposit = 118 * 10000 / (10000 - (0 + 1)) = 118.0118012 = 118 (Round down)
```

Subsequently, the code will perform a flash-swap via the `swapPtForUnderlying` function. It will borrow 123 PT from the pool, which must be repaid later.

In the swap callback function, the code will transfer 118 underlying assets to the Tranche and execute the `Tranche.issue` function to mint/issue PY + YT.

Within the `Tranche.issue` function, it will trigger the `adapter.prefundedDeposit()` function to mint the estETH/shares. The following is the number of estETH/shares minted:

```solidity
shares = assets * (total supply/total assets)
sahres = 118 * 100e18 / 120e18 = 98.33333333 = 98 shares
```

Next, Line 219 below of the `Tranche.issue` function will compute the number of PY+YT to be issued/minted

```solidity
issued = (sharesUsed - fee).mulWadDown(_maxscale);
issued = (sharesUsed - 0).mulWadDown(_maxscale);
issued = sharesUsed.mulWadDown(_maxscale);

issued = sharesUsed * _maxscale / WAD
issued = 98 * 1.25e18 / 1e18 = 122.5 = 122 PT (Round down)
```

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L219

```solidity
File: Tranche.sol
179:     function issue(
180:         address to,
181:         uint256 underlyingAmount
182:     ) external nonReentrant whenNotPaused notExpired returns (uint256 issued) {
..SNIP..
217:         uint256 sharesUsed = sharesMinted + accruedInTarget;
218:         uint256 fee = sharesUsed.mulDivUp(issuanceFeeBps, MAX_BPS);
219:         issued = (sharesUsed - fee).mulWadDown(_maxscale);
```

At the end of the `Tranche.issue` function, 122 PY + YT is issued/minted back to the Router.

Note that 123 PT was flash-loaned earlier, and 123 PT needs to be repaid. Otherwise, the code at Line 164 below will revert. The main problem is that only 122 PY was issued/minted (a shortfall of 1 PY). Thus, the swap TX will revert at the end.

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L164

```solidity
File: NapierRouter.sol
085:     function swapCallback(int256 underlyingDelta, int256 ptDelta, bytes calldata data) external override {
..SNIP..
161:             uint256 pyIssued = params.pt.issue({to: address(this), underlyingAmount: params.underlyingDeposit});
162: 
163:             // Repay the PT to Napier pool
164:             if (pyIssued < pyDesired) revert Errors.RouterInsufficientPtRepay();
```

## Impact

The core function (swapUnderlyingForYt) of the Router will break. Users who intend to swap underlying assets to YT tokens via the Router will not be able to do so.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L353

## Tool used

Manual Review

## Recommendation

The buffer does not appear to be the correct approach to manage this rounding error. One could increase the buffer from 0.01% to 1% and solve the issue in the above example, but a different or larger number might cause a rounding error to surface again. Also, a larger buffer means that many unnecessary PTs will be issued. 

Thus, it is recommended that a round-up division be performed when computing the `uDepositNoFee` and `uDeposit` using functions such as `divWadUp` so that the issued/minted PT can cover the debt.