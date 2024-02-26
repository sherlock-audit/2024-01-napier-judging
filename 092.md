Noisy Coal Python

high

# YT holders cannot receive a portion of the principal allocated by the PT holders due to the manipulation

## Summary

YT holders cannot receive a portion of the principal allocated by the PT holders due to the manipulation, leading to a loss of assets for the victims.

## Vulnerability Detail

On a sunny day, the YT holders can gain/earn more as they can receive a portion of the principal allocated by the PT holders. On the other hand, the PT holders will lose a portion of their principal during a sunny day.

Malicious PT holders can ensure that the Tranche is always in a "not sunny day" state upon maturity by performing the following manipulation:

1) When the Tranche and Adaptor are deployed, immediately mint the smallest possible number of shares. Attackers can do so by calling the Adaptor's `prefundedDeposit` function directly if they want to avoid issuance fees OR call the `Tranche.issue` function
2) Transfer large amounts of assets to the Adaptor directly. With a small amount of total supply and a large amount of total assets, the Adaptor's scale will be extremely large. Let the large scale at this point be $L$.
3) Trigger any function that will trigger an update to the global scale. The max scale will be updated to $L$ and locked at this large scale throughout the entire lifecycle of the Tranche, as it is not possible for the scale to exceed $L$ based on the current market yield or condition.
4) Attacker withdraws all their shares and assets via Adaptor's `prefundedDeposit` function or Tranche's redeem functions. Note that the attacker will not incur any fees during withdrawal. Thus, this attack is economically cheap and easy to execute.
5) After the attacker's withdrawal, the current scale of the adapter will revert back to normal (e.g. 1.0 exchange rate), and the scale will only increase due to the yield from staked ETH (e.g. daily rebase) and increase progressively (e.g., 1.0 > 1.05 > 1.10 > 1.15)

The conditions for a sunny day are as follows, where $S(t_m)$ is the max scale and $s(t_m)$ is the maturity/current scale.

$$
\frac{s\left(t_m\right)}{S\left(t_m\right)} \geqslant 1-\theta
$$

Since the denominator ( $S(t_m)$ ) is an extremely large value ($L$), the $\frac{s\left(t_m\right)}{S\left(t_m\right)}$ component in the formula will be extremely small and always smaller than $1-\theta$. Thus, the Tranche is always in a "not sunny day" state.

As a result, PT holders will prevent losing $\theta$ amount of principle to YT holders.

## Impact

Loss of assets for the YT holders as they cannot receive a portion of the principal allocated by the PT holders due to the manipulation.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L42

## Tool used

Manual Review

## Recommendation

Ensure that measures are in place to prevent malicious actors from manipulating the scale of the adaptor, as they impact many calculations that depend on max or maturity scales. 