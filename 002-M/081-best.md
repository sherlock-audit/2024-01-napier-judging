Noisy Coal Python

high

# `swapUnderlyingForPt` does not sweep the remaining unused ETH back to users

## Summary

`swapUnderlyingForPt` does not sweep the remaining unused ETH back to users, resulting in a loss of assets for the victim.

## Vulnerability Detail

The Napier Router is designed to support both WETH and native ETH for its operation.

Assume that Bob set the `underlyingInMax` to 100 ETH. He deposited 100 Native ETH to the Router contract. He chooses the Native ETH approach.

The swap concluded that 90 ETH is needed to get his desired number of PT. So, in the swap callback, the 90 ETH will be wrapped to WETH and forwarded to the Napier Pool.

Notice that the remaining 10 ETH that is unused is not swept back to Bob at the end of the TX. Thus, the 10 ETH will be stolen by MEV or front-runner after the TX, as any ETH residing on the Router can be stolen.

> [!NOTE]
>
> When `underlyingInMax` is set to 100 ETH, that does not mean all the 100 ETH will be used up during the swap. The amount of underlying to be charged for the swap is based on the on-chain market condition when the TX is executed. Any remaining unused ETH must be returned back to the user.

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L239

```solidity
File: NapierRouter.sol
239:     function swapUnderlyingForPt(
240:         address pool,
241:         uint256 index,
242:         uint256 ptOutDesired,
243:         uint256 underlyingInMax,
244:         address recipient,
245:         uint256 deadline
246:     ) external payable override nonReentrant checkDeadline(deadline) returns (uint256) {
247:         // dev: Optimistically call to the `pool` provided by the untrusted caller.
248:         // And then verify the pool using CREATE2.
249:         (address underlying, address basePool) = INapierPool(pool).getAssets();
250: 
251:         // if `pool` doesn't matched, it would be reverted.
252:         if (INapierPool(pool) != PoolAddress.computeAddress(basePool, underlying, POOL_CREATION_HASH, address(factory)))
253:         {
254:             revert Errors.RouterPoolNotFound();
255:         }
256: 
257:         IERC20 pt = INapierPool(pool).principalTokens()[index];
258: 
259:         // Abi encode callback data to be used in swapCallback
260:         bytes memory data = new bytes(0xa0);
261:         {
262:             uint256 callbackType = uint256(CallbackType.SwapUnderlyingForPt);
263:             assembly {
264:                 // Equivanlent to:
265:                 // data = abi.encode(CallbackType.SwapUnderlyingForPt, underlying, basePool, CallbackDataTypes.SwapUnderlyingForPtData({payer: msg.sender, underlyingInMax: underlyingInMax}))
266:                 mstore(add(data, 0x20), callbackType)
267:                 mstore(add(data, 0x40), underlying)
268:                 mstore(add(data, 0x60), basePool)
269:                 mstore(add(data, 0x80), caller()) // dev: Ensure 'payer' is always 'msg.sender' to prevent allowance theft on callback.
270:                 mstore(add(data, 0xa0), underlyingInMax)
271:             }
272:         }
273: 
274:         uint256 prevBalance = pt.balanceOf(address(this));
275:         uint256 underlyingUsed = INapierPool(pool).swapUnderlyingForPt(
276:             index,
277:             ptOutDesired,
278:             address(this), // this contract will receive principal token from pool
279:             data
280:         );
281: 
282:         pt.safeTransfer(recipient, pt.balanceOf(address(this)) - prevBalance);
283:         return underlyingUsed;
284:     }
```

## Impact

Loss of assets for the victim.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L239

## Tool used

Manual Review

## Recommendation

The remaining unused ETH should be returned to the caller after the TX.