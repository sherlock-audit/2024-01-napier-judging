Flat Iris Huskie

high

# Users are not receiving full yield when collect on sunny days.

## Summary
Some users will receive more target tokens on sunny days than they are supposed to. 
## Vulnerability Detail
Let's look at how computation is going on after maturity
```solidity
    function collect() public nonReentrant whenNotPaused returns (uint256) {
        uint256 _lscale = lscales[msg.sender];
        uint256 accruedInTarget = unclaimedYields[msg.sender];

        if (_lscale == 0) revert NoAccruedYield();

        GlobalScales memory _gscales = gscales;
        _updateGlobalScalesCache(_gscales);

        uint256 yBal = _yt.balanceOf(msg.sender);
        accruedInTarget += _computeAccruedInterestInTarget(_gscales.maxscale, _lscale, yBal);
...

        if (block.timestamp >= maturity) {
            accruedInTarget += _computeTargetBelongsToYT(_gscales, yBal);
            _yt.burn(msg.sender, yBal);
        }
```
[napier-v1/src/Tranche.sol#L417](https://github.com/sherlock-audit/2024-01-napier/blob/3ba0b38b63a2658bcb1596a1f0fee13c46176301/napier-v1/src/Tranche.sol#L417)

`_computeTargetBelongsToYT` takes `_yt.balanceOf(msg.sender)` as a yield token balance of a user, but whenever user calls `issue(address(user), 0)` before maturity all of his `unclaimedYields + _computeAccruedInterestInTarget(_gscales.maxscale, _lscale, yBal)` will convert to yield tokens minus fees, so in `collect` function he will receive more.

```solidity
    function issue(
        address to,
        uint256 underlyingAmount
    ) external nonReentrant whenNotPaused notExpired returns (uint256 issued) {
        uint256 _lscale = lscales[to];
        uint256 accruedInTarget = unclaimedYields[to];
...
        if (_lscale != 0) {
            accruedInTarget += _computeAccruedInterestInTarget(_maxscale, _lscale, yBal);
        }
...
        uint256 sharesUsed = sharesMinted + accruedInTarget;
        uint256 fee = sharesUsed.mulDivUp(issuanceFeeBps, MAX_BPS);
        issued = (sharesUsed - fee).mulWadDown(_maxscale);

        _yt.mint(to, issued);

        emit Issue(msg.sender, to, issued, sharesUsed);
    }

```

## Impact
I think users will receive less than they should after maturity in some cases
## Code Snippet

## Tool used

Manual Review

## Recommendation
Add the accrued amount of yield tokens and unclaimed to the sunny formula calculation similar to this but take into account scale at maturity time
```diff
    function collect() public nonReentrant whenNotPaused returns (uint256) {
        uint256 _lscale = lscales[msg.sender];
        uint256 accruedInTarget = unclaimedYields[msg.sender];

        if (_lscale == 0) revert NoAccruedYield();

        GlobalScales memory _gscales = gscales;
        _updateGlobalScalesCache(_gscales);

        uint256 yBal = _yt.balanceOf(msg.sender);
        accruedInTarget += _computeAccruedInterestInTarget(_gscales.maxscale, _lscale, yBal);
        lscales[msg.sender] = _gscales.maxscale;
        delete unclaimedYields[msg.sender];
        gscales = _gscales;

        if (block.timestamp >= maturity) {
            // If matured, burn YT and add the principal portion to the accrued yield
-            accruedInTarget += _computeTargetBelongsToYT(_gscales, yBal);
+            uint256 fee = accruedInTarget.mulDivUp(issuanceFeeBps, MAX_BPS);
+            uint issued = (accruedInTarget - fee).mulWadDown(_gscales.maxscale);
+            accruedInTarget += _computeTargetBelongsToYT(_gscales, yBal + issued);
+            issuanceFees += fee;
            _yt.burn(msg.sender, yBal);
        }

        // Convert the accrued yield in Target token to underlying token and transfer it to the `msg.sender`
        // Target token may revert if zero-amount transfer is not allowed.
        _target.safeTransfer(address(adapter), accruedInTarget);
        (uint256 accrued, ) = adapter.prefundedRedeem(msg.sender);
        emit Collect(msg.sender, accruedInTarget);
        return accrued;
    }
```
