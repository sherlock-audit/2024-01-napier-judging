Noisy Coal Python

medium

# FRAX admin can adjust fee rate to harm Napier and its users

## Summary

FRAX admin can adjust fee rates to harm Napier and its users, preventing Napier users from withdrawing.

## Vulnerability Detail

Per the contest page, the admins of the protocols that Napier integrates with are considered "RESTRICTED". This means that any issue related to FRAX's admin action that could negatively affect Napier protocol/users will be considered valid in this audit contest.

> Q: Are the admins of the protocols your contracts integrate with (if any) TRUSTED or RESTRICTED?
> RESTRICTED

Following is one of the ways that FRAX admin can harm Napier and its users.

FRAX admin can set the fee to 100%.

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3413

```solidity
File: FraxEtherRedemptionQueue.sol
217:     /// @notice Sets the fee for redeeming
218:     /// @param _newFee New redemption fee given in percentage terms, using 1e6 precision
219:     function setRedemptionFee(uint64 _newFee) external {
220:         _requireSenderIsTimelock();
221:         if (_newFee > FEE_PRECISION) revert ExceedsMaxRedemptionFee(_newFee, FEE_PRECISION);
222: 
223:         emit SetRedemptionFee({ oldRedemptionFee: redemptionQueueState.redemptionFee, newRedemptionFee: _newFee });
224: 
225:         redemptionQueueState.redemptionFee = _newFee;
226:     }
```

When the adaptor attempts to redeem the staked ETH from FRAX via the `enterRedemptionQueue` function, the 100% fee will consume the entire amount of the staked fee, leaving nothing for Napier's adaptor.

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3645

```solidity
File: FraxEtherRedemptionQueue.sol
343:     function enterRedemptionQueue(address _recipient, uint120 _amountToRedeem) public nonReentrant {
344:         // Get queue information
345:         RedemptionQueueState memory _redemptionQueueState = redemptionQueueState;
346:         RedemptionQueueAccounting memory _redemptionQueueAccounting = redemptionQueueAccounting;
347: 
348:         // Calculations: redemption fee
349:         uint120 _redemptionFeeAmount = ((uint256(_amountToRedeem) * _redemptionQueueState.redemptionFee) /
350:             FEE_PRECISION).toUint120();
351: 
352:         // Calculations: amount of ETH owed to the user
353:         uint120 _amountEtherOwedToUser = _amountToRedeem - _redemptionFeeAmount;
354: 
355:         // Calculations: increment ether liabilities by the amount of ether owed to the user
356:         _redemptionQueueAccounting.etherLiabilities += uint128(_amountEtherOwedToUser);
357: 
358:         // Calculations: increment unclaimed fees by the redemption fee taken
359:         _redemptionQueueAccounting.unclaimedFees += _redemptionFeeAmount;
360: 
361:         // Calculations: maturity timestamp
362:         uint64 _maturityTimestamp = uint64(block.timestamp) + _redemptionQueueState.queueLengthSecs;
363: 
364:         // Effects: Initialize the redemption ticket NFT information
365:         nftInformation[_redemptionQueueState.nextNftId] = RedemptionQueueItem({
366:             amount: _amountEtherOwedToUser,
367:             maturity: _maturityTimestamp,
368:             hasBeenRedeemed: false,
369:             earlyExitFee: _redemptionQueueState.earlyExitFee
370:         });
```

## Impact

Users unable to withdraw their assets. Loss of assets for the victim.

## Code Snippet

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3413

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3645

## Tool used

Manual Review

## Recommendation

Ensure that the protocol team and its users are aware of the risks of such an event and develop a contingency plan to manage it.