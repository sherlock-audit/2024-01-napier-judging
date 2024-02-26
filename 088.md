Noisy Coal Python

high

# Faster redeemer gets more assets

## Summary

Faster redeemers get more assets than those that redeem later, resulting in a loss of assets for those that redeem later and creating unfairness within the system.

## Vulnerability Detail

Per the [Notion Page](https://hackmd.io/W2mPhP7YRjGxqnAc93omLg#FAQ) shared in the Contest Page: 

> 1. In real case, why $S(t_m)$ instead of $s(t_m)$?
>    A. we use the max scale because it would be unfair to some YT holders if we used the current scale ~ If we did that, the YT holders that were fastest to call collect would get more yield out than those that waited, assuming the scale goes down at any point in time.

It was understood that the protocol intended to prevent a scenario where the faster YT redeemer gets more yield than those that redeem later if the scale goes down to ensure fairness in the system by using the max scale when computing the amount of interest accrued for a user.

However, in the current Tranche + Adaptor design within the protocol, it was observed that using only the max scale will not achieve that.

The following example shows that faster redeemers can get more yield than those who redeem later if the scale goes down.

Assume that Alice and Bob both own 50 YT each, and their $lscale$ is 1.0

At $T0$​, the scale of the adaptor is 1.25 and the current max scale is 1.25, and the state is as follows:

```solidity
scale = totalAssets() / totalSupply()
scale = (withdrawalQueueEth + bufferEth + stEthBalance) / totalSupply()
scale = (0 + 50 ETH + 75 ETH) / 100 shares
scale = 125 ETH / 100 shares
scale = 1.25
```

Alice collects the yield for her 50 YT, and she will receive 10 estETH (Target Tokens)

$$
\begin{aligned}
targetAmt = \frac{50\ YT}{lscale} - \frac{50\ YT}{maxScale} \\
targetAmt = \frac{50\ YT}{1.0} - \frac{50\ YT}{1.25} \\
targetAmt = 50 - 40 = 10\ estETH
\end{aligned}
$$

The 10 estETH will be forwarded to the adaptor to convert them to underlying assets (ETH) and transfer them to Alice. The amount of underlying assets one will receive is based on the current scale of the adaptor. At this point, the current scale is 1.25, and Alice will receive 12.5 ETH

```solidity
10 * 1.25 = 12.5 ETH
```

After Alice collected the yield, the state of the adaptor was as follows:

The total ETH will be deducted by 12.5 ETH sent to Alice earlier, and the total supply will be deducted by 10 estETH burned during the redemption earlier.

```solidity
Before collection:
scale = (0 + 50 ETH + 75 ETH) / 100 shares = 1.25

After collection:
scale = (0 + 37.5 ETH + 75 ETH) / 90 shares = 1.25
scale = 112.5 ETH / 90 shares = 1.25
```

At $T1$​, the value of the stETH drops (e.g., because of mass slashing) from 75 stETH to 60 stETH.

```solidity
scale = totalAssets() / totalSupply()
scale = (withdrawalQueueEth + bufferEth + stEthBalance) / totalSupply()
scale = (0 + 37.5 ETH + 60 ETH) / 90 shares
scale =  97.5 ETH / 90 shares
scale = 1.0833
```

Bob collects the yield for his 50 YT, and he will receive 10 estETH (Target Tokens)

$$
\begin{aligned}
targetAmt = \frac{50\ YT}{lscale} - \frac{50\ YT}{maxScale} \\
targetAmt = \frac{50\ YT}{1.0} - \frac{50\ YT}{1.25} \\
targetAmt = 50 - 40 = 10\ estETH
\end{aligned}
$$

The 10 estETH will be forwarded to the adaptor to convert them to underlying assets (ETH) and transfer them to Bob. The amount of underlying assets one will receive is based on the current scale of the adaptor. At this point, the current scale is 1.083, and Bob will receive 10.833 ETH

```solidity
10 * 1.0833 = 10.833 ETH
```

In the above example, Alice gets 12.5 ETH, while Bob gets 10.833 ETH. This shows that the faster redeemer (Alice) can get more yield than those who redeem later (Bob) if the scale goes down. 

## Impact

Faster redeemers get more assets than those that redeem later, resulting in a loss of assets for those that redeem later and creating unfairness within the system.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L399

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L704

## Tool used

Manual Review

## Recommendation

The root cause of this issue is that while the `maxScale` and `mscale` are "frozen," the current scale of the adaptor is still floating according to the market condition. Thus, when converting the target token to underlying assets, the conversion is still subjected to variation of the current scale in the adaptor.

This was not an issue for Sense Finance as they forwarded the Target Token (e.g. wstETH) directly to the users without converting them to underlying assets, thus avoiding this issue. However, this is not possible for Napier as its target token is estETH adaptor token, which is not usable for the users in the market if they are not unwrapped.

A more sophisticated settlement mechanism should be implemented to prevent the timing of redemption from affecting the number of assets the users will receive after maturity. For instance, after maturity, a settlement process should be initiated to redeem/claim all the assets (e.g., stETH) invested in the external market (e.g., LIDO). This can be done by a bot that initiates this process automatically on or after maturity. After all the assets have been redeemed or claimed, lock the final scale that will be used for every redeemer. Users can only start redeeming against the same final scale after the settlement. This will ensure fairness for everyone as the number of assets received during redemption will no longer be time-dependent.