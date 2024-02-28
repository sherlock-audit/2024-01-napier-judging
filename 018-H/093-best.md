Noisy Coal Python

high

# The final APR of their PT holdings will be lower than expected due to the Tranche's tilt

## Summary

The final APR of their PT holdings will be lower than expected due to the Tranche's tilt.

## Vulnerability Detail

The approach Napier users can earn a fixed yield via Naiper's PT is by buying the PT at a discount before maturity and redeeming it at face value on or after maturity. Following is one example to illustrate how Bob can earn a fixed yield of 11.11%.

```solidity
Bob Buy 1 PT (ETH) at 0.9 ETH ===== 1 Year Later ====> Bob Redeem 1 ETH 

APR = (End Value/Beginning Value - 1) * 100% = (1/0.9 - 1) * 100% = 11.11%
Bob gets a 11.11% APR 
```

The Napier AMM pool is designed to have an exchange rate between underlying assets and PT progressively converging to one as it gets closer to maturity. At maturity, the exchange rate will be one.

1 PT must be redeemed for 1 underlying asset upon maturity. Otherwise, the fundamental design of the system will be broken. 

However, the issue is that the Tranche introduced a tilt mechanism. Assume that the tilt is set to 10%.

On a sunny day, if users attempt to redeem 1 PT from the Tranche, they will not get back the face value of 1 underlying asset due to the tilt. The tilt will cause a certain percentage (in this case, 10%) of the principle to be reserved and given to the YT holder on a sunny day. Thus, the users who redeem 1 PT will only receive back 0.9 PT worth of underlying assets, not 1 PT worth of underlying assets.

Using back the same Bob example above, Bob expects to earn an APR of around 11% upon maturity. However, due to the 10% tilt, Bob will earn a much lower APR upon maturity as the tilt will cause him to lose 10% of its principal, which is not what he expected.

## Impact

Loss of assets for the PT holders. The final APR of their PT holdings will be lower than expected due to the Tranche's tilt.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L42

## Tool used

Manual Review

## Recommendation

Naiper protocol attempts to incorporate the design of two different protocols (Pendle and Sense)

The Napier AMM pool is inspired by Pendle AMM pool (with Notional V2 math). In Pendle, 1 PT can always be exchanged for 1 underlying asset upon maturity.

The Tranche's design is inspired by Sense's Divider, which introduces the concept of tilt (sunny/not sunny day). The following [spreadsheet](https://docs.google.com/spreadsheets/d/1WVUcatGfpweWH78HXKYuB1_c60o9hw7p3b4NhJVre6I/edit?usp=sharing) attempts to simulate how the tilt system works under various conditions (reference from Sense Finance's [test suite](https://github.com/sense-finance/sense-v1/blob/c5844f31589a038be4f2fcdc211b62e8d0901688/pkg/core/src/tests/Divider.t.sol#L955)), which might be helpful. The "Tilt" and the "Amt of U Alice want to redeem after maturity" fields at the top left table are editable.

```solidity
U = Underlying Assets, T = Target Tokens, "x_z" = amount of the Target Tokens PT holders get, "x_c" = the amount of Target Tokens YT holder get
```

Thus, the tilt math inspired by Sense Finance might be incompatible with the Napier AMM pool math.