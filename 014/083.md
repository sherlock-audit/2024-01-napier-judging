Noisy Coal Python

high

# Anyone can convert someone's unclaimed yield to PT + YT

## Summary

Anyone can convert someone's unclaimed yield to PT + YT, leading to a loss of assets for the victim.

## Vulnerability Detail

Assume that Alice has accumulated 100 Target Tokens in her account's unclaimed yields. She is only interested in holding the Target token (e.g., she is might be long Target token). She intends to collect those Target Tokens sometime later.

Bob could disrupt Alice's plan by calling `issue` function with the parameter (`to=Alice, underlyingAmount=0`). The function will convert all 100 Target Tokens stored within Alice's account's unclaimed yield to PT + YT and send them to her, which Alice does not want or need in the first place.

Line 196 below will clear Alice's entire unclaimed yield before computing the accrued interest at Line 203. The accrued interest will be used to mint the PT and YT (Refer to Line 217 and Line 224 below).

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L179

```solidity
File: Tranche.sol
179:     function issue(
180:         address to,
181:         uint256 underlyingAmount
182:     ) external nonReentrant whenNotPaused notExpired returns (uint256 issued) {
..SNIP..
196:         lscales[to] = _maxscale;
197:         delete unclaimedYields[to];
198: 
199:         uint256 yBal = _yt.balanceOf(to);
200:         // If recipient has unclaimed interest, claim it and then reinvest it to issue more PT and YT.
201:         // Reminder: lscale is the last scale when the YT balance of the user was updated.
202:         if (_lscale != 0) {
203:             accruedInTarget += _computeAccruedInterestInTarget(_maxscale, _lscale, yBal);
204:         }
..SNIP..
217:         uint256 sharesUsed = sharesMinted + accruedInTarget;
218:         uint256 fee = sharesUsed.mulDivUp(issuanceFeeBps, MAX_BPS);
219:         issued = (sharesUsed - fee).mulWadDown(_maxscale);
220: 
221:         // Accumulate issueance fee in units of target token
222:         issuanceFees += fee;
223:         // Mint PT and YT to user
224:         _mint(to, issued);
225:         _yt.mint(to, issued);
```

The market value of the PT + YT might be lower than the market value of the Target Token. In this case, Alice will lose due to Bob's malicious action.

Another issue is that when Bob calls `issue` function on behalf of Alice's account, the unclaimed target tokens will be subjected to the issuance fee (See Line 218 above). Thus, even if the market value of PT + YT is exactly the same as that of the Target Token, Alice is still guaranteed to suffer a loss from Bob's malicious action.

If Alice had collected the unclaimed yield via the collect function, she would have received the total value of the yield in the underlying asset terms, as a collection of yield is not subjected to any fee.

## Impact

Loss of assets for the victim.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L179

## Tool used

Manual Review

## Recommendation

Consider not allowing anyone to issue PY+YT on behalf of someone's account.