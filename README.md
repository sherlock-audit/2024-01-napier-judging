# Issue H-1: All yield could be drained if users set any ````> 0```` allowance to others 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/28 

## Found by 
DenTonylifer, KingNFT, cawfree
## Summary
````Tranche.redeemWithYT()```` is not well implemented, all yield could be drained if users set any ````> 0```` allowance to others.

## Vulnerability Detail
The issue arises on L283, all ````accruedInTarget```` is sent out, this will not work while users have allowances to others. Let's say, alice has ````1000 YT```` (yield token) which has generated ````100 TT```` (target token), and if she approves bob ````100 YT```` allowance, then bob should only be allowed to take the proportional target token, which is ```` 100 TT * (100 YT / 1000 YT) = 10 TT````.
```solidity
File: src\Tranche.sol
246:     function redeemWithYT(address from, address to, uint256 pyAmount) external nonReentrant returns (uint256) {
...
261:         accruedInTarget += _computeAccruedInterestInTarget(
262:             _gscales.maxscale,
263:             _lscale,
...
268:             _yt.balanceOf(from)
269:         );
..
271:         uint256 sharesRedeemed = pyAmount.divWadDown(_gscales.maxscale);
...
283:         _target.safeTransfer(address(adapter), sharesRedeemed + accruedInTarget);
284:         (uint256 amountWithdrawn, ) = adapter.prefundedRedeem(to);
...
287:         return amountWithdrawn;
288:     }

```

The following coded PoC shows all unclaimed and  unaccrued target token could be drained out, even if the allowance is as low as ````1wei````.
```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {TestTranche} from "./Tranche.t.sol";
import "forge-std/console2.sol";

contract TrancheAllowanceIssue is TestTranche {
    address bob = address(0x22);
    function setUp() public virtual override {
        super.setUp();
    }

    function testTrancheAllowanceIssue() public {
        // 1. issue some PT and YT
        deal(address(underlying), address(this), 1_000e6, true);
        tranche.issue(address(this), 1_000e6);

        // 2. generating some unclaimed yield
        vm.warp(block.timestamp + 30 days);
        _simulateScaleIncrease();

        // 3. give bob any negligible allowance, could be as low as only 1wei
        tranche.approve(bob, 1);
        yt.approve(bob, 1);

        // 4. all unclaimed and pending yield drained by bob
        assertEq(0, underlying.balanceOf(bob));
        vm.prank(bob);
        tranche.redeemWithYT(address(this), bob, 1);
        assertTrue(underlying.balanceOf(bob) > 494e6);
    }
}
```

And the logs:
```solidity
2024-01-napier\napier-v1> forge test --match-test testTrancheAllowanceIssue -vv
[⠔] Compiling...
[⠊] Compiling 42 files with 0.8.19
[⠔] Solc 0.8.19 finished in 82.11sCompiler run successful!
[⠒] Solc 0.8.19 finished in 82.11s

Running 1 test for test/unit/TrancheAllowanceIssue.t.sol:TrancheAllowanceIssue
[PASS] testTrancheAllowanceIssue() (gas: 497585)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.06ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```


## Impact
Users lost all unclaimed and unaccrued yield 

## Code Snippet
https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L283
## Tool used

Manual Review

## Recommendation
```diff
diff --git a/napier-v1/src/Tranche.sol b/napier-v1/src/Tranche.sol
index 62d9562..65db5c6 100644
--- a/napier-v1/src/Tranche.sol
+++ b/napier-v1/src/Tranche.sol
@@ -275,12 +275,15 @@ contract Tranche is BaseToken, ReentrancyGuard, Pausable, ITranche {
         delete unclaimedYields[from];
         gscales = _gscales;

+        uint256 accruedProportional = accruedInTarget * pyAmount / _yt.balanceOf(from);
+        unclaimedYields[from] = accruedInTarget - accruedProportional;
+        
         // Burn PT and YT tokens from `from`
         _burnFrom(from, pyAmount);
         _yt.burnFrom(from, msg.sender, pyAmount);

         // Withdraw underlying tokens from the adapter and transfer them to the user
-        _target.safeTransfer(address(adapter), sharesRedeemed + accruedInTarget);
+        _target.safeTransfer(address(adapter), sharesRedeemed + accruedProportional);
         (uint256 amountWithdrawn, ) = adapter.prefundedRedeem(to);

         emit RedeemWithYT(from, to, amountWithdrawn);
```



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: high(1)



# Issue H-2: Withdrawal can be blocked 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/89 

## Found by 
xiaoming90
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



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: high(8)



# Issue H-3: LP Tokens always valued at 3 PTs 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/90 

## Found by 
xiaoming90
## Summary

LP Tokens are always valued at 3 PTs. As a result, users of the AMM pool might receive fewer assets/PTs than expected. The AMM pool might be unfairly arbitraged, resulting in a loss for the pool's LPs.

## Vulnerability Detail

The Napier AMM pool facilitates trade between underlying assets and PTs. The PTs in the pool are represented by the Curve's Base LP Token of the Curve's pool that holds the PTs. The Napier AMM pool and Router math assumes that the Base LP Token is equivalent to 3 times the amount of PTs, as shown below. When the pool is initially deployed, it is correct that the LP token is equivalent to 3 times the amount of PT.

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/libs/PoolMath.sol#L106


```solidity
File: PoolMath.sol
83:     int256 internal constant N_COINS = 3;
..SNIP..
96:     function swapExactBaseLpTokenForUnderlying(PoolState memory pool, uint256 exactBaseLptIn)
97:         internal
..SNIP..
105:             // Note: Here we are multiplying by N_COINS because the swap formula is defined in terms of the amount of PT being swapped.
106:             // BaseLpt is equivalent to 3 times the amount of PT due to the initial deposit of 1:1:1:1=pt1:pt2:pt3:Lp share in Curve pool.
107:             exactBaseLptIn.neg() * N_COINS
..SNIP..
120:     function swapUnderlyingForExactBaseLpToken(PoolState memory pool, uint256 exactBaseLptOut)
..SNIP..
125:         (int256 _netUnderlyingToAccount18, int256 _netUnderlyingFee18, int256 _netUnderlyingToProtocol18) = executeSwap(
126:             pool,
127:             // Note: sign is defined from the perspective of the swapper.
128:             // positive because the swapper is buying pt
129:             exactBaseLptOut.toInt256() * N_COINS
```

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierPool.sol#L48

```solidity
File: NapierPool.sol
47:     /// @dev Number of coins in the BasePool
48:     uint256 internal constant N_COINS = 3;
..SNIP..
226:                     totalBaseLptTimesN: baseLptUsed * N_COINS,
..SNIP..
584:             totalBaseLptTimesN: totalBaseLpt * N_COINS,
```

