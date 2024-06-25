# Issue H-1: Wrong precision when adding balance within the `restoreBridgeTransaction` function 

Source: https://github.com/sherlock-audit/2024-06-symmetrical-update-2-judging/issues/5 

## Found by 
0xAadi, slowfi, xiaoming90
## Summary

Wrong precision when adding balance within the `restoreBridgeTransaction` function, leading to loss of assets.

## Vulnerability Detail

In Line 90 below, the `AccountStorage.layout().balances` stores the account's balance in 18 precision, while the `bridgeTransaction.amount` stores the amount of token to be bridged in token native precision (e.g., USDC = 6 decimals).

https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Bridge/BridgeFacetImpl.sol#L90

```solidity
File: BridgeFacetImpl.sol
83: 	function restoreBridgeTransaction(uint256 transactionId, uint256 validAmount) internal {
84: 		BridgeStorage.Layout storage bridgeLayout = BridgeStorage.layout();
85: 		BridgeTransaction storage bridgeTransaction = bridgeLayout.bridgeTransactions[transactionId];
86: 
87: 		require(bridgeTransaction.status == BridgeTransactionStatus.SUSPENDED, "BridgeFacet: Invalid status");
88: 		require(bridgeLayout.invalidBridgedAmountsPool != address(0), "BridgeFacet: Zero address");
89: 
90: 		AccountStorage.layout().balances[bridgeLayout.invalidBridgedAmountsPool] += (bridgeTransaction.amount - validAmount);
91: 		bridgeTransaction.status = BridgeTransactionStatus.RECEIVED;
92: 		bridgeTransaction.amount = validAmount;
93: 	}
```

Assume that the number of tokens to bridge is 10000 USDC (10000e6). Thus, `bridgeTransaction.amount` will be set to 10000e6. The protocol detects an anomaly with the bridging transaction and suspends it. After reviewing the transaction, the protocol decides to deduct 50% of the total bridged amount (5000 USDC).

The protocol executes `restoreBridgeTransaction` function with `validAmount` parameter set to 5000 USDC (1e6). The balance of "invalidBridgedAmountsPool" account will be incremented by 5000e6, as shown below. This is incorrect because the account balance in the protocol is denominated in 18 decimal precision. Over here, the code fails to convert the native token precision to the protocol's native precision (18) before assigning it to the account balance.

```solidity
AccountStorage.layout().balances[bridgeLayout.invalidBridgedAmountsPool] += (bridgeTransaction.amount - validAmount);
AccountStorage.layout().balances[bridgeLayout.invalidBridgedAmountsPool] += 10000e6 - 5000e6
AccountStorage.layout().balances[bridgeLayout.invalidBridgedAmountsPool] += 5000e6
```

When the protocol attempts to withdraw the assets from the "invalidBridgedAmountsPool" account, the `accountLayout.balances[msg.sender]` will be 5000e6, and thus, the maximum value of `amountWith18Decimals` will be 5000e6. If `amountWith18Decimals` is 5000e6, the maximum `amount` that can be withdrawn will be 0.000000000000005 USDC based on the following formula. 

The protocol should have received 5000 USDC, but due to a precision error, it could only receive a maximum of 0.000000000000005 USDC, resulting in a loss of assets.

```solidity
amountWith18Decimals = (amount * 1e18) / (10 ** IERC20Metadata(appLayout.collateral).decimals());
5000e6 = (amount * 1e18) / 1e6
5000e6 / 1e6 = amount * 1e18
5000 = amount * 1e18
amount = 5000/1e18
amount = 0.000000000000005
```
https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Account/AccountFacetImpl.sol#L33

```solidity
File: AccountFacetImpl.sol
26: 	function withdraw(address user, uint256 amount) internal {
27: 		AccountStorage.Layout storage accountLayout = AccountStorage.layout();
28: 		GlobalAppStorage.Layout storage appLayout = GlobalAppStorage.layout();
29: 		require(
30: 			block.timestamp >= accountLayout.withdrawCooldown[msg.sender] + MAStorage.layout().deallocateCooldown,
31: 			"AccountFacet: Cooldown hasn't reached"
32: 		);
33: 		uint256 amountWith18Decimals = (amount * 1e18) / (10 ** IERC20Metadata(appLayout.collateral).decimals());
34: 		accountLayout.balances[msg.sender] -= amountWith18Decimals;
35: 		IERC20(appLayout.collateral).safeTransfer(user, amount);
36: 	}
```

