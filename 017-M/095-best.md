Noisy Coal Python

medium

# Users unable to withdraw their funds due to FRAX admin action

## Summary

FRAX admin action can lead to the fund of Naiper protocol and its users being stuck, resulting in users being unable to withdraw their assets.

## Vulnerability Detail

Per the contest page, the admins of the protocols that Napier integrates with are considered "RESTRICTED". This means that any issue related to FRAX's admin action that could negatively affect Napier protocol/users will be considered valid in this audit contest.

> Q: Are the admins of the protocols your contracts integrate with (if any) TRUSTED or RESTRICTED?
> RESTRICTED

When the Adaptor needs to unstake its staked ETH to replenish its ETH buffer so that users can redeem/withdraw their funds, it will first join the FRAX's redemption queue, and the queue will issue a redemption NFT afterward. After a certain period, the adaptor can claim their ETH by burning the redemption NFT at Line 65 via the `burnRedemptionTicketNft` function.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/frax/SFrxETHAdapter.sol#L65

```solidity
File: SFrxETHAdapter.sol
53:     function claimWithdrawal() external override {
54:         uint256 _requestId = requestId;
55:         uint256 _withdrawalQueueEth = withdrawalQueueEth;
56:         if (_requestId == 0) revert NoPendingWithdrawal();
57: 
58:         /// WRITE ///
59:         delete withdrawalQueueEth;
60:         delete requestId;
61:         bufferEth += _withdrawalQueueEth.toUint128();
62: 
63:         /// INTERACT ///
64:         uint256 balanceBefore = address(this).balance;
65:         REDEMPTION_QUEUE.burnRedemptionTicketNft(_requestId, payable(this));
66:         if (address(this).balance < balanceBefore + _withdrawalQueueEth) revert InvariantViolation();
67: 
68:         IWETH9(Constants.WETH).deposit{value: _withdrawalQueueEth}();
69:     }
```

However, it is possible for FRAX's admin to disrupt the redemption process of the adaptor, resulting in Napier users being unable to withdraw their funds. When the `burnRedemptionTicketNft` function is executed, the redemption NFT will be burned, and native ETH residing in the `FraxEtherRedemptionQueue` contract will be sent to the adaptor at Line 498 below

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3761

```solidity
File: FraxEtherRedemptionQueue.sol
473:     function burnRedemptionTicketNft(uint256 _nftId, address payable _recipient) external nonReentrant {
..SNIP..
494:         // Effects: Burn frxEth to match the amount of ether sent to user 1:1
495:         FRX_ETH.burn(_redemptionQueueItem.amount);
496: 
497:         // Interactions: Transfer ETH to recipient, minus the fee
498:         (bool _success, ) = _recipient.call{ value: _redemptionQueueItem.amount }("");
499:         if (!_success) revert InvalidEthTransfer();
```

FRAX admin could execute the `recoverEther` function to transfer out all the Native ETH residing in the `FraxEtherRedemptionQueue` contract, resulting in the NFT redemption failing due to lack of ETH. 

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3381

```solidity
File: FraxEtherRedemptionQueue.sol
185:     /// @notice Recover ETH from exits where people early exited their NFT for frxETH, or when someone mistakenly directly sends ETH here
186:     /// @param _amount Amount of ETH to recover
187:     function recoverEther(uint256 _amount) external {
188:         _requireSenderIsTimelock();
189: 
190:         (bool _success, ) = address(msg.sender).call{ value: _amount }("");
191:         if (!_success) revert InvalidEthTransfer();
192: 
193:         emit RecoverEther({ recipient: msg.sender, amount: _amount });
194:     }
```

As a result, Napier users will not be able to withdraw their funds.

## Impact

The fund of Naiper protocol and its users will be stuck, resulting in users being unable to withdraw their assets.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/frax/SFrxETHAdapter.sol#L65

## Tool used

Manual Review

## Recommendation

Ensure that the protocol team and its users are aware of the risks of such an event and develop a contingency plan to manage it.