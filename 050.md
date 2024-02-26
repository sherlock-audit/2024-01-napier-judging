Cool Beige Bat

medium

# incorrect allowance check in NapierRouter::removeLiquidityOneUnderlying()

## Summary
incorrect allowance check in NapierRouter::removeLiquidityOneUnderlying()

## Vulnerability Detail
In function removeLiquidityOneUnderlying(), we remove liquidity from pool to get underlying token and baseLpt in router. And then we will swap baseLpt to underlying in pool. So before we swap baseLpt to underlying, we need to check whether pool has enough allowance to transfer router's baseLpt token.
```solidity
In function removeLiquidityOneUnderlying(), before router 
    function removeLiquidityOneUnderlying(
        address pool,
        uint256 index,
        uint256 liquidity,
        uint256 underlyingOutMin,
        address recipient,
        uint256 deadline
    ) external override nonReentrant checkDeadline(deadline) returns (uint256) {
        ...
        // The withdrawn base LP token is exchanged for underlying in two different ways depending on the maturity.
        // If maturity has passed, redeem else swap for underlying.
        // 1. Swapping is used when maturity hasn't passed because redeeming is disabled before maturity.
        // 2. Redeeming is preferred because it doesn't cause slippage.
        if (block.timestamp < INapierPool(pool).maturity()) {
            // Swap base LP token for underlying
            // approve max
    @==>if (IERC20(basePool).allowance(address(this), basePool) < baseLptOut) {
                IERC20(basePool).forceApprove(pool, type(uint256).max);
            }
            uint256 removed = INapierPool(pool).swapExactBaseLpTokenForUnderlying(baseLptOut, address(this));
            underlyingOut += removed;
        }
```

It should be 
```solidity
        if (block.timestamp < INapierPool(pool).maturity()) {
            // Swap base LP token for underlying
            // approve max
            // (pool, not base pool)
@==>        if (IERC20(basePool).allowance(address(this), pool) < baseLptOut) {
                IERC20(basePool).forceApprove(pool, type(uint256).max);
            }
            uint256 removed = INapierPool(pool).swapExactBaseLpTokenForUnderlying(baseLptOut, address(this));
            underlyingOut += removed;
        }
```
## Impact
Incorrect allowance check.

## Code Snippet
https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L744-L752

## Tool used

Manual Review

## Recommendation
change basePool to pool
```solidity
        if (block.timestamp < INapierPool(pool).maturity()) {
            // Swap base LP token for underlying
            // approve max
   ==>    if (IERC20(basePool).allowance(address(this), pool) < baseLptOut) {
                IERC20(basePool).forceApprove(pool, type(uint256).max);
            }
            uint256 removed = INapierPool(pool).swapExactBaseLpTokenForUnderlying(baseLptOut, address(this));
            underlyingOut += removed;
        } 
```