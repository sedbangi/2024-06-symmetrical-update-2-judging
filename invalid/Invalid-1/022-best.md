Noisy Rusty Wasp

Medium

# Potential Loss of Fees Due to Missing Registered Fee Collector in `openPosition` Function

## Summary
In the `openPosition` function of `PartyBFacetImpl.sol`, even if there is a valid affiliate, it is possible that there is no registered fee collector, which may result in storing the fees on the zero address. This issue arises due to the lack of checks for the existence of a registered fee collector for the affiliate address. The relevant code can be found in `PartyBFacetImpl.sol` lines [97-103](https://github.com/SYMM-IO/protocol-core/blob/develop/contracts/facets/PartyB/PartyBFacetImpl.sol#L97-L103).

## Vulnerability Detail
The flow for interaction between parties require: PartyA creates a quote, then PartyB can open a position based on that quote previously created. The function for creating a quote by PartyA should be `sendQuoteWithAffiliate` from the `PartyAFacet.sol` contract. The function to open a position by PartyB should be `openPosition` from the `PartyBFacet.sol`.

Quotes can have affiliates that may receive the balances of the fees on a given interaction. Fees balances are stored on the  account storage corresponding to the address stored on the mapping `affiliateFeeCollector` stored on the global app storage `accountLayout.balances[GlobalAppStorage.layout().affiliateFeeCollector[quote.affiliate]]`. 

However there is no check/guarantee that a valid affiliate does have a valid `affiliateFeeCollector` address assigned. The system does not enforce this requirement. So, it is possible that fees can be stored on the account storage corresponding to address zero.

It is important to remark that this can also happen for quotes created before the upgrade of the code. If positions are opened after as `affiliate` may be zero fees would also be stored on the address zero account storage.

## Impact
Improper handling of funds for the actors of the protocol.

## Code Snippet

[PartyBFacetImpl.sol#L97-L103](https://github.com/SYMM-IO/protocol-core/blob/develop/contracts/facets/PartyB/PartyBFacetImpl.sol#L97-L103)
```solidity
if (quote.orderType == OrderType.LIMIT) {
	require(quote.quantity >= filledAmount && filledAmount > 0, "PartyBFacet: Invalid filledAmount");
	accountLayout.balances[GlobalAppStorage.layout().affiliateFeeCollector[quote.affiliate]] += (filledAmount * quote.requestedOpenPrice * quote.tradingFee) / 1e36;
} else {
	require(quote.quantity == filledAmount, "PartyBFacet: Invalid filledAmount");
	accountLayout.balances[GlobalAppStorage.layout().affiliateFeeCollector[quote.affiliate]] += (filledAmount * quote.marketPrice * quote.tradingFee) / 1e36;
}
```

Code for creating a quote that does not check for a valid registered fee collector contract. [PartyAFacetImpl.sol#L72](https://github.com/SYMM-IO/protocol-core/blob/develop/contracts/facets/PartyA/PartyAFacetImpl.sol#L72)
```solidity
require(maLayout.affiliateStatus[affiliate], "PartyAFacet: Invalid affiliate");

// lock funds the in middle of way
accountLayout.pendingLockedBalances[msg.sender].add(lockedValues);
currentId = ++quoteLayout.lastId;

// create quote.
Quote memory quote = Quote({
	id: currentId,
	partyBsWhiteList: partyBsWhiteList,
	symbolId: symbolId,
	positionType: positionType,
	orderType: orderType,
	openedPrice: 0,
	initialOpenedPrice: 0,
	requestedOpenPrice: price,
	marketPrice: upnlSig.price,
	quantity: quantity,
	closedAmount: 0,
	lockedValues: lockedValues,
	initialLockedValues: lockedValues,
	maxFundingRate: maxFundingRate,
	partyA: msg.sender,
	partyB: address(0),
	quoteStatus: QuoteStatus.PENDING,
	avgClosedPrice: 0,
	requestedClosePrice: 0,
	parentId: 0,
	createTimestamp: block.timestamp,
	statusModifyTimestamp: block.timestamp,
	quantityToClose: 0,
	lastFundingPaymentTimestamp: 0,
	deadline: deadline,
	tradingFee: symbolLayout.symbols[symbolId].tradingFee,
	affiliate: affiliate
});
quoteLayout.quoteIdsOf[msg.sender].push(currentId);
quoteLayout.partyAPendingQuotes[msg.sender].push(currentId);
quoteLayout.quotes[currentId] = quote;

uint256 fee = LibQuote.getTradingFee(currentId);
accountLayout.allocatedBalances[msg.sender] -= fee;
emit SharedEvents.BalanceChangePartyA(msg.sender, fee, SharedEvents.BalanceChangeType.PLATFORM_FEE_OUT);
```

Code for creating a valid affiliate [ControlFacet.sol#L80-L84](https://github.com/SYMM-IO/protocol-core/blob/develop/contracts/facets/control/ControlFacet.sol#L80-L84)
```solidity
function registerAffiliate(address affiliate) external onlyRole(LibAccessibility.AFFILIATE_MANAGER_ROLE) {
    require(!MAStorage.layout().affiliateStatus[affiliate], "ControlFacet: Address is already registered");
    MAStorage.layout().affiliateStatus[affiliate] = true;
    emit RegisterAffiliate(affiliate);
}
```

Code for registering a fee collector [ControlFacet.sol#L141-L146](https://github.com/SYMM-IO/protocol-core/blob/develop/contracts/facets/control/ControlFacet.sol#L141-L146)
```solidity
function setFeeCollector(address affiliate, address feeCollector) external onlyRole(LibAccessibility.AFFILIATE_MANAGER_ROLE) {
    require(feeCollector != address(0), "ControlFacet: Zero address");
    require(MAStorage.layout().affiliateStatus[affiliate], "ControlFacet: Invalid affiliate");
    emit SetFeeCollector(affiliate, GlobalAppStorage.layout().affiliateFeeCollector[affiliate], feeCollector);
    GlobalAppStorage.layout().affiliateFeeCollector[affiliate] = feeCollector;
}
```

## Tool used

Manual Review

## Recommendation
Add checks to verify that `GlobalAppStorage.layout().affiliateFeeCollector[quote.affiliate]` is not the zero address. Consider performing this check when the quote is created. Also consider creating a workflow for quotes created before the update that may be opened after the code is upgraded.
Other possible approach is to ensure to enforce registering a fee collector when registering the affiliate.