In Curve, LP tokens are generally priced by computing the underlying tokens per share, hence dividing the total underlying token amounts by the total supply of the LP token. Given that the underlying assets in Curve’s stable swap are pegged to each other, the invariant’s $D$ value can be computed to estimate the total value of the underlying tokens.

Curve itself provides a function `get_virtual_price` that computes the price of the LP token by dividing $D$ with the total supply. 

Note that for LP tokens, the ratio of the total underlying value and the total supply will grow (fee mechanism) over time. Thus, the virtual price’s value will increase over time.

This means the LP token will be worth more than 3 PTs in the Curve Pool over time. However, the Naiper AMM pool still values its LP token at a constant value of 3 PTs. This discrepancy between the value of the LP tokens in the Napier AMM pool and Curve pool might result in various issues, such as the following:

- Investors brought LP tokens at the price of 3.X PT from the market. The LP tokens are deposited into or swap into the Napier AMM pool. The Naiper Pool will always assume that the price of the LP token is 3 PTs, thus shortchanging the number of assets or PTs returned to users.
- Potential arbitrage opportunity where malicious users obtain the LP token from the Naiper AMM pool at a value of 3 PT and redeem the LP token at a value higher than 3 PTs, pocketing the differences.

## Impact

Users of the AMM pool might receive fewer assets/PTs than expected. The AMM pool might be unfairly arbitraged, resulting in a loss for the pool's LPs.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/libs/PoolMath.sol#L106

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierPool.sol#L48

## Tool used

Manual Review

## Recommendation

Naiper and Pendle share the same core math for their AMM pool.

In Pendle, the AMM stores the PTs and SY (Standard Yield Token). When performing any operation (e.g., deposit, swap), the SY will be converted to the underlying assets based on SY's current rate before performing any math operation. If it is a SY (wstETH), the SY's rate will be the [current exchange rate](https://github.com/pendle-finance/pendle-core-v2-public/blob/39f3a613e2ec55dc38a7d3c562529d67844a263a/contracts/core/StandardizedYield/implementations/PendleWstEthSY.sol#L88) for wstETH to stETH/ETH. One could also think the AMM's reserve is PTs and Underlying Assets under the hood.

In Napier, the AMM stores the PTs and Curve's LP tokens. When performing any operation, the math will always convert the LP token to underlying assets using a static exchange rate of 3. However, this is incorrect, as the value of an LP token will grow over time. The AMM should value the LP tokens based on their current value. The virtual price of the LP token and other information can be leveraged to derive the current value of the LP tokens to facilitate the math operation within the pool.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid; high(9)



# Issue H-4: YT holders cannot receive a portion of the principal allocated by the PT holders due to the manipulation 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/92 

## Found by 
xiaoming90
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

# Issue H-5: Victim's fund can be stolen due to rounding error and exchange rate manipulation 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/94 

## Found by 
Bandit, LTDingZhen, cawfree, jennifer37, xAlismx, xiaoming90
## Summary

Victim's funds can be stolen by malicious users by exploiting the rounding error and through exchange rate manipulation.

## Vulnerability Detail

The LST Adaptor attempts to guard against the well-known vault inflation attack by reverting the TX when the amount of shares minted is rounded down to zero in Line 78 below.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L71

```solidity
File: BaseLSTAdapter.sol
71:     function prefundedDeposit() external nonReentrant returns (uint256, uint256) {
72:         uint256 bufferEthCache = bufferEth; // cache storage reads
73:         uint256 queueEthCache = withdrawalQueueEth; // cache storage reads
74:         uint256 assets = IWETH9(WETH).balanceOf(address(this)) - bufferEthCache; // amount of WETH deposited at this time
75:         uint256 shares = previewDeposit(assets);
76: 
77:         if (assets == 0) return (0, 0);
78:         if (shares == 0) revert ZeroShares();
```

However, this control alone is not sufficient to guard against vault inflation attacks. 

Let's assume the following scenario (ignoring fee for simplicity's sake):

