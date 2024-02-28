Noisy Coal Python

high

# Victim's fund can be stolen due to rounding error and exchange rate manipulation

## Summary

Victim's funds can be stolen by malicious users by exploiting the rounding error and through exchange rate manipulation.

## Vulnerability Detail

The LST Adaptor attempts to guard against the well-known vault inflation attack by reverting the TX when the amount of shares minted is rounded down to zero in Line 78 below.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L71

```solidity
File: BaseLSTAdapter.sol
71:     function prefundedDeposit() external nonReentrant returns (uint256, uint256) {
72:         uint256 bufferEthCache = bufferEth; // cache storage reads
73:         uint256 queueEthCache = withdrawalQueueEth; // cache storage reads
74:         uint256 assets = IWETH9(WETH).balanceOf(address(this)) - bufferEthCache; // amount of WETH deposited at this time
75:         uint256 shares = previewDeposit(assets);
76: 
77:         if (assets == 0) return (0, 0);
78:         if (shares == 0) revert ZeroShares();
```

However, this control alone is not sufficient to guard against vault inflation attacks. 

Let's assume the following scenario (ignoring fee for simplicity's sake):

1. The victim initiates a transaction that deposits 10 ETH as the underlying asset when there are no issued estETH shares.
2. The attacker observes the victim’s transaction and deposits 1 wei of ETH (issuing 1 wei of estETH share) before the victim’s transaction. 1 wei of estETH share worth of PT and TY will be minted to the attacker.
3. Then, the attacker executes a transaction to directly transfer 5 stETH to the adaptor. The exchange rate at this point is `1 wei / (5 ETH + 1 wei)`. Note that the [`totalAssets`](https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/lido/StEtherAdapter.sol#L115) function uses the `balanceOf` function to compute the total underlying assets owned by the adaptor. Thus, this direct transfer will increase the total assets amount.
4. When the victim’s transaction is executed, the number of estETH shares issued is calculated as `10 ETH * 1 wei / (5 ETH + 1 wei)`, resulting in 1 wei being issued due to round-down.
5. The attacker will combine the PT + YT obtained earlier to redeem 1 wei of estETH share from the adaptor.
6. The attacker, holding 50% of the issued estETH shares (indirectly via the PT+YT he owned), receives `(15 ETH + 1 wei) / 2` as the underlying asset. 
7. The attacker seizes 25% of the underlying asset (2.5 ETH) deposited by the victim.

This scenario demonstrates that even when a revert is triggered due to the number of issued estETH share being 0, it does not prevent the attacker from capturing the user’s funds through exchange rate manipulation.

## Impact

Loss of assets for the victim.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L71

## Tool used

Manual Review

## Recommendation

Following are some of the measures that could help to prevent such an attack:

- Mint a certain amount of shares to zero address (dead address) during contract deployment (similar to what has been implemented in Uniswap V2)
- Avoid using the `balanceOf` so that malicious users cannot transfer directly to the contract to increase the assets per share. Track the total assets internally via a variable.