## Impact

Loss of assets due to precision error, as shown in the above scenario.

## Code Snippet

https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Bridge/BridgeFacetImpl.sol#L90

https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Account/AccountFacetImpl.sol#L33

## Tool used

Manual Review

## Recommendation

Scale up to the protocol's native precision of 18 decimals before assigning it to the account balance.

```diff
- AccountStorage.layout().balances[bridgeLayout.invalidBridgedAmountsPool] += (bridgeTransaction.amount - validAmount);
+ AccountStorage.layout().balances[bridgeLayout.invalidBridgedAmountsPool] += ((bridgeTransaction.amount - validAmount) * 1e18) / (10 ** IERC20Metadata(appLayout.collateral).decimals());
```

# Issue H-2: Suspended bridge transactions cannot be restored 

Source: https://github.com/sherlock-audit/2024-06-symmetrical-update-2-judging/issues/9 

## Found by 
slowfi, xiaoming90
## Summary

Suspended bridge transactions cannot be restored. As a result, the assets will be stuck, and bridge service providers cannot reclaim the assets they have transferred to the users from the protocol. 

## Vulnerability Detail

In Line 90 below, when restoring the bridge transaction, the invalid assets will be deposited into the account of `bridgeLayout.invalidBridgedAmountsPool`. These invalid assets can be withdrawn from this account/pool at a later time.

Per Line 88 below, if the ``bridgeLayout.invalidBridgedAmountsPool`` is zero, the `restoreBridgeTransaction` transaction will revert.

However, within the codebase, there is no way to update the `bridgeLayout.invalidBridgedAmountsPool` value. Thus, the `bridgeLayout.invalidBridgedAmountsPool` will always be zero. The `restoreBridgeTransaction` transaction will always revert and there is no way to restore a suspended bridge transaction. As a result, the assets will be stuck, and bridge service providers will not be able to reclaim the assets they have transferred to the users from the protocol.

https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Bridge/BridgeFacetImpl.sol#L88

```solidity
File: BridgeFacetImpl.sol
83: 	function restoreBridgeTransaction(uint256 transactionId, uint256 validAmount) internal {
84: 		BridgeStorage.Layout storage bridgeLayout = BridgeStorage.layout();
85: 		BridgeTransaction storage bridgeTransaction = bridgeLayout.bridgeTransactions[transactionId];
86: 
87: 		require(bridgeTransaction.status == BridgeTransactionStatus.SUSPENDED, "BridgeFacet: Invalid status");
88: 		require(bridgeLayout.invalidBridgedAmountsPool != address(0), "BridgeFacet: Zero address");
89: 
90: 		AccountStorage.layout().balances[bridgeLayout.invalidBridgedAmountsPool] += (bridgeTransaction.amount - validAmount);
91: 		bridgeTransaction.status = BridgeTransactionStatus.RECEIVED;
92: 		bridgeTransaction.amount = validAmount;
93: 	}
```

## Impact

Loss of assets. The assets will be stuck, and bridge service providers cannot reclaim the assets they have transferred to the users from the protocol. 

## Code Snippet

https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Bridge/BridgeFacetImpl.sol#L88

## Tool used

Manual Review

## Recommendation

Implement a setter function for the `invalidBridgedAmountsPool` variable.

```diff
+ function updateInvalidBridgedAmountsPool(address poolAddress) external onlyRole(LibAccessibility.DEFAULT_ADMIN_ROLE) {
+  BridgeStorage.layout().invalidBridgedAmountsPool = poolAddress;
+ }
```

# Issue M-1: Collateral can still be allocated to PartyA when the system is paused by exploiting the new internal transfer function 

Source: https://github.com/sherlock-audit/2024-06-symmetrical-update-2-judging/issues/11 

## Found by 
xiaoming90
## Summary

