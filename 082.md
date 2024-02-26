Noisy Coal Python

high

# Benign esfrxETH holders incur more loss than expected

## Summary

Malicious esfrxETH holders can avoid "pro-rated" loss and have the remaining esfrxETH holders incur all the loss due to the fee charged by FRAX during unstaking. As a result, the rest of the esfrxETH holders incur more losses than expected compared to if malicious esfrxETH holders had not used this trick in the first place.

## Vulnerability Detail

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/frax/SFrxETHAdapter.sol#L22

```solidity
File: SFrxETHAdapter.sol
17: /// @title SFrxETHAdapter - esfrxETH
18: /// @dev Important security note:
19: /// 1. The vault share price (esfrxETH / WETH) increases as sfrxETH accrues staking rewards.
20: /// However, the share price decreases when frxETH (sfrxETH) is withdrawn.
21: /// Withdrawals are processed by the FraxEther redemption queue contract.
22: /// Frax takes a fee at the time of withdrawal requests, which temporarily reduces the share price.
23: /// This loss is pro-rated among all esfrxETH holders.
24: /// As a mitigation measure, we allow only authorized rebalancers to request withdrawals.
25: ///
26: /// 2. This contract doesn't independently keep track of the sfrxETH balance, so it is possible
27: /// for an attacker to directly transfer sfrxETH to this contract, increase the share price.
28: contract SFrxETHAdapter is BaseLSTAdapter, IERC721Receiver {
```

In the `SFrxETHAdapter`'s comments above, it is stated that the share price will decrease due to the fee taken by FRAX during the withdrawal request. This loss is supposed to be 'pro-rated' among all esfrxETH holders. However, this report reveals that malicious esfrxETH holders can circumvent this 'pro-rated' loss, leaving the remaining esfrxETH holders to bear the entire loss. Furthermore, the report demonstrates that the current mitigation measure, which allows only authorized rebalancers to request withdrawals, is insufficient to prevent this exploitation.

Whenever a rebalancers submit a withdrawal request to withdraw staked ETH from FRAX, it will first reside in the mempool of the blockchain and anyone can see it. Malicious esfrxETH holders can front-run it to withdraw their shares from the adaptor.

When the withdrawal request TX is executed, the remaining esfrxETH holders in the adaptor will incur the fee. Once executed, the malicious esfrxETH deposits back to the adaptors.

Note that no fee is charged to the users for any deposit or withdrawal operation. Thus, as long as the gain from this action is more than the gas cost, it makes sense for the esfrxETH holders to do so.

## Impact

The rest of the esfrxETH holders incur more losses than expected compared to if malicious esfrxETH holders had not used this trick in the first place.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/frax/SFrxETHAdapter.sol#L22

## Tool used

Manual Review

## Recommendation

The best way to discourage users from withdrawing their assets and depositing them back to take advantage of a particular event is to impose a fee upon depositing and withdrawing. 