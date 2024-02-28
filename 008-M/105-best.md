Noisy Coal Python

medium

# Unable to deposit to Tranche/Adaptor under certain conditions

## Summary

Minting of PT and YT is the core feature of the protocol. Without the ability to mint PT and YT, the protocol would not operate. 

The user cannot deposit into the Tranche to issue new PT + YT under certain conditions.

## Vulnerability Detail

The comment in Line 133 below mentioned that the `stakeAmount` can be zero. 

The reason is that when `targetBufferEth < (availableEth + queueEthCache)`,  it is possible that there is a pending withdrawal request (`queueEthCache`) and no available ETH left in the buffer (`availableEth = 0`). Refer to the comment in Line 123 below.

As a result, the code at Line 127 below will restrict the amount of ETH to be staked and set the `stakeAmount` to zero.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L133

```solidity
File: BaseLSTAdapter.sol
071:     function prefundedDeposit() external nonReentrant returns (uint256, uint256) {
..SNIP..
113:         uint256 stakeAmount;
114:         unchecked {
115:             stakeAmount = availableEth + queueEthCache - targetBufferEth; // non-zero, no underflow
116:         }
117:         // If the stake amount exceeds 95% of the available ETH, cap the stake amount.
118:         // This is to prevent the buffer from being completely drained. This is not a complete solution.
119:         //
120:         // The condition: stakeAmount > availableEth, is equivalent to: queueEthCache > targetBufferEth
121:         // Possible scenarios:
122:         // - Target buffer percentage was changed to a lower value and there is a large withdrawal request pending.
123:         // - There is a pending withdrawal request and the available ETH are not left in the buffer.
124:         // - There is no pending withdrawal request and the available ETH are not left in the buffer.
125:         uint256 maxStakeAmount = (availableEth * 95) / 100;
126:         if (stakeAmount > maxStakeAmount) {
127:             stakeAmount = maxStakeAmount; // max 95% of the available ETH
128:         }
129: 
130:         /// INTERACT ///
131:         // Deposit into the yield source
132:         // Actual amount of ETH spent may be less than the requested amount.
133:         stakeAmount = _stake(stakeAmount); // stake amount can be 0
```

However, the issue is that when `_stake` function is called with `stakeAmount` set to zero, it will result in zero ETH being staked and Line 77 below will revert.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/lido/StEtherAdapter.sol#L77

```solidity
File: StEtherAdapter.sol
64:     /// @inheritdoc BaseLSTAdapter
65:     /// @dev Lido has a limit on the amount of ETH that can be staked.
66:     /// @dev Need to check the current staking limit before staking to prevent DoS.
67:     function _stake(uint256 stakeAmount) internal override returns (uint256) {
68:         uint256 stakeLimit = STETH.getCurrentStakeLimit();
69:         if (stakeAmount > stakeLimit) {
70:             // Cap stake amount
71:             stakeAmount = stakeLimit;
72:         }
73: 
74:         IWETH9(Constants.WETH).withdraw(stakeAmount);
75:         uint256 _stETHAmt = STETH.submit{value: stakeAmount}(address(this));
76: 
77:         if (_stETHAmt == 0) revert InvariantViolation();
78:         return stakeAmount;
79:     }
```

A similar issue also occurs for the sFRXETH adaptor. If `FRXETH_MINTER.submit` function is called with `stakeAmount == 0`, it will revert.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/frax/SFrxETHAdapter.sol#L76

```solidity
File: SFrxETHAdapter.sol
71:     /// @notice Mint sfrxETH using WETH
72:     function _stake(uint256 stakeAmount) internal override returns (uint256) {
73:         IWETH9(Constants.WETH).withdraw(stakeAmount);
74:         FRXETH_MINTER.submit{value: stakeAmount}();
75:         uint256 received = STAKED_FRXETH.deposit(stakeAmount, address(this));
76:         if (received == 0) revert InvariantViolation();
77: 
78:         return stakeAmount;
79:     }
```

The following shows that the `FRXETH_MINTER.submit` function will revert if submitted ETH is zero below.

https://etherscan.io/address/0xbAFA44EFE7901E04E39Dad13167D089C559c1138#code#F1#L89

```solidity
/// @notice Mint frxETH to the recipient using sender's funds. Internal portion
function _submit(address recipient) internal nonReentrant {
    // Initial pause and value checks
    require(!submitPaused, "Submit is paused");
    require(msg.value != 0, "Cannot submit 0");
```

## Impact

Minting of PT and YT is the core feature of the protocol. Without the ability to mint PT and YT, the protocol would not operate. The user cannot deposit into the Tranche to issue new PT + YT under certain conditions. Breaking of core protocol/contract functionality.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L133

## Tool used

Manual Review

## Recommendation

Short-circuit the `_stake` function by returning zero value immediately if the `stakeAmount` is zero.

File: StEtherAdapter.sol

```diff
function _stake(uint256 stakeAmount) internal override returns (uint256) {
+	if (stakeAmount == 0) return 0;	
	uint256 stakeLimit = STETH.getCurrentStakeLimit();
	if (stakeAmount > stakeLimit) {
		// Cap stake amount
		stakeAmount = stakeLimit;
	}

	IWETH9(Constants.WETH).withdraw(stakeAmount);
	uint256 _stETHAmt = STETH.submit{value: stakeAmount}(address(this));

	if (_stETHAmt == 0) revert InvariantViolation();
	return stakeAmount;
}
```

File: SFrxETHAdapter.sol

```diff
function _stake(uint256 stakeAmount) internal override returns (uint256) {
+	if (stakeAmount == 0) return 0;	
	IWETH9(Constants.WETH).withdraw(stakeAmount);
	FRXETH_MINTER.submit{value: stakeAmount}();
	uint256 received = STAKED_FRXETH.deposit(stakeAmount, address(this));
	if (received == 0) revert InvariantViolation();

	return stakeAmount;
}
```