Creamy Tawny Fly

medium

# Missing whenNotPaused modifiers in tranche `Redeemwithyt`,`redeem`, and `withdraw`

## Summary

In most cases having the ability to pause your protocol is typically used as a security measure to prevent exploiters from completely draining protocols of their tokens.
 
In the event that an attacker does exploit Napier or it's related pools for principle tokens and or yield tokens, they will be able to exchange for underlying token and exit the protocol even when the Napier team invokes their pause. The attacker could do this by using two redeem functions, `RedeemWithYt`, `redeem`, and `withdraw`.

 While specs are not included in scope it should be noted that pause modifiers were intended to be used for `RedeemWithYt` and `redeem`. See yield-stripping-system.md #64 and #79

## Vulnerability Detail

 https://github.com/sherlock-audit/2024-01-napier/blob/6313f34110b0d12677b389f0ecb3197038211e12/napier-v1/src/Tranche.sol#L246

 Attacker calls `redeemWithYT` due to missing pause modifier

 https://github.com/sherlock-audit/2024-01-napier/blob/6313f34110b0d12677b389f0ecb3197038211e12/napier-v1/src/Tranche.sol#L279-L280
 
 yt.burn() does not result in pausing, since the transfer function is not invoked. 

https://github.com/sherlock-audit/2024-01-napier/blob/6313f34110b0d12677b389f0ecb3197038211e12/napier-v1/src/Tranche.sol#L283C17-L283C29

Attacker receives available underlying token and successfully exits the protocol.

The same attack flow is used in `redeem` and `withdraw`. 

## Impact

Napier would fail to successfully freeze the exploiters funds when enabling their pause mechanics

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/6313f34110b0d12677b389f0ecb3197038211e12/napier-v1/src/Tranche.sol#L246

https://github.com/sherlock-audit/2024-01-napier/blob/6313f34110b0d12677b389f0ecb3197038211e12/napier-v1/src/Tranche.sol#L298-L302

https://github.com/sherlock-audit/2024-01-napier/blob/6313f34110b0d12677b389f0ecb3197038211e12/napier-v1/src/Tranche.sol#L328-L332

## Tool used

Manual Review

## Recommendation

The recommendation is to include whenNotPaused modifiers on `RedeemWithYt`, `redeem`, and `withdraw`.
