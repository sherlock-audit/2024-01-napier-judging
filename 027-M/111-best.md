Curved Denim Seal

medium

# The pool verification in `NapierRouter` is prone to collision attacks

## Summary

Using computed pool addresses to verify the callback function callers is collision attack prone and can be abused in order to steal all token allowances of the `NapierRouter` contract

## Vulnerability Detail

With the current state of the `NapierRouter` contract implementation, both the `mintCallback` and `swapCallback` use the `_verifyCallback` function in order to verify that the address that called them is a Napier pool deployed by the `PoolFactory` attached to the router. This is done by computing the Create2 pool address using the `basePool` and `underlying` values passed in to either of the callbacks as arguments and the comparing that address to the `msg.sender`.

However, this method of verifying the caller address is collision prone, as the computation of the Create2 address is done by truncating a 256 bit keccak256 hash to 160 bits, meaning that for each address there are 2^96 possible hashes that will result in it after being truncated. Furthermore, what this means is that if a `basePool` and `underlying` combination that results in an address controlled by a malicious user is found, all token allowances given to the `NapierRouter` contract can be stolen.

In this [article](https://mystenlabs.com/blog/ambush-attacks-on-160bit-objectids-addresses) and this [report](https://github.com/sherlock-audit/2023-07-kyber-swap-judging/issues/90) from a past Sherlock contest, it is shown that this vulnerability will likely cost a few million dollars as of today, but due to the rapid advancement in computing capabilities, it is likely that it will cost much less a few years down the line.

## Impact

All token allowances of the `NapierRouter` can be stolen

## Code Snippet

[NapierRouter.sol#L65-L68](https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L65-L68)

## Tool used

Manual Review

## Recommendation

In the `PoolFactory` contract, create a function that verifies that a given pool address has been deployed by it using the `_pools` mapping. Example implementation:

```diff
contract PoolFactory is IPoolFactory, Ownable2Step {
    ...
    /// @notice Mapping of NapierPool to PoolAssets
    mapping(address => PoolAssets) internal _pools;
    ...
+   function isFactoryDeployedPool(address poolAddress) external view returns (bool) {
+       PoolAssets memory poolAssets = _pools[poolAddress];
+       return (
+           poolAssets.basePool != address(0) &&
+           poolAssets.underlying != address(0) &&
+           poolAssets.principalTokens.length != 0
+       );
    }
}
```

And then use it in the `NapierRouter::_verifyCallback` function in place of the computed address comparison logic:

```diff
    function _verifyCallback(address basePool, address underlying) internal view {
-       if (
-           PoolAddress.computeAddress(basePool, underlying, POOL_CREATION_HASH, address(factory))
-               != INapierPool(msg.sender)
-       ) revert Errors.RouterCallbackNotNapierPool();
+       if (factory.isFactoryDeployedPool(msg.sender)) revert Errors.RouterCallbackNotNapierPool();
    }
```
