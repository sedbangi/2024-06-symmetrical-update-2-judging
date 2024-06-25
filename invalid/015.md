Noisy Rusty Wasp

Medium

# Lack of Transaction ID Validation Allows Suspension and Restoration of Non-Existent Transactions

## Summary

The `suspendBridgeTransaction` and `restoreBridgeTransaction` functions do not validate if the transaction ID is valid and corresponds to an existing transaction. This can lead to suspension and restoration of non-existent transactions.

## Vulnerability Detail

Both `suspendBridgeTransaction` and `restoreBridgeTransaction` functions allow operations on transactions without checking if the transaction ID corresponds to an existing transaction. As the `enum` [`BridgeTransactionStatus.RECEIVED`](https://github.com/SYMM-IO/protocol-core/blob/develop/contracts/storages/BridgeStorage.sol#L16-L20) is equal to zero, the validations on `suspendBridgeTransaction` can by bypassed with an invalid ID.

## Impact

This vulnerability can lead to suspension and restoration of non-existent transactions. This violates a basic invariant of the system. Although it does not disrupt with the normal working of the system.

## Code Snippets

[suspendBridgeTransaction](https://github.com/SYMM-IO/protocol-core/blob/develop/contracts/facets/Bridge/BridgeFacetImpl.sol#L75-L81) function:
```solidity
function suspendBridgeTransaction(uint256 transactionId) internal {
    BridgeStorage.Layout storage bridgeLayout = BridgeStorage.layout();
    BridgeTransaction storage bridgeTransaction = bridgeLayout.bridgeTransactions[transactionId];

    require(bridgeTransaction.status == BridgeTransactionStatus.RECEIVED, "BridgeFacet: Invalid status");
    bridgeTransaction.status = BridgeTransactionStatus.SUSPENDED;
}
```

[restoreBridgeTransaction](https://github.com/SYMM-IO/protocol-core/blob/develop/contracts/facets/Bridge/BridgeFacetImpl.sol#L83-L93) function:
```solidity
function restoreBridgeTransaction(uint256 transactionId, uint256 validAmount) internal {
    BridgeStorage.Layout storage bridgeLayout = BridgeStorage.layout();
    BridgeTransaction storage bridgeTransaction = bridgeLayout.bridgeTransactions[transactionId];

    require(bridgeTransaction.status == BridgeTransactionStatus.SUSPENDED, "BridgeFacet: Invalid status");
    require(bridgeLayout.invalidBridgedAmountsPool != address(0), "BridgeFacet: Zero address");

    AccountStorage.layout().balances[bridgeLayout.invalidBridgedAmountsPool] += (bridgeTransaction.amount - validAmount);
    bridgeTransaction.status = BridgeTransactionStatus.RECEIVED;
    bridgeTransaction.amount = validAmount;
}
```

In the above functions, there is no check to ensure that the `transactionId` corresponds to a valid and existing transaction.

## Tool used

Manual Review

## Recommendation

Add validation checks to ensure that the `transactionId` corresponds to an existing transaction before performing any operations on it. For example:

```solidity
function suspendBridgeTransaction(uint256 transactionId) internal {
    BridgeStorage.Layout storage bridgeLayout = BridgeStorage.layout();
    require(transactionId <= bridgeLayout.lastId, "BridgeFacet: Invalid transactionId");
    BridgeTransaction storage bridgeTransaction = bridgeLayout.bridgeTransactions[transactionId];

    require(bridgeTransaction.status == BridgeTransactionStatus.RECEIVED, "BridgeFacet: Invalid status");
    bridgeTransaction.status = BridgeTransactionStatus.SUSPENDED;
}
```

It is not required to add any validation for the `restoreBridgeTransaction` as the `enum` validation of `BridgeTransactionStatus.SUSPENDED` can not be bypassed if `suspendBridgeTransaction` is controlled appropriately.