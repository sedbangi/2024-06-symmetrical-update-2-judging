Dancing Navy Condor

High

# Suspended bridge transactions cannot be restored

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