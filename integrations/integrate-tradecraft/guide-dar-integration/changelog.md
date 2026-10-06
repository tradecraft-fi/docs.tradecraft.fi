---
icon: clock-rotate-left
---

# Changelog



{% updates format="full" %}
{% update date="2026-10-06" %}
## **Guide v1.3.4**

Version 1.3.4 of the guide is for dar version 1.3.12.

### **Breaking changes**

**Trading balance withdrawal is now asynchronous (3.4.4).** `AMMRules_WithdrawTradingBalance` has been replaced by `AMMRules_RequestTradingBalanceWithdrawal`.

{% hint style="warning" %}
Delivery now requires the recipient to have a transfer pre-approval. Without one, the settlement stays pending; it no longer falls back to a transfer offer.
{% endhint %}

* The new choice returns `ContractId PhysicalSettlement` instead of `TransferInstructionResult`.
* The `instrumentTransferInfo` and `holdingCids` arguments are gone, and so is the `POST /v1/vault-holdings` step.
* The choice debits the TradingBalance right away. The venue delivers the tokens later by exercising `PhysicalSettlement_Redeem`.

### **New features**

**Users can now withdraw committed allocations (new section 3.3.6).**

* The actor creates a `SettlementWithdrawalRequest` (module `TC.V4.Settle.WithdrawalRequest`) with a plain create command. Each request covers one pool and one instrument.
* The venue either fulfills it, releasing the entire funding to the wallet, or rejects it with `RejectReason_PendingSettlements` or `RejectReason_InsufficientFunds`. The actor can cancel a request before the venue acts on it.
* Partial withdrawals aren't supported. Pending queued orders on the pool will fail once one of their allocations is gone.
* The section documents the `SettlementWithdrawalRequest_Fulfill_Result` type.


{% endupdate %}
{% endupdates %}
