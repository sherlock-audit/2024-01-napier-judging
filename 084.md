Noisy Coal Python

high

# Lack of slippage control for `issue` function

## Summary

The lack of slippage control for `issue` function can lead to a loss of assets for the affected users.

## Vulnerability Detail

During the issuance, the user will deposit underlying assets (e.g., ETH) to the Tranche contract, and the Tranche contract will forward them to the Adaptor contract for depositing at Line 208 below. The number of shares minted is depending on the current scale of the adaptor. The current scale of the adaptor can increase or decrease at any time, depending on the current on-chain condition when the transaction is executed. For instance, the LIDO's daily oracle/rebase update will increase the stETH balance, which will, in turn, increase the adaptor's scale. On the other hand, if there is a mass validator slashing event, the ETH claimed from the withdrawal queue will be less than expected, leading to a decrease in the adaptor's scale. Thus, one cannot ensure the result from the off-chain simulation will be the same as the on-chain execution.

Having said that, the number of shared minted will vary (larger or smaller than expected) if there is a change in the current scale. Assuming that Alice determined off-chain that depositing 100 ETH would issue $x$ amount of PT/YT. When she executes the TX, the scale increases, leading to the amount of PT/YT issued being less than $x$. The slippage is more than what she can accept.

In summary, the `issue` function lacks the slippage control that allows the users to revert if the amount of PT/YT they received is less than the amount they expected.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L179

```solidity
File: Tranche.sol
179:     function issue(
180:         address to,
181:         uint256 underlyingAmount
182:     ) external nonReentrant whenNotPaused notExpired returns (uint256 issued) {
..SNIP..
206:         // Transfer underlying from user to adapter and deposit it into adapter to get target token
207:         _underlying.safeTransferFrom(msg.sender, address(adapter), underlyingAmount);
208:         (, uint256 sharesMinted) = adapter.prefundedDeposit();
209: 
210:         // Deduct the issuance fee from the amount of target token minted + reinvested yield
211:         // Fee should be rounded up towards the protocol (against the user) so that issued principal is rounded down
212:         // Hackmd: F0
213:         // ptIssued
214:         // = (u/s + y - fee) * S
215:         // = (sharesUsed - fee) * S
216:         // where u = underlyingAmount, s = current scale, y = reinvested yield, S = maxscale
217:         uint256 sharesUsed = sharesMinted + accruedInTarget;
218:         uint256 fee = sharesUsed.mulDivUp(issuanceFeeBps, MAX_BPS);
219:         issued = (sharesUsed - fee).mulWadDown(_maxscale);
220: 
221:         // Accumulate issueance fee in units of target token
222:         issuanceFees += fee;
223:         // Mint PT and YT to user
224:         _mint(to, issued);
225:         _yt.mint(to, issued);
226: 
227:         emit Issue(msg.sender, to, issued, sharesUsed);
228:     }
229: 
```

## Impact

Loss of assets for the affected users.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L179

## Tool used

Manual Review

## Recommendation

Implement a slippage control that allows the users to revert if the amount of PT/YT they received is less than the amount they expected.