Collateral can still be allocated to PartyA when the system is paused by exploiting the new internal transfer function.

## Vulnerability Detail

The `allocate` and `depositAndAllocate` functions are guarded by the `whenNotAccountingPaused` modifier to ensure that collateral can only be allocated when the accounting is not paused.

https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Account/AccountFacet.sol#L48

```solidity
File: AccountFacet.sol
46: 	/// @notice Allows Party A to allocate a specified amount of collateral. Allocated amounts are which user can actually trade on.
47: 	/// @param amount The precise amount of collateral to be allocated, specified in 18 decimals.
48: 	function allocate(uint256 amount) external whenNotAccountingPaused notSuspended(msg.sender) notLiquidatedPartyA(msg.sender) {
49: 		AccountFacetImpl.allocate(amount);
..SNIP..
52: 	}
53: 
54: 	/// @notice Allows Party A to deposit a specified amount of collateral and immediately allocate it.
55: 	/// @param amount The precise amount of collateral to be deposited and allocated, specified in collateral decimals.
56: 	function depositAndAllocate(uint256 amount) external whenNotAccountingPaused notLiquidatedPartyA(msg.sender) notSuspended(msg.sender) {
57: 		AccountFacetImpl.deposit(msg.sender, amount);
58: 		uint256 amountWith18Decimals = (amount * 1e18) / (10 ** IERC20Metadata(GlobalAppStorage.layout().collateral).decimals());
59: 		AccountFacetImpl.allocate(amountWith18Decimals);
..SNIP..
63: 	}
```

However, malicious users can bypass this restriction by exploiting the newly implemented `AccountFacet.internalTransfer` function. When the global pause (`globalPaused`) and accounting pause (`accountingPaused`) are enabled, malicious users can use the `AccountFacet.internalTransfer` function, which is not guarded by the `whenNotAccountingPaused` modifier, to continue allocating collateral to their accounts, effectively bypassing the pause.

https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Account/AccountFacet.sol#L79

```solidity
File: AccountFacet.sol
74: 	/// @notice Transfers the sender's deposited balance to the user allocated balance.
75: 	/// @dev The sender and the recipient user cannot be partyB.
76: 	/// @dev PartyA should not be in the liquidation process.
77: 	/// @param user The address of the user to whom the amount will be allocated.
78: 	/// @param amount The amount to transfer and allocate in 18 decimals.
79: 	function internalTransfer(address user, uint256 amount) external whenNotAccountingPaused whenNotInternalTransferPaused notPartyB userNotPartyB(user) notSuspended(msg.sender) notLiquidatedPartyA(user){
80: 		AccountFacetImpl.internalTransfer(user, amount);
..SNIP..
85: 	}

```

## Impact

When the global pause (`globalPaused`) and accounting pause (`accountingPaused`) are enabled, this might indicate that:

1) There is an issue, error, or bug in certain areas (e.g., accounting) of the system. Thus, the funds transfer should be halted to prevent further errors from accumulating and to prevent users from suffering further losses due to this issue
2) There is an ongoing attack in which the attack path involves transferring/allocating funds to an account. Thus, the global pause (`globalPaused`) and accounting pause (`accountingPaused`) have been activated to stop the attack. However, it does not work as intended, and the hackers can continue to exploit the system by leveraging the new internal transfer function to workaround the restriction.

In both scenarios, this could lead to a loss of assets.

## Code Snippet

https://github.com/sherlock-audit/2024-06-symmetrical-update-2/blob/main/protocol-core/contracts/facets/Account/AccountFacet.sol#L79

## Tool used

Manual Review

## Recommendation

Add the `whenNotAccountingPaused` modifier to the `internalTransfer` function.

```diff
- function internalTransfer(address user, uint256 amount) external whenNotInternalTransferPaused notPartyB userNotPartyB(user) notSuspended(msg.sender) notLiquidatedPartyA(user){
+ function internalTransfer(address user, uint256 amount) external whenNotInternalTransferPaused notPartyB userNotPartyB(user) notSuspended(msg.sender) notLiquidatedPartyA(user){
  AccountFacetImpl.internalTransfer(user, amount);
```

