Noisy Coal Python

high

# YT holder are unable to claim their interest

## Summary

A malicious user could prevent YT holders from claiming their interest, leading to a loss of assets.

## Vulnerability Detail

The interest accrued (in target token) by a user is computed based on the following formula, where $lscale$ is the user's last scale update and $maxScale$ is the max scale observed so far. The formula is taken from [here](https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L722):

$$
interestAccrued = y(\frac{1}{lscale} - \frac{1}{maxScale})
$$

A malicious user could perform the following steps to manipulate the $maxScale$ to a large value to prevent YT holders from claiming their interest.

1) When the Tranche and Adaptor are deployed, immediately mint the smallest possible number of shares. Attackers can do so by calling the Adaptor's `prefundedDeposit` function directly if they want to avoid issuance fees OR call the `Tranche.issue` function
2) Transfer large amounts of assets to the Adaptor directly. With a small amount of total supply and a large amount of total assets, the Adaptor's scale will be extremely large. Let the large scale at this point be $L$.
3) Trigger any function that will trigger an update to the global scale. The max scale will be updated to $L$ and locked at this large scale throughout the entire lifecycle of the Tranche, as it is not possible for the scale to exceed $L$ based on the current market yield or condition.
4) Attacker withdraws all their shares and assets via Adaptor's `prefundedDeposit` function or Tranche's redeem functions. Note that the attacker will not incur any fees during withdrawal. Thus, this attack is economically cheap and easy to execute.
5) After the attacker's withdrawal, the current scale of the adapter will revert back to normal (e.g. 1.0 exchange rate), and the scale will only increase due to the yield from staked ETH (e.g. daily rebase) and increase progressively (e.g., 1.0 > 1.05 > 1.10 > 1.15)

When a user issues/mints new PT and YT, their `lscales[user]` will be set to the $maxScale$, which is $L$. 

A user's `lscales[user]` will only be updated if the adaptor's current scale is larger than $L$. As stated earlier, it is not possible for the scale to exceed $L$ based on the current market yield or condition. Thus, the user's `lscales[user]` will be stuck at $L$ throughout the entire lifecycle of the Tranche.

As a result, there is no way for the YT holder to claim their interest because, since $lscale == maxScale$, the interest accrued computed from the formula will always be zero. Thus, even if the Tranche/Adaptor started gaining yield from staked ETH, there is no way for YT holders to claim that.

## Impact

YT holders cannot claim their interest, leading to a loss of assets.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L704

## Tool used

Manual Review

## Recommendation

Ensure that the YT holders can continue to claim their accrued interest earned within the Tranche/Adaptor regardless of the max scale.