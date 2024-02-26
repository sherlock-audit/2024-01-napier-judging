Noisy Coal Python

medium

# Permissioned rebalancing functions leading to loss of assets

## Summary

Permissioned rebalancing functions that could only be accessed by admin could lead to a loss of assets.

## Vulnerability Detail

Per the contest's README page, it stated that the admin/owner is "RESTRICTED". Thus, any finding showing that the owner/admin can steal a user's funds, cause loss of funds or harm to the users, or cause the user's fund to be struck is valid in this audit contest.

> Q: Is the admin/owner of the protocol/contracts TRUSTED or RESTRICTED?
>
> RESTRICTED

The following describes a way where the admin can block users from withdrawing their assets from the protocol

1. The admin calls the `setRebalancer` function to set the rebalance to a wallet address owned by them.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L245

```solidity
File: BaseLSTAdapter.sol
245:     function setRebalancer(address _rebalancer) external onlyOwner {
246:         rebalancer = _rebalancer;
247:     }
```

2. The admin calls the `setTargetBufferPercentage` the set the `targetBufferPercentage` to the smallest possible value of 1%. This will cause only 1% of the total ETH deposited by all the users to reside on the adaptor contract. This will cause the ETH buffer to deplete quickly and cause all the redemption and withdrawal to revert.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L251

```solidity
File: BaseLSTAdapter.sol
251:     function setTargetBufferPercentage(uint256 _targetBufferPercentage) external onlyRebalancer {
252:         if (_targetBufferPercentage < MIN_BUFFER_PERCENTAGE || _targetBufferPercentage > BUFFER_PERCENTAGE_PRECISION) {
253:             revert InvalidBufferPercentage();
254:         }
255:         targetBufferPercentage = _targetBufferPercentage;
256:     }
```

3. The owner calls the `setRebalancer` function again and sets the rebalancer address to `address(0)`. As such, no one has the ability to call functions that are only accessible by rebalancer. The `requestWithdrawal` and `requestWithdrawalAll` functions are only accessible by rebalancer. Thus, no one can call these two functions to replenish the ETH buffer in the adaptor contract.
4. When this state is reached, users can no longer withdraw their assets from the protocol, and their assets are stuck in the contract. This effectively causes them to lose their assets.

## Impact

Loss of assets for the victim.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L245

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L251

## Tool used

Manual Review

## Recommendation

To prevent the above scenario, the minimum `targetBufferPercentage` should be set to a higher percentage such as 5 or 10%, and the `requestWithdrawal` function should be made permissionless, so that even if the rebalancer does not do its job, anyone else can still initiate the rebalancing process to replenish the adaptor's ETH buffer for user's withdrawal.