1. The victim initiates a transaction that deposits 10 ETH as the underlying asset when there are no issued estETH shares.
2. The attacker observes the victim’s transaction and deposits 1 wei of ETH (issuing 1 wei of estETH share) before the victim’s transaction. 1 wei of estETH share worth of PT and TY will be minted to the attacker.
3. Then, the attacker executes a transaction to directly transfer 5 stETH to the adaptor. The exchange rate at this point is `1 wei / (5 ETH + 1 wei)`. Note that the [`totalAssets`](https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/lido/StEtherAdapter.sol#L115) function uses the `balanceOf` function to compute the total underlying assets owned by the adaptor. Thus, this direct transfer will increase the total assets amount.
4. When the victim’s transaction is executed, the number of estETH shares issued is calculated as `10 ETH * 1 wei / (5 ETH + 1 wei)`, resulting in 1 wei being issued due to round-down.
5. The attacker will combine the PT + YT obtained earlier to redeem 1 wei of estETH share from the adaptor.
6. The attacker, holding 50% of the issued estETH shares (indirectly via the PT+YT he owned), receives `(15 ETH + 1 wei) / 2` as the underlying asset. 
7. The attacker seizes 25% of the underlying asset (2.5 ETH) deposited by the victim.

This scenario demonstrates that even when a revert is triggered due to the number of issued estETH share being 0, it does not prevent the attacker from capturing the user’s funds through exchange rate manipulation.

## Impact

Loss of assets for the victim.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L71

## Tool used

Manual Review

## Recommendation

Following are some of the measures that could help to prevent such an attack:

- Mint a certain amount of shares to zero address (dead address) during contract deployment (similar to what has been implemented in Uniswap V2)
- Avoid using the `balanceOf` so that malicious users cannot transfer directly to the contract to increase the assets per share. Track the total assets internally via a variable.



## Discussion

**massun-onibakuchi**

Openzeppelin's ERC4626 with a decimalsOffset=0 in the virtual share mitigates the issue, but it is recognized that it does not completely resolve it. We plan to deposit a small amount of tokens after deployment.

# Issue M-1: The last user can't quit ````Tranche```` and loss fund permanently 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/16 

## Found by 
KingNFT
## Summary
After maturity of ````Tranche````, all users are intended to quit and get back their fund and profit. But actually the last user might fail to quit and permanently loss fund due to rounding error in the ````Tranche.withdraw()````. With the consideration there would be many ````Tranche```` instances based on different ````adapters```` and different ````maturity````, many users would suffer this risk, and this vulnerability would be easily triggered to cause users fund loss.

## Vulnerability Detail
The issue arises on L338 of ````withdraw()```` function and L676 \~ L680 of ````_computePrincipalTokenRedeemed()````, ````principalAmount```` is rounded down in favor of users rather than the protocol, which makes no enough principal token burned . This would cause the protocol have no enough ````target```` token to burn for last user.

```solidity
File: src\Tranche.sol
328:     function withdraw(
329:         uint256 underlyingAmount,
330:         address to,
331:         address from
332:     ) external override nonReentrant expired returns (uint256) {

337:         uint256 sharesRedeem = underlyingAmount.divWadDown(cscale);
338:         uint256 principalAmount = _computePrincipalTokenRedeemed(_gscales, sharesRedeem);
...
343:         _burnFrom(from, principalAmount);
...
350:     }

File: src\Tranche.sol
668:     function _computePrincipalTokenRedeemed(
669:         GlobalScales memory _gscales,
670:         uint256 _shares
671:     ) internal view returns (uint256) {
...
674:         if ((_gscales.mscale * MAX_BPS) / _gscales.maxscale >= oneSubTilt) {
...
676:             return (_shares.mulWadDown(_gscales.mscale) * MAX_BPS) / oneSubTilt;
677:         }
...
680:         return _shares.mulWadDown(_gscales.maxscale);
681:     }

```

The coded PoC shows more specific procedures how this would occur:
```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {TestTranche} from "./Tranche.t.sol";
import "forge-std/console2.sol";

contract TrancheRoundIssue is TestTranche {
    address alice = address(0x011);
    address bob = address(0x22);
    function setUp() public virtual override {
        super.setUp();
    }

    function testTrancheRoundIssue() public {
        // 1. issue some PT and YT
        deal(address(underlying), address(this), 1_000e6, true);
        tranche.issue(address(this), 1_000e6);

        // 2. alice and bob hold some YT
        yt.transfer(alice, yt.balanceOf(address(this)) / 2);
        yt.transfer(bob, yt.balanceOf(address(this)));

        // 3. after maturity
        vm.warp(_maturity);
        _simulateScaleIncrease();

        // 4. the withdraw() is called some times, typically causing "1" round error each time
        for (uint256 i; i < 10; ++i) {
            tranche.withdraw(2, address(this), address(this));
        }

        // 5. other operations such as collects, redeems and fee collection
        tranche.redeem(tranche.balanceOf(address(this)), address(this), address(this));
        vm.prank(management);
        tranche.claimIssuanceFees();
        vm.prank(alice);
        tranche.collect();

        // 6. bob becames the last unlucky guy, quit would fail
        vm.startPrank(bob);
        vm.expectRevert("ERC20: transfer amount exceeds balance");
        tranche.collect();
        vm.stopPrank();
    }
}
```

And the logs:
```solidity
2024-01-napier\napier-v1> forge test --match-test testTrancheRoundIssue
[⠔] Compiling...
[⠔] Compiling 40 files with 0.8.19
[⠊] Solc 0.8.19 finished in 74.33sCompiler run successful!
[⠒] Solc 0.8.19 finished in 74.33s

Running 1 test for test/unit/TrancheRoundIssue.t.sol:TrancheRoundIssue
[PASS] testTrancheRoundIssue() (gas: 1266016)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.58ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```


## Impact
The last user would fail to quit and permanently loss fund

## Code Snippet
https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L668
## Tool used

Manual Review

## Recommendation
```diff
diff --git a/napier-v1/src/Tranche.sol b/napier-v1/src/Tranche.sol
index 62d9562..457b6c5 100644
--- a/napier-v1/src/Tranche.sol
+++ b/napier-v1/src/Tranche.sol
@@ -673,11 +673,11 @@ contract Tranche is BaseToken, ReentrancyGuard, Pausable, ITranche {
         // If it's a sunny day, PT holders lose `tilt` % of the principal amount.
         if ((_gscales.mscale * MAX_BPS) / _gscales.maxscale >= oneSubTilt) {
             // Formula: principalAmount = (shares * mscale * MAX_BPS) / oneSubTilt
-            return (_shares.mulWadDown(_gscales.mscale) * MAX_BPS) / oneSubTilt;
+            return (_shares.mulWadUp(_gscales.mscale) * MAX_BPS) / oneSubTilt;
         }
         // If it's not a sunny day,
         // Formula: principalAmount = shares * maxscale
-        return _shares.mulWadDown(_gscales.maxscale);
+        return _shares.mulWadUp(_gscales.maxscale);
     }
```

# Issue M-2: incorrect allowance check in NapierRouter::removeLiquidityOneUnderlying() 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/50 

## Found by 
jennifer37
## Summary
incorrect allowance check in NapierRouter::removeLiquidityOneUnderlying()

## Vulnerability Detail
In function removeLiquidityOneUnderlying(), we remove liquidity from pool to get underlying token and baseLpt in router. And then we will swap baseLpt to underlying in pool. So before we swap baseLpt to underlying, we need to check whether pool has enough allowance to transfer router's baseLpt token.
```solidity
In function removeLiquidityOneUnderlying(), before router 
    function removeLiquidityOneUnderlying(
        address pool,
        uint256 index,
        uint256 liquidity,
        uint256 underlyingOutMin,
        address recipient,
        uint256 deadline
    ) external override nonReentrant checkDeadline(deadline) returns (uint256) {
        ...
        // The withdrawn base LP token is exchanged for underlying in two different ways depending on the maturity.
        // If maturity has passed, redeem else swap for underlying.
        // 1. Swapping is used when maturity hasn't passed because redeeming is disabled before maturity.
        // 2. Redeeming is preferred because it doesn't cause slippage.
        if (block.timestamp < INapierPool(pool).maturity()) {
            // Swap base LP token for underlying
            // approve max
    @==>if (IERC20(basePool).allowance(address(this), basePool) < baseLptOut) {
                IERC20(basePool).forceApprove(pool, type(uint256).max);
            }
            uint256 removed = INapierPool(pool).swapExactBaseLpTokenForUnderlying(baseLptOut, address(this));
            underlyingOut += removed;
        }
```

It should be 
```solidity
        if (block.timestamp < INapierPool(pool).maturity()) {
            // Swap base LP token for underlying
            // approve max
            // (pool, not base pool)
@==>        if (IERC20(basePool).allowance(address(this), pool) < baseLptOut) {
                IERC20(basePool).forceApprove(pool, type(uint256).max);
            }
            uint256 removed = INapierPool(pool).swapExactBaseLpTokenForUnderlying(baseLptOut, address(this));
            underlyingOut += removed;
        }
```
## Impact
Incorrect allowance check.

## Code Snippet
https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L744-L752

## Tool used

Manual Review

## Recommendation
change basePool to pool
```solidity
        if (block.timestamp < INapierPool(pool).maturity()) {
            // Swap base LP token for underlying
            // approve max
   ==>    if (IERC20(basePool).allowance(address(this), pool) < baseLptOut) {
                IERC20(basePool).forceApprove(pool, type(uint256).max);
            }
            uint256 removed = INapierPool(pool).swapExactBaseLpTokenForUnderlying(baseLptOut, address(this));
            underlyingOut += removed;
        } 
```

# Issue M-3: YT holder are unable to claim their interest 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/80 

The protocol has acknowledged this issue.

## Found by 
xiaoming90
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

# Issue M-4: Anyone can convert someone's unclaimed yield to PT + YT 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/83 

## Found by 
KingNFT, xiaoming90
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



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid: he saves her a fee by doing so; similar: https://discordapp.com/channels/812037309376495636/1192483776622759956/1199612656395505684



# Issue M-5: Lack of slippage control for `issue` function 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/84 

## Found by 
xiaoming90
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



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: slippage; medium(15)



# Issue M-6: Users unable to withdraw their funds due to FRAX admin action 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/95 

The protocol has acknowledged this issue.

## Found by 
xiaoming90
## Summary

FRAX admin action can lead to the fund of Naiper protocol and its users being stuck, resulting in users being unable to withdraw their assets.

## Vulnerability Detail

Per the contest page, the admins of the protocols that Napier integrates with are considered "RESTRICTED". This means that any issue related to FRAX's admin action that could negatively affect Napier protocol/users will be considered valid in this audit contest.

> Q: Are the admins of the protocols your contracts integrate with (if any) TRUSTED or RESTRICTED?
> RESTRICTED

When the Adaptor needs to unstake its staked ETH to replenish its ETH buffer so that users can redeem/withdraw their funds, it will first join the FRAX's redemption queue, and the queue will issue a redemption NFT afterward. After a certain period, the adaptor can claim their ETH by burning the redemption NFT at Line 65 via the `burnRedemptionTicketNft` function.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/frax/SFrxETHAdapter.sol#L65

```solidity
File: SFrxETHAdapter.sol
53:     function claimWithdrawal() external override {
54:         uint256 _requestId = requestId;
55:         uint256 _withdrawalQueueEth = withdrawalQueueEth;
56:         if (_requestId == 0) revert NoPendingWithdrawal();
57: 
58:         /// WRITE ///
59:         delete withdrawalQueueEth;
60:         delete requestId;
61:         bufferEth += _withdrawalQueueEth.toUint128();
62: 
63:         /// INTERACT ///
64:         uint256 balanceBefore = address(this).balance;
65:         REDEMPTION_QUEUE.burnRedemptionTicketNft(_requestId, payable(this));
66:         if (address(this).balance < balanceBefore + _withdrawalQueueEth) revert InvariantViolation();
67: 
68:         IWETH9(Constants.WETH).deposit{value: _withdrawalQueueEth}();
69:     }
```

However, it is possible for FRAX's admin to disrupt the redemption process of the adaptor, resulting in Napier users being unable to withdraw their funds. When the `burnRedemptionTicketNft` function is executed, the redemption NFT will be burned, and native ETH residing in the `FraxEtherRedemptionQueue` contract will be sent to the adaptor at Line 498 below

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3761

```solidity
File: FraxEtherRedemptionQueue.sol
473:     function burnRedemptionTicketNft(uint256 _nftId, address payable _recipient) external nonReentrant {
..SNIP..
494:         // Effects: Burn frxEth to match the amount of ether sent to user 1:1
495:         FRX_ETH.burn(_redemptionQueueItem.amount);
496: 
497:         // Interactions: Transfer ETH to recipient, minus the fee
498:         (bool _success, ) = _recipient.call{ value: _redemptionQueueItem.amount }("");
499:         if (!_success) revert InvalidEthTransfer();
```

FRAX admin could execute the `recoverEther` function to transfer out all the Native ETH residing in the `FraxEtherRedemptionQueue` contract, resulting in the NFT redemption failing due to lack of ETH. 

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3381

```solidity
File: FraxEtherRedemptionQueue.sol
185:     /// @notice Recover ETH from exits where people early exited their NFT for frxETH, or when someone mistakenly directly sends ETH here
186:     /// @param _amount Amount of ETH to recover
187:     function recoverEther(uint256 _amount) external {
188:         _requireSenderIsTimelock();
189: 
190:         (bool _success, ) = address(msg.sender).call{ value: _amount }("");
191:         if (!_success) revert InvalidEthTransfer();
192: 
193:         emit RecoverEther({ recipient: msg.sender, amount: _amount });
194:     }
```

As a result, Napier users will not be able to withdraw their funds.

## Impact

The fund of Naiper protocol and its users will be stuck, resulting in users being unable to withdraw their assets.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/frax/SFrxETHAdapter.sol#L65

## Tool used

Manual Review

## Recommendation

Ensure that the protocol team and its users are aware of the risks of such an event and develop a contingency plan to manage it.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: per contest ReadMe; this should be valid; medium(11)



# Issue M-7: `withdraw` function does not comply with ERC5095 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/96 

## Found by 
0xVolodya, Bandit, jennifer37, xiaoming90
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



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: medium(4)



# Issue M-8: AMM will revert if exchange rate is one 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/100 

The protocol has acknowledged this issue.

## Found by 
xiaoming90
## Summary

The AMM will stop working unexpectedly when the `preTradeExchangeRate` is 1.0.

## Vulnerability Detail

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/libs/PoolMath.sol#L213

```solidity
File: PoolMath.sol
203:     function _getRateAnchor(
204:         uint256 totalBaseLptTimesN,
205:         uint256 lastLnImpliedRate,
206:         uint256 totalUnderlying18,
207:         int256 rateScalar,
208:         uint256 timeToExpiry
209:     ) internal pure returns (int256 rateAnchor) {
210:         // `extRate(t*) = e^(lastLnImpliedRate * yearsToExpiry(t))`
211:         // Get pre-trade exchange rate with zero-fee
212:         int256 preTradeExchangeRate = _getExchangeRateFromImpliedRate(lastLnImpliedRate, timeToExpiry);
..SNIP..
213:         // exchangeRate should not be below 1.
214:         // But it is mathematically almost impossible to happen because `exp(x) < 1` is satisfied for all `x < 0`.
215:         // Here x = lastLnImpliedRate * yearsToExpiry(t), which is very unlikely to be negative.(or
216:         // more accurately the natural log rounds down to zero). `lastLnImpliedRate` is guaranteed to be positive when it is set
217:         // and `yearsToExpiry(t)` is guaranteed to be positive because swap can only happen before maturity.
218:         // We still check for this case to be safe.
219:         require(preTradeExchangeRate > SignedMath.WAD);
220:         uint256 proportion = totalBaseLptTimesN.divWadDown(totalBaseLptTimesN + totalUnderlying18);
221:         int256 lnProportion = _logProportion(proportion);
222: 
223:         // Compute `rateAnchor(t) = extRate(t*) - ln(portion(t*)) / rateScalar(t)`
224:         rateAnchor = preTradeExchangeRate - lnProportion.divWadDown(rateScalar);
```

At Line 219, it requires that the `preTradeExchangeRate` be larger than 1.0. However, technically, the exchange rate can be 1.0 based on the comment in Line 213 that the exchange rate should not be below one, which means that the exchange rate should be 1.0 or above. Thus, when the `preTradeExchangeRate` is 1.0, the AMM will revert unexpectedly.

## Impact

The AMM might stop working unexpectedly. Breaking of core protocol/contract functionality.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/libs/PoolMath.sol#L213

## Tool used

Manual Review

## Recommendation

Consider making the following change:

```diff
function _getRateAnchor(
    uint256 totalBaseLptTimesN,
    uint256 lastLnImpliedRate,
    uint256 totalUnderlying18,
    int256 rateScalar,
    uint256 timeToExpiry
) internal pure returns (int256 rateAnchor) {
	..SNIP..
-    require(preTradeExchangeRate > SignedMath.WAD);
+    require(preTradeExchangeRate >= SignedMath.WAD);
```

Sidenote: This is also implemented in [Pendle's Math Library](https://github.com/pendle-finance/pendle-core-v2-public/blob/2de25376697d077629f28f5d2fc165582f565aac/contracts/libraries/math/MarketMathCore.sol#L251)

# Issue M-9: `swapUnderlyingForYt` revert due to rounding issues 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/101 

## Found by 
xiaoming90
## Summary

The core function (`swapUnderlyingForYt`) of the Router will revert due to rounding issues. Users who intend to swap underlying assets to YT tokens via the Router will be unable to do so.

## Vulnerability Detail

The `swapUnderlyingForYt` allows users to swap underlying assets to a specific number of YT tokens they desire.

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L353

```solidity
File: NapierRouter.sol
297:     function swapUnderlyingForYt(
298:         address pool,
299:         uint256 index,
300:         uint256 ytOutDesired,
301:         uint256 underlyingInMax,
302:         address recipient,
303:         uint256 deadline
304:     ) external payable override nonReentrant checkDeadline(deadline) returns (uint256) {
..SNIP..
330:             // Variable Definitions:
331:             // - `uDeposit`: The amount of underlying asset that needs to be deposited to issue PT and YT.
332:             // - `ytOutDesired`: The desired amount of PT and YT to be issued.
333:             // - `cscale`: Current scale of the Tranche.
334:             // - `maxscale`: Maximum scale of the Tranche (denoted as 'S' in the formula).
335:             // - `issuanceFee`: Issuance fee in basis points. (10000 =100%).
336: 
337:             // Formula for `Tranche.issue`:
338:             // ```
339:             // shares = uDeposit / s
340:             // fee = shares * issuanceFeeBps / 10000
341:             // pyIssue = (shares - fee) * S
342:             // ```
343: 
344:             // Solving for `uDeposit`:
345:             // ```
346:             // uDeposit = (pyIssue * s / S) / (1 - issuanceFeeBps / 10000)
347:             // ```
348:             // Hack:
349:             // Buffer is added to the denominator.
350:             // This ensures that at least `ytOutDesired` amount of PT and YT are issued.
351:             // If maximum scale and current scale are significantly different or `ytOutDesired` is small, the function might fail.
352:             // Without this buffer, any rounding errors that reduce the issued PT and YT could lead to an insufficient amount of PT to be repaid to the pool.
353:             uint256 uDepositNoFee = cscale * ytOutDesired / maxscale;
354:             uDeposit = uDepositNoFee * MAX_BPS / (MAX_BPS - (series.issuanceFee + 1)); // 0.01 bps buffer
```

Line 353-354 above compute the number of underlying deposits needed to send to the Tranche to issue the amount of YT token the users desired. It attempts to add a buffer of 0.01 bps buffer to prevent rounding errors that could lead to insufficient PT being repaid to the pool and result in a revert. During the audit, it was found that this buffer is ineffective in achieving its purpose.

The following example/POC demonstrates a revert could still occur due to insufficient PT being repaid despite having a buffer:

Let the state be the following:

- cscale = 1.2e18
- maxScale = 1.25e18
- ytOutDesired = 123
- issuanceFee = 0% (For simplicity's sake, the fee is set to zero. Having fee or not does not affect the validity of this issue as this is a math problem)

The following computes the number of underlying assets to be transferred to the Tranche to mint/issue PY + YT

```solidity
uDepositNoFee = cscale * ytOutDesired / maxscale;
uDepositNoFee = 1.2e18 * 123 / 1.25e18 = 118.08 = 118 (Round down)

uDeposit = uDepositNoFee * MAX_BPS / (MAX_BPS - (series.issuanceFee + 1))
uDeposit = 118 * 10000 / (10000 - (0 + 1)) = 118.0118012 = 118 (Round down)
```

Subsequently, the code will perform a flash-swap via the `swapPtForUnderlying` function. It will borrow 123 PT from the pool, which must be repaid later.

In the swap callback function, the code will transfer 118 underlying assets to the Tranche and execute the `Tranche.issue` function to mint/issue PY + YT.

Within the `Tranche.issue` function, it will trigger the `adapter.prefundedDeposit()` function to mint the estETH/shares. The following is the number of estETH/shares minted:

```solidity
shares = assets * (total supply/total assets)
sahres = 118 * 100e18 / 120e18 = 98.33333333 = 98 shares
```

Next, Line 219 below of the `Tranche.issue` function will compute the number of PY+YT to be issued/minted

```solidity
issued = (sharesUsed - fee).mulWadDown(_maxscale);
issued = (sharesUsed - 0).mulWadDown(_maxscale);
issued = sharesUsed.mulWadDown(_maxscale);

issued = sharesUsed * _maxscale / WAD
issued = 98 * 1.25e18 / 1e18 = 122.5 = 122 PT (Round down)
```

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/Tranche.sol#L219

```solidity
File: Tranche.sol
179:     function issue(
180:         address to,
181:         uint256 underlyingAmount
182:     ) external nonReentrant whenNotPaused notExpired returns (uint256 issued) {
..SNIP..
217:         uint256 sharesUsed = sharesMinted + accruedInTarget;
218:         uint256 fee = sharesUsed.mulDivUp(issuanceFeeBps, MAX_BPS);
219:         issued = (sharesUsed - fee).mulWadDown(_maxscale);
```

At the end of the `Tranche.issue` function, 122 PY + YT is issued/minted back to the Router.

Note that 123 PT was flash-loaned earlier, and 123 PT needs to be repaid. Otherwise, the code at Line 164 below will revert. The main problem is that only 122 PY was issued/minted (a shortfall of 1 PY). Thus, the swap TX will revert at the end.

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L164

```solidity
File: NapierRouter.sol
085:     function swapCallback(int256 underlyingDelta, int256 ptDelta, bytes calldata data) external override {
..SNIP..
161:             uint256 pyIssued = params.pt.issue({to: address(this), underlyingAmount: params.underlyingDeposit});
162: 
163:             // Repay the PT to Napier pool
164:             if (pyIssued < pyDesired) revert Errors.RouterInsufficientPtRepay();
```

## Impact

The core function (swapUnderlyingForYt) of the Router will break. Users who intend to swap underlying assets to YT tokens via the Router will not be able to do so.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L353

## Tool used

Manual Review

## Recommendation

The buffer does not appear to be the correct approach to manage this rounding error. One could increase the buffer from 0.01% to 1% and solve the issue in the above example, but a different or larger number might cause a rounding error to surface again. Also, a larger buffer means that many unnecessary PTs will be issued. 

Thus, it is recommended that a round-up division be performed when computing the `uDepositNoFee` and `uDeposit` using functions such as `divWadUp` so that the issued/minted PT can cover the debt.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: rounding error: medium(7)



# Issue M-10: Unable to deposit to Tranche/Adaptor under certain conditions 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/105 

## Found by 
AuditorPraise, xiaoming90
## Summary

Minting of PT and YT is the core feature of the protocol. Without the ability to mint PT and YT, the protocol would not operate. 

The user cannot deposit into the Tranche to issue new PT + YT under certain conditions.

## Vulnerability Detail

The comment in Line 133 below mentioned that the `stakeAmount` can be zero. 

The reason is that when `targetBufferEth < (availableEth + queueEthCache)`,  it is possible that there is a pending withdrawal request (`queueEthCache`) and no available ETH left in the buffer (`availableEth = 0`). Refer to the comment in Line 123 below.

As a result, the code at Line 127 below will restrict the amount of ETH to be staked and set the `stakeAmount` to zero.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L133

```solidity
File: BaseLSTAdapter.sol
071:     function prefundedDeposit() external nonReentrant returns (uint256, uint256) {
..SNIP..
113:         uint256 stakeAmount;
114:         unchecked {
115:             stakeAmount = availableEth + queueEthCache - targetBufferEth; // non-zero, no underflow
116:         }
117:         // If the stake amount exceeds 95% of the available ETH, cap the stake amount.
118:         // This is to prevent the buffer from being completely drained. This is not a complete solution.
119:         //
120:         // The condition: stakeAmount > availableEth, is equivalent to: queueEthCache > targetBufferEth
121:         // Possible scenarios:
122:         // - Target buffer percentage was changed to a lower value and there is a large withdrawal request pending.
123:         // - There is a pending withdrawal request and the available ETH are not left in the buffer.
124:         // - There is no pending withdrawal request and the available ETH are not left in the buffer.
125:         uint256 maxStakeAmount = (availableEth * 95) / 100;
126:         if (stakeAmount > maxStakeAmount) {
127:             stakeAmount = maxStakeAmount; // max 95% of the available ETH
128:         }
129: 
130:         /// INTERACT ///
131:         // Deposit into the yield source
132:         // Actual amount of ETH spent may be less than the requested amount.
133:         stakeAmount = _stake(stakeAmount); // stake amount can be 0
```

However, the issue is that when `_stake` function is called with `stakeAmount` set to zero, it will result in zero ETH being staked and Line 77 below will revert.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/lido/StEtherAdapter.sol#L77

```solidity
File: StEtherAdapter.sol
64:     /// @inheritdoc BaseLSTAdapter
65:     /// @dev Lido has a limit on the amount of ETH that can be staked.
66:     /// @dev Need to check the current staking limit before staking to prevent DoS.
67:     function _stake(uint256 stakeAmount) internal override returns (uint256) {
68:         uint256 stakeLimit = STETH.getCurrentStakeLimit();
69:         if (stakeAmount > stakeLimit) {
70:             // Cap stake amount
71:             stakeAmount = stakeLimit;
72:         }
73: 
74:         IWETH9(Constants.WETH).withdraw(stakeAmount);
75:         uint256 _stETHAmt = STETH.submit{value: stakeAmount}(address(this));
76: 
77:         if (_stETHAmt == 0) revert InvariantViolation();
78:         return stakeAmount;
79:     }
```

A similar issue also occurs for the sFRXETH adaptor. If `FRXETH_MINTER.submit` function is called with `stakeAmount == 0`, it will revert.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/frax/SFrxETHAdapter.sol#L76

```solidity
File: SFrxETHAdapter.sol
71:     /// @notice Mint sfrxETH using WETH
72:     function _stake(uint256 stakeAmount) internal override returns (uint256) {
73:         IWETH9(Constants.WETH).withdraw(stakeAmount);
74:         FRXETH_MINTER.submit{value: stakeAmount}();
75:         uint256 received = STAKED_FRXETH.deposit(stakeAmount, address(this));
76:         if (received == 0) revert InvariantViolation();
77: 
78:         return stakeAmount;
79:     }
```

The following shows that the `FRXETH_MINTER.submit` function will revert if submitted ETH is zero below.

https://etherscan.io/address/0xbAFA44EFE7901E04E39Dad13167D089C559c1138#code#F1#L89

```solidity
/// @notice Mint frxETH to the recipient using sender's funds. Internal portion
function _submit(address recipient) internal nonReentrant {
    // Initial pause and value checks
    require(!submitPaused, "Submit is paused");
    require(msg.value != 0, "Cannot submit 0");
```

## Impact

Minting of PT and YT is the core feature of the protocol. Without the ability to mint PT and YT, the protocol would not operate. The user cannot deposit into the Tranche to issue new PT + YT under certain conditions. Breaking of core protocol/contract functionality.

## Code Snippet

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L133

## Tool used

Manual Review

## Recommendation

Short-circuit the `_stake` function by returning zero value immediately if the `stakeAmount` is zero.

File: StEtherAdapter.sol

```diff
function _stake(uint256 stakeAmount) internal override returns (uint256) {
+	if (stakeAmount == 0) return 0;	
	uint256 stakeLimit = STETH.getCurrentStakeLimit();
	if (stakeAmount > stakeLimit) {
		// Cap stake amount
		stakeAmount = stakeLimit;
	}

	IWETH9(Constants.WETH).withdraw(stakeAmount);
	uint256 _stETHAmt = STETH.submit{value: stakeAmount}(address(this));

	if (_stETHAmt == 0) revert InvariantViolation();
	return stakeAmount;
}
```

File: SFrxETHAdapter.sol

```diff
function _stake(uint256 stakeAmount) internal override returns (uint256) {
+	if (stakeAmount == 0) return 0;	
	IWETH9(Constants.WETH).withdraw(stakeAmount);
	FRXETH_MINTER.submit{value: stakeAmount}();
	uint256 received = STAKED_FRXETH.deposit(stakeAmount, address(this));
	if (received == 0) revert InvariantViolation();

	return stakeAmount;
}
```



## Discussion

**sherlock-admin3**

The protocol team fixed this issue in PR/commit https://github.com/napierfi/napier-v1/pull/169.

# Issue M-11: FRAX admin can adjust fee rate to harm Napier and its users 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/108 

The protocol has acknowledged this issue.

## Found by 
xiaoming90
## Summary

FRAX admin can adjust fee rates to harm Napier and its users, preventing Napier users from withdrawing.

## Vulnerability Detail

Per the contest page, the admins of the protocols that Napier integrates with are considered "RESTRICTED". This means that any issue related to FRAX's admin action that could negatively affect Napier protocol/users will be considered valid in this audit contest.

> Q: Are the admins of the protocols your contracts integrate with (if any) TRUSTED or RESTRICTED?
> RESTRICTED

Following is one of the ways that FRAX admin can harm Napier and its users.

FRAX admin can set the fee to 100%.

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3413

```solidity
File: FraxEtherRedemptionQueue.sol
217:     /// @notice Sets the fee for redeeming
218:     /// @param _newFee New redemption fee given in percentage terms, using 1e6 precision
219:     function setRedemptionFee(uint64 _newFee) external {
220:         _requireSenderIsTimelock();
221:         if (_newFee > FEE_PRECISION) revert ExceedsMaxRedemptionFee(_newFee, FEE_PRECISION);
222: 
223:         emit SetRedemptionFee({ oldRedemptionFee: redemptionQueueState.redemptionFee, newRedemptionFee: _newFee });
224: 
225:         redemptionQueueState.redemptionFee = _newFee;
226:     }
```

When the adaptor attempts to redeem the staked ETH from FRAX via the `enterRedemptionQueue` function, the 100% fee will consume the entire amount of the staked fee, leaving nothing for Napier's adaptor.

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3645

```solidity
File: FraxEtherRedemptionQueue.sol
343:     function enterRedemptionQueue(address _recipient, uint120 _amountToRedeem) public nonReentrant {
344:         // Get queue information
345:         RedemptionQueueState memory _redemptionQueueState = redemptionQueueState;
346:         RedemptionQueueAccounting memory _redemptionQueueAccounting = redemptionQueueAccounting;
347: 
348:         // Calculations: redemption fee
349:         uint120 _redemptionFeeAmount = ((uint256(_amountToRedeem) * _redemptionQueueState.redemptionFee) /
350:             FEE_PRECISION).toUint120();
351: 
352:         // Calculations: amount of ETH owed to the user
353:         uint120 _amountEtherOwedToUser = _amountToRedeem - _redemptionFeeAmount;
354: 
355:         // Calculations: increment ether liabilities by the amount of ether owed to the user
356:         _redemptionQueueAccounting.etherLiabilities += uint128(_amountEtherOwedToUser);
357: 
358:         // Calculations: increment unclaimed fees by the redemption fee taken
359:         _redemptionQueueAccounting.unclaimedFees += _redemptionFeeAmount;
360: 
361:         // Calculations: maturity timestamp
362:         uint64 _maturityTimestamp = uint64(block.timestamp) + _redemptionQueueState.queueLengthSecs;
363: 
364:         // Effects: Initialize the redemption ticket NFT information
365:         nftInformation[_redemptionQueueState.nextNftId] = RedemptionQueueItem({
366:             amount: _amountEtherOwedToUser,
367:             maturity: _maturityTimestamp,
368:             hasBeenRedeemed: false,
369:             earlyExitFee: _redemptionQueueState.earlyExitFee
370:         });
```

## Impact

Users unable to withdraw their assets. Loss of assets for the victim.

## Code Snippet

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3413

https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#code#L3645

## Tool used

Manual Review

## Recommendation

Ensure that the protocol team and its users are aware of the risks of such an event and develop a contingency plan to manage it.



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: medium(11)



# Issue M-12: The pool verification in `NapierRouter` is prone to collision attacks 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/111 

The protocol has acknowledged this issue.

## Found by 
Arabadzhiev
## Summary

Using computed pool addresses to verify the callback function callers is collision attack prone and can be abused in order to steal all token allowances of the `NapierRouter` contract

## Vulnerability Detail

With the current state of the `NapierRouter` contract implementation, both the `mintCallback` and `swapCallback` use the `_verifyCallback` function in order to verify that the address that called them is a Napier pool deployed by the `PoolFactory` attached to the router. This is done by computing the Create2 pool address using the `basePool` and `underlying` values passed in to either of the callbacks as arguments and the comparing that address to the `msg.sender`.

However, this method of verifying the caller address is collision prone, as the computation of the Create2 address is done by truncating a 256 bit keccak256 hash to 160 bits, meaning that for each address there are 2^96 possible hashes that will result in it after being truncated. Furthermore, what this means is that if a `basePool` and `underlying` combination that results in an address controlled by a malicious user is found, all token allowances given to the `NapierRouter` contract can be stolen.

In this [article](https://mystenlabs.com/blog/ambush-attacks-on-160bit-objectids-addresses) and this [report](https://github.com/sherlock-audit/2023-07-kyber-swap-judging/issues/90) from a past Sherlock contest, it is shown that this vulnerability will likely cost a few million dollars as of today, but due to the rapid advancement in computing capabilities, it is likely that it will cost much less a few years down the line.

## Impact

All token allowances of the `NapierRouter` can be stolen

## Code Snippet

[NapierRouter.sol#L65-L68](https://github.com/sherlock-audit/2024-01-napier/blob/main/v1-pool/src/NapierRouter.sol#L65-L68)

## Tool used

Manual Review

## Recommendation

In the `PoolFactory` contract, create a function that verifies that a given pool address has been deployed by it using the `_pools` mapping. Example implementation:

```diff
contract PoolFactory is IPoolFactory, Ownable2Step {
    ...
    /// @notice Mapping of NapierPool to PoolAssets
    mapping(address => PoolAssets) internal _pools;
    ...
+   function isFactoryDeployedPool(address poolAddress) external view returns (bool) {
+       PoolAssets memory poolAssets = _pools[poolAddress];
+       return (
+           poolAssets.basePool != address(0) &&
+           poolAssets.underlying != address(0) &&
+           poolAssets.principalTokens.length != 0
+       );
    }
}
```

And then use it in the `NapierRouter::_verifyCallback` function in place of the computed address comparison logic:

```diff
    function _verifyCallback(address basePool, address underlying) internal view {
-       if (
-           PoolAddress.computeAddress(basePool, underlying, POOL_CREATION_HASH, address(factory))
-               != INapierPool(msg.sender)
-       ) revert Errors.RouterCallbackNotNapierPool();
+       if (factory.isFactoryDeployedPool(msg.sender)) revert Errors.RouterCallbackNotNapierPool();
    }
```

# Issue M-13: SFrxETHAdapter redemptionQueue waiting period can DOS adapter functions 

Source: https://github.com/sherlock-audit/2024-01-napier-judging/issues/120 

The protocol has acknowledged this issue.

## Found by 
Falconhoof
## Links
https://github.com/FraxFinance/frax-ether-redemption-queue/blob/17ebebcddf31b7780e92c23a6b440dc789e5ceac/src/contracts/FraxEtherRedemptionQueue.sol#L417-L461
https://github.com/FraxFinance/frax-ether-redemption-queue/blob/17ebebcddf31b7780e92c23a6b440dc789e5ceac/src/contracts/FraxEtherRedemptionQueue.sol#L235-L246
https://github.com/sherlock-audit/2024-01-napier/blob/6313f34110b0d12677b389f0ecb3197038211e12/napier-v1/src/adapters/BaseLSTAdapter.sol#L71-L139
https://github.com/sherlock-audit/2024-01-napier/blob/6313f34110b0d12677b389f0ecb3197038211e12/napier-v1/src/adapters/BaseLSTAdapter.sol#L146-L169

## Summary
The waiting period between `rebalancer` address making a withdrawal request and the withdrawn funds being ready to claim from `FraxEtherRedemptionQueue` is extremely long which can lead to a significant period of time where some of the protocol's functions are either unusable or work in a diminished capacity.

## Vulnerability Detail
In `FraxEtherRedemptionQueue.sol`; the Queue wait time is stored in the state struct `redemptionQueueState` as `redemptionQueueState.queueLengthSecs` and is curently set to `1_296_000 Seconds` or `15 Days`; as recently as January however it was at `1_555_200 Seconds` or `18 Days`. View current setting by calling `redemptionQueueState()` [here](https://etherscan.io/address/0x82bA8da44Cd5261762e629dd5c605b17715727bd#readContract).

`BaseLSTAdapter::requestWithdrawal()` is an essential function which helps to maintain `bufferEth` at a defined, healthy level. 
`bufferEth` is a facility which smooth running of redemptions and deposits.

For `redemptions`; it allows users to redeem `underlying` without having to wait for any period of time.
However, redemption amounts requested which are less than `bufferEth` will be rejected as can be seen below in `BaseLSTAdapter::prefundedRedeem()`.
Further, there is nothing preventing `redemptions` from bringing `bufferEth` all the way to `0`.

```solidity
    function prefundedRedeem(address recipient) external virtual returns (uint256, uint256) {
        // SOME CODE

        // If the buffer is insufficient, shares cannot be redeemed immediately
        // Need to wait for the withdrawal to be completed and the buffer to be refilled.
>>>     if (assets > bufferEthCache) revert InsufficientBuffer();

        // SOME CODE
    }
```

For `deposits`; where `bufferEth` is too low, it keeps user deposits in the contract until a deposit is made which brings `bufferEth` above it's target, at which point it stakes. During this time, the deposits, which are kept in the adapter, do not earn any yield; making those funds unprofitable.

```solidity
    function prefundedDeposit() external nonReentrant returns (uint256, uint256) {
    // SOME CODE
>>>     if (targetBufferEth >= availableEth + queueEthCache) {
>>>         bufferEth = availableEth.toUint128();
            return (assets, shares);
        }
    // SOME CODE
    }
```

## Impact
If the `SFrxETHAdapter` experiences a large net redemption, bringing `bufferEth` significantly below `targetBufferEth`, the rebalancer can be required to make a withdrawal request in order to replenish the buffer.
However, this will be an ineffective action given the current, 15 Day waiting period. During the waiting period if `redemptions > deposits`, the bufferEth can be brought down to `0` which will mean a complete DOSing of the `prefundedRedeem()` function.

During the wait period too; if `redemptions >= deposits`, no new funds will be staked in `FRAX` so yields for users will decrease and may in turn lead to more redemptions.

These conditions could also necessitate the immediate calling again of `requestWithdrawal()`, given that withdrawal requests can only bring `bufferEth` up to it's target level and not beyond and during the wait period there could be further redemptions.

## Code Snippet
Simple example with Yield on `sFrxETHBalance` ignored:

Start off with 100 wETH deposited; 10 wETH `bufferEth`
> totalAssets() = (withdrawalQueueEth + bufferEth + sFrxETHBalance)
> totalAssets() = (0 + 10 + 90)

Net Redemption 5 wETH reduces bufferEth so rebalancer makes Withdrawl Request of 4.5 wETH to bring bufferEth to 10% (9.5 wEth)
> totalAssets() = (4.5 + 5 + 85.5)

During the wait period, continued Net Redemption reduces bufferEth further requiring another withdrawl request by rebalancer for 4.05 wEth
> totalAssets() = (0 + 4.5 + 85.5)

## Tool used
Foundry Testing
Manual Review

## Recommendation
Consider adding a function allowing the rebalancer call `earlyBurnRedemptionTicketNft()` in `FraxEtherRedemptionQueue.sol` when there is a necessity.
This will allow an immediate withdrawal for a fee of `0.5%`; see function [here]( https://github.com/FraxFinance/frax-ether-redemption-queue/blob/17ebebcddf31b7780e92c23a6b440dc789e5ceac/src/contracts/FraxEtherRedemptionQueue.sol#L417-L461
)



## Discussion

**massun-onibakuchi**

We are aware of the situation. Therefore, we plan to set the TARGET BUFFER PERCENTAGE passively.

