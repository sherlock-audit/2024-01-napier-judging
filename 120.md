Huge Carmine Kestrel

medium

# SFrxETHAdapter redemptionQueue waiting period can DOS adapter functions

## Links
https://github.com/FraxFinance/frax-ether-redemption-queue/blob/17ebebcddf31b7780e92c23a6b440dc789e5ceac/src/contracts/FraxEtherRedemptionQueue.sol#L417-L461
https://github.com/FraxFinance/frax-ether-redemption-queue/blob/17ebebcddf31b7780e92c23a6b440dc789e5ceac/src/contracts/FraxEtherRedemptionQueue.sol#L235-L246
https://github.com/sherlock-audit/2024-01-napier/blob/6313f34110b0d12677b389f0ecb3197038211e12/napier-v1/src/adapters/BaseLSTAdapter.sol#L71-L139
https://github.com/sherlock-audit/2024-01-napier/blob/6313f34110b0d12677b389f0ecb3197038211e12/napier-v1/src/adapters/BaseLSTAdapter.sol#L146-L169

## Summary
The waiting period between `rebalancer` address making a withdrawal request and the withdrawn funds being ready to claim from `FraxEtherRedemptionQueue` is extremely long which can lead to a significant period of time where some of the protocol's functions are either unusable or work in a diminished capacity.

## Vulnerability Detail
In `FraxEtherRedemptionQueue.sol`; the Queue wait time is stored in the state struct `redemptionQueueState` as `redemptionQueueState.queueLengthSecs` and is curently set to `1_296_000 Seconds` or `15 Days`; as recently as January however it was at `1_555_200 Seconds` or `18 Days`. View current setting by calling `redemptionQueueState()` [here](https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#readContract).

`BaseLSTAdapter::requestWithdrawal()` is an essential function which helps to maintain `bufferEth` at a defined, healthy level. 
`bufferEth` is a facility which smooth running of redemptions and deposits.

For `redemptions`; it allows users to redeem `underlying` without having to wait for any period of time.
However, redemption amounts requested which are less than `bufferEth` will be rejected as can be seen below in `BaseLSTAdapter::prefundedRedeem()`.
Further, there is nothing preventing `redemptions` from bringing `bufferEth` all the way to `0`.

```solidity
    function prefundedRedeem(address recipient) external virtual returns (uint256, uint256) {
        // SOME CODE

        // If the buffer is insufficient, shares cannot be redeemed immediately
        // Need to wait for the withdrawal to be completed and the buffer to be refilled.
>>>     if (assets > bufferEthCache) revert InsufficientBuffer();

        // SOME CODE
    }
```

For `deposits`; where `bufferEth` is too low, it keeps user deposits in the contract until a deposit is made which brings `bufferEth` above it's target, at which point it stakes. During this time, the deposits, which are kept in the adapter, do not earn any yield; making those funds unprofitable.

```solidity
    function prefundedDeposit() external nonReentrant returns (uint256, uint256) {
    // SOME CODE
>>>     if (targetBufferEth >= availableEth + queueEthCache) {
>>>         bufferEth = availableEth.toUint128();
            return (assets, shares);
        }
    // SOME CODE
    }
```

## Impact
If the `SFrxETHAdapter` experiences a large net redemption, bringing `bufferEth` significantly below `targetBufferEth`, the rebalancer can be required to make a withdrawal request in order to replenish the buffer.
However, this will be an ineffective action given the current, 15 Day waiting period. During the waiting period if `redemptions > deposits`, the bufferEth can be brought down to `0` which will mean a complete DOSing of the `prefundedRedeem()` function.

During the wait period too; if `redemptions >= deposits`, no new funds will be staked in `FRAX` so yields for users will decrease and may in turn lead to more redemptions.

These conditions could also necessitate the immediate calling again of `requestWithdrawal()`, given that withdrawal requests can only bring `bufferEth` up to it's target level and not beyond and during the wait period there could be further redemptions.

## Code Snippet
Simple example with Yield on `sFrxETHBalance` ignored:

Start off with 100 wETH deposited; 10 wETH `bufferEth`
> totalAssets() = (withdrawalQueueEth + bufferEth + sFrxETHBalance)
> totalAssets() = (0 + 10 + 90)

Net Redemption 5 wETH reduces bufferEth so rebalancer makes Withdrawl Request of 4.5 wETH to bring bufferEth to 10% (9.5 wEth)
> totalAssets() = (4.5 + 5 + 85.5)

During the wait period, continued Net Redemption reduces bufferEth further requiring another withdrawl request by rebalancer for 4.05 wEth
> totalAssets() = (0 + 4.5 + 85.5)

## Tool used
Foundry Testing
Manual Review

## Recommendation
Consider adding a function allowing the rebalancer call `earlyBurnRedemptionTicketNft()` in `FraxEtherRedemptionQueue.sol` when there is a necessity.
This will allow an immediate withdrawal for a fee of `0.5%`; see function [here]( https://github.com/FraxFinance/frax-ether-redemption-queue/blob/17ebebcddf31b7780e92c23a6b440dc789e5ceac/src/contracts/FraxEtherRedemptionQueue.sol#L417-L461
)
