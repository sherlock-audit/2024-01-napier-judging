Cool Beige Bat

high

# Missing compute target belongs to YT in Tranche::redeemWithYT()

## Summary
Function redeemWithYT() will redeem underlying tokens and burn the related PT and YT. YT should earn some profits in some cases, which is missed in redeemWithYT()

## Vulnerability Detail
In current implementation, there will be some amount of principal reserved for YT holders if the maturity has passed. We can see that more clearly in function collect():
```solidity
function collect() public nonReentrant whenNotPaused returns (uint256) {
        uint256 _lscale = lscales[msg.sender];
        uint256 accruedInTarget = unclaimedYields[msg.sender];
        ...

        if (block.timestamp >= maturity) {
            // If matured, burn YT and add the principal portion to the accrued yield
            accruedInTarget += _computeTargetBelongsToYT(_gscales, yBal);
            _yt.burn(msg.sender, yBal);
        }
        ...
        return accrued;
    }
```

However, in redeemWithYT(), imagine that the maturity has passed and users want to redeem underlying via redeemWithYT(), all PT and YT will be burned and related underlying tokens will be returned back to users except principal reserved for YT holders.
```solidity
function redeemWithYT(address from, address to, uint256 pyAmount) external nonReentrant returns (uint256) {
        uint256 _lscale = lscales[from];
        uint256 accruedInTarget = unclaimedYields[from];
        ...
        // Compute the accrued yield from the time when the YT balance is updated last to now
        // The accrued yield in units of target is computed as:
        // Formula: yield = ytBalance * (1/lscale - 1/maxscale)
        // Sum up the accrued yield, plus the unclaimed yield from the last time to now
        accruedInTarget += _computeAccruedInterestInTarget(
            _gscales.maxscale,
            _lscale,
            // Use yt balance instead of `pyAmount`
            // because we'll update the user's lscale to the current maxscale after this line
            // regardless of whether the user redeems all of their yt or not.
            // Otherwise, the user will lose some accrued yield from the last time to now.
            _yt.balanceOf(from)
        );
        // Compute shares equivalent to the amount of principal token to redeem
        uint256 sharesRedeemed = pyAmount.divWadDown(_gscales.maxscale);
        ...
        // Burn PT and YT tokens from `from`
        _burnFrom(from, pyAmount);
        
        @ YT token just burn without check possible rewards==> _yt.burnFrom(from, msg.sender, pyAmount);

        // Withdraw underlying tokens from the adapter and transfer them to the user
        _target.safeTransfer(address(adapter), sharesRedeemed + accruedInTarget);
        (uint256 amountWithdrawn, ) = adapter.prefundedRedeem(to);

        emit RedeemWithYT(from, to, amountWithdrawn);
        return amountWithdrawn;
    }

```
 

## Impact
Users might lose some profits belongs to Yield tokens.

## Code Snippet
https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L246-L280
https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L399-L418

## Tool used
Manual Review

## Recommendation
Refer to collect(), add similar implementation
```solidity
        if (block.timestamp >= maturity) {
            // If matured, burn YT and add the principal portion to the accrued yield
            accruedInTarget += _computeTargetBelongsToYT(_gscales, yBal);
            _yt.burn(msg.sender, yBal);
        }
```