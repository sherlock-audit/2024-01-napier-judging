Noisy Coal Python

medium

# `withdraw` function does not comply with ERC5095

## Summary

The `withdraw` function of Tranche/PT does not comply with ERC5095 as it does not return the exact amount of underlying assets requested by the users.

## Vulnerability Detail

Per the contest's [README](https://github.com/sherlock-audit/2024-01-napier/tree/main?tab=readme-ov-file#q-is-the-codecontract-expected-to-comply-with-any-eips-are-there-specific-assumptions-around-adhering-to-those-eips-that-watsons-should-be-aware-of) page, it stated that the code is expected to comply with ERC5095 (https://eips.ethereum.org/EIPS/eip-5095). As such, any non-compliance to ERC5095 found during the contest is considered valid.

> Q: Is the code/contract expected to comply with any EIPs? Are there specific assumptions around adhering to those EIPs that Watsons should be aware of?
> EIP20 and IERC5095

Following is the specification of the `withdraw` function of ERC5095. It stated that the user must receive exactly `underlyingAmount` of underlying tokens.

> withdraw
> Burns principalAmount from holder and sends exactly underlyingAmount of underlying tokens to receiver.

However, the `withdraw` function does not comply with this requirement.

On a high-level, the reason is that Line 337 will compute the number of shares that need to be redeemed to receive `underlyingAmount` number of underlying tokens from the adaptor. The main problem here is that the division done here is rounded down. Thus, the `sharesRedeem` will be lower than expected. Consequently, when `sharesRedeem` number of shares are redeemed at Line 346 below, the users will not receive an exact number of `underlyingAmount` of underlying tokens.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L328

```solidity
File: Tranche.sol
328:     function withdraw(
329:         uint256 underlyingAmount,
330:         address to,
331:         address from
332:     ) external override nonReentrant expired returns (uint256) {
333:         GlobalScales memory _gscales = gscales;
334:         uint256 cscale = _updateGlobalScalesCache(_gscales);
335: 
336:         // Compute the shares to be redeemed
337:         uint256 sharesRedeem = underlyingAmount.divWadDown(cscale);
338:         uint256 principalAmount = _computePrincipalTokenRedeemed(_gscales, sharesRedeem);
339: 
340:         // Update the global scales
341:         gscales = _gscales;
342:         // Burn PT tokens from `from`
343:         _burnFrom(from, principalAmount);
344:         // Withdraw underlying tokens from the adapter and transfer them to `to`
345:         _target.safeTransfer(address(adapter), sharesRedeem);
346:         (uint256 underlyingWithdrawn, ) = adapter.prefundedRedeem(to);
```

## Impact

The tranche/PT does not align with the ERC5095 specification.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L328

## Tool used

Manual Review

## Recommendation

Update the `withdraw` function to send exactly `underlyingAmount` number of underlying tokens to the caller so that the Tranche will be aligned with the ERC5095 specification.