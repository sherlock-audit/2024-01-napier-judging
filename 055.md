Cold Slate Copperhead

high

# Missing stETH conversion

## Summary

`totalAssets()` in ETH staking adapters should convert to the `ETH`. However this isn't done in for the Lido Eth adapter.

## Vulnerability Detail

The `StEthAdapater.sol`, `totalAssets` is denominated in `ETH`. However, the function is missing the step where the STETH balance is converted to ETH. We can look at the `SFrxETHAdapter` for an example of the correct conversion.

Wrong logic in `StEthAdapter.sol` - `STETH.balanceOf()` is not converted to ETH value.

```solidity
    function totalAssets() public view override returns (uint256) {
        uint256 stEthBalance = STETH.balanceOf(address(this));
        return withdrawalQueueEth + bufferEth + stEthBalance;
    }
```

Correct logic in `SFrxETHAdapter.sol` - note the line `uint256 balanceInFrxEth = STAKED_FRXETH.convertToAssets(balance);` converts the `staked_frxEth` to a value in `frxEth`

```solidity
    function totalAssets() public view override returns (uint256) {
        uint256 balance = STAKED_FRXETH.balanceOf(address(this));
        uint256 balanceInFrxEth = STAKED_FRXETH.convertToAssets(balance);
        return withdrawalQueueEth + bufferEth + balanceInFrxEth; // 1 frxEth = 1 ETH
    }
```

## Impact

Wrong value is always returned for totalAssets and thereforce scale of the Lido Frax adapter. This breaks the entire accounting logic of the Tranches.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L232-L234

## Tool used

Manual Review

## Recommendation

Convert the stEth balance to the corresponding Eth value.
