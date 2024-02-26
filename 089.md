Noisy Coal Python

high

# Withdrawal can be blocked

## Summary

Malicious users can block withdrawal, preventing protocol users from withdrawing their funds.

## Vulnerability Detail

When the adaptor does not have sufficient buffer ETH, users cannot redeem their PT tokens from the Tranche. It will need to wait for the buffer to be refilled.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L156

```solidity
File: BaseLSTAdapter.sol
146:     function prefundedRedeem(address recipient) external virtual returns (uint256, uint256) {
147:         uint256 shares = balanceOf(address(this));
148:         uint256 assets = previewRedeem(shares);
149: 
150:         if (shares == 0) return (0, 0);
151:         if (assets == 0) revert ZeroAssets();
152: 
153:         uint256 bufferEthCache = bufferEth;
154:         // If the buffer is insufficient, shares cannot be redeemed immediately
155:         // Need to wait for the withdrawal to be completed and the buffer to be refilled.
156:         if (assets > bufferEthCache) revert InsufficientBuffer();
```

Let's assume the following:

- `targetBufferPercentage` is set to 10%
- current scale is 1.0
- total assets are 90 ETH (9 ETH buffer + 81 staked ETH)
- total supply = 90 shares

Bob (malicious user) could use the following formula to solve for "Deposit" variable OR perform off-chain simulation to determine the right number of assets to deposit for the attack:

```solidity
Current TotalAsset = 90 ETH

(Current TotalAsset + Deposit) * 0.1 - Deposit = 0
(90 ETH + Deposit) * 0.1 - Deposit = 0
Deposit = 10 ETH
```

Bob deposits 10 ETH. It will become 100 ETH (10 ETH buffer + 90 staked ETH), and the total supply will increase to 100 shares. Bob will receive 10 shares in return after the deposit.

Bob redeems his 10 shares, and the adaptor will send him 10 ETH, which depletes the entire amount of ETH buffer within the adaptor. Since there is no fee charged during deposit and withdraw on the adaptor, Bob will not lose any of the initial 10 ETH.

```solidity
assets = 10 shares * current scale = 10 ETH
```

Since malicious Bob has depleted the ETH buffer, the rest of the Napier users cannot withdraw.

Bob will perform the above attacks within a single transaction to DOS Napier users, preventing them from withdrawing. The users can only withdraw when the rebalancer bot unstake the staked ETH to replenish the buffer. However, the issue is that in the worst-case scenario, it will take 5 days for the redemption to be processed before the adaptor can start claiming the ETH from LIDO.

Following is the withdrawal period (waiting time) for unstaking ETH:

- LIDO: Between 1-5 days. Note that if more people exit the staking queue in Ethereum, the waiting period will increase further.
- FRAX: The sum total of both entry and exit queues ([Reference](https://docs.frax.finance/frax-ether/redemption#frxeth-redemption-queue))

Note that the deposits by other users will not mitigate this issue due to the following reasons:

- After maturity, no one can issue/mint PT+YT anymore from Tranche, resulting in no new ETH flowing into the adaptor. Yet, the attacker can directly call the Adaptor's `prefundedDeposit` and `prefundedRedeem` functions to carry out this attack as they are still accessible after maturity.
- As time moves closer to maturity, there will be fewer deposits naturally.

## Impact

Users are unable to withdraw. This attack is cheap to execute (gas fee would be a few bucks), but the negative consequence to the protocol is significant (e.g., block withdrawal for 5 days). To DOS Napier for a month, one could execute the attack 6 times every 5 days, and the total costs are still cheap for the attacker, short traders, or competitors (other protocols).

Marking this as a High issue as the impact is High and probability is High (due to ease of execution and cheap to execute)

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L156

## Tool used

Manual Review

## Recommendation

Consider any of the following measures to mitigate the issues:

- Charge fee upon depositing and withdrawal
- Restrict the amount of withdrawal allowed based on the existing ETH buffer remaining on the adaptor. For instance, using the same example, if Bob has 10 shares, totalSupply is 100 shares and the current ETH buffer is 10 ETH, he should only be allowed to withdraw 1 ETH at the maximum, which is 10% of the current ETH buffer based on his portion of shares in the adaptor.
- Restrict access to `prefundedDeposit` and `prefundedRedeem` function only to the Tranche. This does not entirely prevent this attack, but it makes the attack more expensive due to the issuance fee charged during the deposit when performed via Tranche.