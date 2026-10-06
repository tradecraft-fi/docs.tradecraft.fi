---
icon: file-shield
---

# Guide: DAR Integration

### Building on _Tradecraft_.

**INTEGRATION GUIDE - V1.3.3**\
A developer's reference for integrating the Canton-native AMM into your application. The Tradecraft daml package is available on request. Contact us by email at [info@tradecraft.fi](mailto:info@tradecraft.fi).

***

### 1.0 - Overview

#### How Tradecraft works

Tradecraft is a decentralized exchange protocol built natively on the Canton Network in [daml](https://github.com/digital-asset/daml). As an integrator, you will work with a small set of templates and a single shared contract:

**`AMMRules`** : _singleton_\
The protocol contract that exposes every order-creation choice. Every choice an integrator exercises on it is _nonconsuming_, so the same contract is reused across every interaction. You will need its disclosure once per pool. The venue administrators occasionally call choices (such as listing a new pool) which archive and recreate it, so refresh the disclosure rather than caching it indefinitely.

**`SwapOrder` - `DepositOrder` - `WithdrawOrder`** : _the order types_\
Each is created by exercising a choice on AMMRules. Swap orders either settle immediately from wallet holdings (3.2) or queue against committed allocations and settle when the venue fills them (3.3).

**`Allocation`** ([token-standard-v2, committed](https://github.com/canton-foundation/cips/blob/main/cip-0112/cip-0112.md#416-committed-allocations-for-prefunded-trading-and-iterated-settlement)) : _per user, per pool, per instrument_\
A committed allocation that pre-funds swap order queueing. On every fill, settlement moves tokens between the user's and the vault's allocations. Each settlement archives the allocation contract and recreates it with updated funding, so track allocations by their metadata rather than by contract ID. See section 3.3.

**`TradingBalance`** : _per user, per instrument_\
Vault-held collateral of a single token, owned by the user. Liquidity deposit and withdrawal orders draw from and settle into TradingBalances. In a future release, these will be deprecated in favor of committed allocations (above).

**The high-level workflow**

1. **Discover** available trading pairs.
2. **Submit** a SwapOrder, DepositOrder, or WithdrawOrder.
3. **Monitor** the order until it fills.

### 2.0 - Network Reference

#### Endpoints & protocol parties

The following parameters identify the live AMMs on each network. Treat them as configuration that your application should read from a single place rather than hard-coding at call sites.

**API Documentation**

* All Environments - `https://docs.tradecraft.fi/api`

**API base**

* Mainnet - `https://api.tradecraft.fi/v1`
* Testnet - `https://tradecraft.validator.test.canton.obsidian.systems/amm-http-api`
* Devnet - `https://tradecraft.validator.dev.canton.obsidian.systems/amm-http-api`

**Venue**

* Mainnet - `Tradecraft::122096fe076cc065af0cb38f94caa60e8ddfecbe8f0cfe10655ae7aa06fab99c66b7`
* Testnet - `Tradecraft::122087bab51ae50157a06730e296081f8c941d64ec96f9a2e186e159bae25a553d04`
* Devnet - `Tradecraft::122090f9041ae7a635c8471c7d496cf3158c294c154dd5468b19f8d37e949875203e`

**Vault**

* Mainnet - `cs-vault::1220b4cd6098eebafd4c88efd2b3986e86542bdc391060675432cd195ad26bcf013b`
* Testnet - `tradecraft-vault::1220369596a3a24a39f0f383600508971774e4c63e68fcc01313a0707ed70130be8e`
* Devnet - `decentralized-party::1220b1f5f3fe39721e02a886b36a5a57dc215f5d2de275d9823107bcd825b186689d`

### 3.0 - The Integration Workflow

All three order types (`SwapOrder`, `DepositOrder`, and `WithdrawOrder`) are created by exercising a nonconsuming choice on the same `AMMRules` singleton. The resulting order contracts are themselves templates you can query for status.

The workflow is organized into three tracks. Read the one that matches your integration:

* **3.2 - "Immediate" swap orders** : the simplest path for swaps.
* **3.3 - "Queued" swap orders** : use committed allocations for regular trading.
* **3.4 - Liquidity operations** : adding and removing pool liquidity.

Queued and immediate swap orders are complementary options. Integrators can offer either or both per trade. Immediate orders require transfer pre-approval on both tokens; queued orders settle through committed allocations and require no pre-approval.

{% hint style="info" %}
**NOTE:** An [API endpoint for obtaining a CIP token's allocation factory](https://docs.tradecraft.fi/api/routes/tokens#get-allocation-factory-token) may be required for some of the steps described below. See [https://docs.tradecraft.fi/api](https://docs.tradecraft.fi/api) for more information.
{% endhint %}

#### 3.1 - Discover trading pairs

Fetch the list of every active AMM pool. Each pool's `lp_token_name` in the response is its `ammId`, which you'll need for every subsequent order.

```shell
$ curl https://api.tradecraft.fi/v1/pools | jq
```

{% hint style="info" %}
**NOTE:** Each pool's `ammId` is the only identifier you need to route an order to it. The venue and vault are the same across every pool on a given network.
{% endhint %}

#### 3.2 - "Immediate" swap orders: _the simplest path_ **(\~23 kB)**

**Best for wallets that want to surface swaps to UI users, and for integrations that are not expected to make regular trades.**

{% hint style="danger" %}
**Critical safety notice:_For immediate mode swaps, a transfer pre-approval MUST be active for both tokens the user is swapping between BEFORE the order is submitted._**

Without it, **the swap still executes**: the input is consumed on fill, but **the output is NOT credited to the wallet**. It arrives as a pending transfer instruction that the user must accept before it expires, and so does the refunded input when an order fails at execution (for example on `minOut`). An instruction that is not accepted in time is never delivered, so missing pre-approval risks **loss of funds**. An order cancelled before it executes never moves the input, which stays in the user's allocation until the user withdraws it.
{% endhint %}

This call allocates the input amount from the actor's wallet holdings and creates the `SwapOrder`. On fill, the input is consumed and the output settles to the actor's wallet atomically.

The `what` field's two constructors describe the two natural shapes of a swap intent:

* **`Short`** - _Sell_ a fixed quantity of the instrument. Receive whatever the AMM quotes.
* **`Long`** - _Buy_ a fixed quantity of the instrument. Pay whatever the AMM quotes.

{% hint style="info" %}
**NOTE:** Only `Short` is currently supported. `Long` orders fail with `"Only short orders are currently supported"`.
{% endhint %}

We highly recommend testing immediate swaps on testnet first, confirming that tokens arrive in the destination wallet before going live.

**Choice Signature**

<pre class="language-daml"><code class="lang-daml"><strong>nonconsuming choice AMMRules_CreateSwapOrderFromHoldingsV2 : AMMRules_CreateSwapOrderFromHoldingsV2_Result
</strong><strong>  with
</strong>    actor : Party
      -- ^ The user performing the swap
    ammId : Text
      -- ^ Identifies which AMM pool this order is for, as returned by /pools
    what : SwapDirection
      -- ^ Direction and amount of swap. Short only - see NOTE above.
    minOut : Optional Decimal
      -- ^ Minimum output amount (slippage protection)
    holdingCids : [ContractId Holding]
      -- ^ The actor's wallet holdings of the input instrument. Holdings in
      --   excess of the swap amount are returned to the actor as change.
    allocationContext : (ContractId AllocationFactory, ExtraArgs)
      -- ^ The input instrument's AllocationFactory and its ExtraArgs, used to
      --   allocate the input amount from the holdings to the vault
</code></pre>

**Result Type**

```daml
data AMMRules_CreateSwapOrderFromHoldingsV2_Result = AMMRules_CreateSwapOrderFromHoldingsV2_Result
  with
    swapOrderCid : ContractId V2.SwapOrder
      -- ^ The pending order; queryable until it fills
    changeCids : [ContractId Holding]
      -- ^ Wallet change : input holdings in excess of the swap amount
```

{% hint style="warning" %}
**Reminder: no pre-approval on the output token = the swap will execute, but the output arrives only as a pending transfer instruction that must be accepted before it expires.**
{% endhint %}

#### 3.3 - "Queued" swap orders: _committed allocations for regular trading_

**Best for inventory providers and market makers who anticipate making regular trades against the same pool.**

The actor creates a **committed allocation** for each pool instrument, funding the one they sell, then queues swap orders against them. Orders settle through the allocations trade after trade, and no transfer pre-approval is required.

**3.3.1 - Committed allocations**

A committed allocation is a token-standard-v2 **`Allocation`** that locks a user's holdings for settlement by the venue. On every fill, settlement moves tokens between the user's and the vault's allocations.

Tradecraft settlement allocations always have the same shape:

* `committed = True` - the owner cannot withdraw the funds; only the venue can settle or cancel the allocation.
* `nextIterationFunding` enabled - the same allocation serves fill after fill. Every settlement, and every top-up (3.3.3), archives the allocation contract and recreates it with updated funding, so track allocations by their metadata rather than by contract ID.
* `settlementDeadline = None` and `executors = [venue]` - the venue is the sole settlement executor.
* Metadata binds the allocation to one pool (`tradecraft.fi/amm-id`) and one instrument (`tradecraft.fi/instrument-id`).

{% hint style="danger" %}
**Critical: a committed allocation with no settlement deadline cannot be withdrawn by its owner directly.** Only the venue (the sole executor) can settle or cancel it. Funds leave a committed allocation only through swap settlement or a venue-side cancellation. Treat allocations as working capital, not storage.\
\
To withdraw a committed allocation, the owner must submit a `SettlementWithdrawalRequest`. The venue cancels the allocation if the owner has no executed-but-unsettled trades pending, and otherwise rejects the request. See 3.3.6.
{% endhint %}

{% hint style="warning" %}
**Keep exactly one allocation per pool and instrument.** The venue selects at most one allocation per pool and instrument when settling your fills. Create the allocation once and top it up with `AMMRules_FundSettlementAllocation` (3.3.3) rather than creating additional shards.
{% endhint %}

**3.3.2 - Create a committed allocation**

A queued trade requires **two allocations: one for the input instrument and one for the output instrument**. Create both before you submit an order (3.3.4). If **either** is missing when the venue attempts to execute the order, the order fails instead of filling.

For the instrument you intend to _sell_, pass `initialFunding` and the holdings that fund it. For the instrument you intend to _receive_, the allocation may start empty (`initialFunding = None`, no holdings) or funded, as you prefer. Either way it is **required**, since it is what receives the settlement proceeds.

You will need:

* `ammCid` - the pool's V4 AMM contract ID, from the `amm` disclosure returned by `GET /disclosures/{tokenA}/{tokenB}`.
* `authorizerAccount` - the actor's token-standard (HoldingV2) account. Its owner must be the actor.
* `allocationFactoryCid` + `extraArgs` - the instrument's V2 allocation factory and its choice context, served by the instrument's token-standard registry at `POST /registry/allocation-instruction/v2/allocation-factory`. The request body wraps the allocate choice you intend to exercise: `{"choiceArguments": <AllocationFactory_Allocate arguments>, "excludeDebugFields": false}`. The response is `{"factoryId", "choiceContext"}` - pass `factoryId` as `allocationFactoryCid`, build `extraArgs` from `choiceContext.choiceContextData` (with empty `meta`), and include `choiceContext.disclosedContracts` in the submission.

The registry must create the allocation in one step. The choice fails if `AllocationFactory_Allocate` returns a pending allocation instruction.

{% hint style="warning" %}
**NOTE:** The [`GET /allocation-factory/{token}`](https://docs.tradecraft.fi/api/routes/tokens#get-allocation-factory-token) endpoint referenced in 3.0 serves the V1 allocation factory used by immediate swaps. It does not return the V2 factory this flow requires.
{% endhint %}

**Choice Signature**

```daml
nonconsuming choice AMMRules_CreateSettlementAllocation : AMMRules_CreateSettlementAllocation_Result
  with
    actor : Party
      -- ^ The user who owns the funds
    ammCid : ContractId V4.AMM
      -- ^ The pool's AMM contract, from /disclosures/{tokenA}/{tokenB}
    instrument : InstrumentId
      -- ^ Must be one of the pool's two instruments
    authorizerAccount : HoldingV2.Account
      -- ^ The actor's token-standard account; must be owned by the actor
    initialFunding : Optional Decimal
      -- ^ Some x (x > 0) = fund the allocation with x from inputHoldingCids.
      --   None = empty receiving allocation (inputHoldingCids must be empty)
    inputHoldingCids : [ContractId HoldingV2.Holding]
      -- ^ The actor's V2 holdings that fund the allocation
    allocationFactoryCid : ContractId AllocationInstructionV2.AllocationFactory
      -- ^ The instrument's V2 allocation factory
    extraArgs : ExtraArgs
      -- ^ Choice context for the allocation factory
  controller actor
```

**Result Type**

```daml
data AMMRules_CreateSettlementAllocation_Result = AMMRules_CreateSettlementAllocation_Result
  with
    allocationCid : ContractId AllocationV2.Allocation
      -- ^ The new committed allocation (queryable via the token-standard interface)
    authorizerChangeCids : TextMap [ContractId HoldingV2.Holding]
      -- ^ Change holdings returned to the actor, keyed by instrument id
```

**3.3.3 - Top up a committed allocation**

This choice adds funds to a live allocation from the actor's spare holdings in a single atomic transaction. You will additionally need the `SettlementFactory` contract and its choice context for the instrument's admin, served by the same registry at `POST /registry/allocation/v2/settlement-factory`. The request has the same shape as the allocation-factory call: the body wraps the `SettlementFactory_SettleBatch` choice arguments, and the response's `factoryId` and `choiceContext` provide `settlementFactoryCid` and `settlementExtraArgs`.

**Choice Signature**

```daml
nonconsuming choice AMMRules_FundSettlementAllocation : AMMRules_FundSettlementAllocation_Result
  with
    actor : Party
    ammCid : ContractId V4.AMM
    authorizerAccount : HoldingV2.Account
    allocationCid : ContractId AllocationV2.Allocation
      -- ^ The existing committed allocation to top up
    instrument : InstrumentId
      -- ^ The allocation's instrument
    additionalFunding : Decimal
      -- ^ Must be positive
    inputHoldingCids : [ContractId HoldingV2.Holding]
      -- ^ The actor's V2 holdings providing the additional funds
    allocationFactoryCid : ContractId AllocationInstructionV2.AllocationFactory
    allocationExtraArgs : ExtraArgs
    settlementFactoryCid : ContractId AllocationV2.SettlementFactory
      -- ^ Settlement factory for the instrument's admin
    settlementExtraArgs : ExtraArgs
  controller actor
```

**Result Type**

```daml
data AMMRules_FundSettlementAllocation_Result = AMMRules_FundSettlementAllocation_Result
  with
    settleBatchResult : AllocationV2.SettlementFactory_SettleBatchResult
      -- ^ The first entry of allocationSettleResults is the topped-up
      --   allocation, including its successor's contract ID
```

**3.3.4 - Submit a queued swap order** **(\~4.5 kB)**

A queued V2 swap order is pure intent; no tokens move at submission. When the venue fills the order, settlement debits the actor's input allocation and credits their output allocation. No wallet transfer pre-approval is needed. The output lands in the actor's allocation, not their wallet.

{% hint style="warning" %}
**Prerequisite:** Before submitting a queued swap, the actor must hold a committed allocation for the _input_ instrument funded with at least the swap input amount (on top of the inputs of the actor's other orders on the pool that execute before it and have not yet settled), and a committed allocation for the _output_ instrument (an empty receiving allocation is fine). If funding is insufficient or an allocation is missing at execution time, the order fails without touching pool state.
{% endhint %}

**Choice Signature**

```daml
nonconsuming choice AMMRules_CreateSwapOrderV2 : ContractId V2.SwapOrder
  with
    actor : Party
    ammId : Text
    what : SwapDirection
      -- ^ Direction and amount of swap. Currently only Short is
      --   supported on queued orders.
    minOut : Optional Decimal
      -- ^ Minimum output amount (slippage protection)
    delegatedOrder : Optional V2.DelegatedOrder
      -- ^ None for queued orders
    requestTradingBalanceWithdrawal : Optional Party
      -- ^ None for queued orders
    allocation : Optional V2.AllocationWithExtraArgs
      -- ^ None for queued orders. Attaching a V1 allocation
      --   settles the swap atomically at fill instead (see 3.2).
    settlementMode : Optional V2.SettlementMode
      -- ^ Some SettlementMode_DVP to settle through committed
      --   allocations. A bare None with delegatedOrder,
      --   requestTradingBalanceWithdrawal, allocation and
      --   transferInstruction all None selects the same mode,
      --   but prefer being explicit.
    transferInstruction : Optional V2.TransferInstructionWithExtraArgs
      -- ^ None for queued orders
  controller actor
```

**Lifecycle**

1. The order is created (template `TC.V4.SwapOrderV2.SwapOrder`, signatories `vault` + `actor`) and is queryable like any other order.
2. When the venue fills the order, it archives it with `Order_Complete`. An order that cannot be filled (slippage, insufficient allocation funding) is archived with `Order_Cancel` instead, whose `reason` text says why. Unlike 3.2, no tokens move in this transaction.
3. Settlement follows in a venue transaction. It moves tokens between the actor's and the vault's committed allocations, archiving the actor's allocations and recreating them with updated funding. This is why tracking them by metadata matters. Watch the output allocation for the proceeds.

To abandon a pending order, the actor exercises `SwapOrder_WithdrawV3` with `tradingBalanceCids = []` and `settlementHelpers = None`. Because no funds have moved yet, there is nothing to refund - the input stays in the actor's committed allocation throughout.

**3.3.5 - Submit queued swap orders in bulk**

If you submit many orders at once, avoid the cost of one transaction per order. Provision a `SwapOrderBatcher` once via `AMMRules_CreateSwapOrderBatcher`, then exercise `SwapOrderBatcher_CreateOrdersV2` on it to create a whole batch in a single transaction. Because the actor observes the batcher (unlike `AMMRules`, which they only receive via disclosure), the submission comes back as one exercise event instead of one create event per order.

**Choice Signatures**

```daml
-- One-time setup, on AMMRules
nonconsuming choice AMMRules_CreateSwapOrderBatcher : AMMRules_CreateSwapOrderBatcher_Result
  with
    actor : Party
  controller actor

-- On the SwapOrderBatcher contract
nonconsuming choice SwapOrderBatcher_CreateOrdersV2 : SwapOrderBatcher_CreateOrdersV2_Result
  with
    ammId : Text
      -- ^ Identifies which AMM pool the orders are for
    instrument : InstrumentId
      -- ^ Instrument sold by every order in the batch
    orders : [SwapOrderSpec]
      -- ^ Per-order amount and minOut
  controller actor

data SwapOrderSpec = SwapOrderSpec
  with
    amount : Decimal
      -- ^ How much of the input instrument to swap
    minOut : Optional Decimal
      -- ^ Minimum output amount (slippage protection)
```

**Result Types**

```daml
data AMMRules_CreateSwapOrderBatcher_Result = AMMRules_CreateSwapOrderBatcher_Result
  with
    batcher : ContractId SwapOrderBatcher
      -- ^ The batcher to exercise SwapOrderBatcher_CreateOrdersV2 on

data SwapOrderBatcher_CreateOrdersV2_Result = SwapOrderBatcher_CreateOrdersV2_Result
  with
    orders : [ContractId V2.SwapOrder]
      -- ^ The created orders
```

{% hint style="info" %}
**NOTE:** `CreateOrdersV2` creates `Short` (exact-input) orders only, and every order in the batch sells the same instrument. The orders carry no attached allocation and no explicit settlement mode, so they queue against the actor's committed allocations exactly like the orders of 3.3.4 - the same funding prerequisites apply.
{% endhint %}

**3.3.6 - Withdraw a committed allocation**

An owner cannot cancel their own committed allocation (3.3.1). Instead, the actor creates a **`SettlementWithdrawalRequest`** naming one pool and one instrument, and the venue cancels the matching allocation on their behalf. The funds return to the actor's wallet as ordinary holdings, so no transfer pre-approval is needed.

The venue releases an allocation only when the actor has no executed-but-unsettled trades on the pool. Once a trade executes, the pool price has already moved, so its settlement must not fail for lack of funding.

**Creating the request**

The actor is the request's only signatory, so the request is created with a plain create command. No `AMMRules` choice or disclosure is involved. Each request covers one instrument; to empty both of a pool's allocations, create one request per instrument.

**Template** (module `TC.V4.Settle.WithdrawalRequest`)

```daml
template SettlementWithdrawalRequest
  with
    actor : Party
      -- ^ The allocation owner
    venue : Party
      -- ^ The venue party, as on the AMMRules contract
    ammId : Text
      -- ^ The pool, as returned by /pools
    instrument : InstrumentId
      -- ^ The pool instrument whose allocation to withdraw
    createdAt : Time
      -- ^ The current time
  where
    signatory actor
    observer venue
```

**Outcomes**

The venue processes open requests automatically. Every outcome archives the request:

* **Fulfilled** (`SettlementWithdrawalRequest_Fulfill`) - the venue cancels the actor's funded allocations for the pool and instrument, releasing their entire funding to the actor's wallet. Partial withdrawals are not supported. The choice checks on-ledger that each allocation belongs to the actor and carries the request's pool and instrument metadata.
* **Rejected with `RejectReason_PendingSettlements`** - the actor has executed trades on the pool that have not settled yet. Pending settlements are not visible to the actor, so create a new request a little later.
* **Rejected with `RejectReason_InsufficientFunds`** - the actor has no funded allocation for the pool and instrument, for example only an empty receiving allocation.
* **Cancelled** (`SettlementWithdrawalRequest_Cancel`) - the actor takes the request back before the venue acts on it.

A rejected or cancelled request leaves the allocation and its funding untouched. To learn the outcome, watch for the request's archival as in 3.5. A fulfillment archives the allocations and creates the released holdings in the same transaction. If the actor still holds an allocation for the pool and instrument after the request is archived, the request was rejected, and the `reason` argument of the `SettlementWithdrawalRequest_Reject` exercise says why.

**Fulfill Result Type**

```daml
data SettlementWithdrawalRequest_Fulfill_Result = SettlementWithdrawalRequest_Fulfill_Result
  with
    withdrawnTotal : Decimal
      -- ^ The funding the allocations held before cancellation
    allocationCids : [ContractId AllocationV2.Allocation]
      -- ^ The cancelled allocations
    releasedHoldingCids : [ContractId HoldingV2.Holding]
      -- ^ The holdings released to the actor's wallet
```

{% hint style="warning" %}
**A fulfilled withdrawal leaves no allocation for that instrument.** Pending queued orders on the pool need both of the actor's allocations, so they fail at execution once one is gone. Withdraw them first with `SwapOrder_WithdrawV3` (3.3.4). To trade again, create a new allocation (3.3.2). To withdraw only part of the funding, withdraw it all and create a new allocation for the amount you want to keep committed.
{% endhint %}

#### 3.4 - Liquidity operations: _adding and removing pool liquidity_

**Best for liquidity providers who want to supply capital to a pool and earn fees.**

Deposit and withdrawal orders draw from and settle into `TradingBalance` contracts (vault-held collateral of a single token, owned by the user), so funding and managing `TradingBalance` contracts is part of every LP workflow.

**3.4.1 - Add `TradingBalance` (\~10 kB)**

Depositing tokens into a `TradingBalance` starts from a token-standard **V1** allocation of the tokens to the vault. With that allocation in place, it is a two-call sequence: fetch the disclosure for the `AMMRules` contract, then exercise the deposit choice.

**Fetching the `AMMRules` disclosure**

The path segments are the two instruments of the pool you intend to deposit into. The example below targets the CBTC/CC pool.

```shell
$ curl https://api.tradecraft.fi/v1/disclosures/CBTC/CC | jq ".amm_rules"
```

**Exercising `AMMRules_AddTradingBalance`**

With the disclosure in hand, exercise the add choice. The user is the actor. `allocationContext` names a V1 `Allocation` the user has already created with the instrument's `AllocationFactory` (see the allocation-factory endpoint in 3.0): its transfer leg sends the deposit amount from the actor to the vault, its settlement executor is the venue, and its `settleBefore` must still be in the future. The choice executes that allocation, so pass the registry's `Allocation_ExecuteTransfer` choice context as the `ExtraArgs` and include its disclosed contracts in the submission. The instrument must belong to an existing pool or be a pool LP token.

**Choice Signature**

```daml
nonconsuming choice AMMRules_AddTradingBalance : ContractId TradingBalance
  with
    actor : Party
      -- ^ The user depositing tokens
    allocationContext : (ContractId AllocationV1.Allocation, ExtraArgs)
      -- ^ V1 allocation of tokens the actor is transferring to the vault
    existingBalances : [ContractId TradingBalance]
      -- ^ Existing balances to consolidate with this one
  controller actor
```

{% hint style="info" %}
**TIP:** Avoid UTXO fragmentation. Pass _**every**_ known `TradingBalance` contract ID for the given asset into `existingBalances` on every call. They will be consolidated atomically into a single new balance, keeping your contract set tidy and reducing downstream gas costs.
{% endhint %}

**3.4.2 - Deposit orders: _adding liquidity to a pool_ (\~5 kB)**

This choice creates a `DepositOrder`. When the venue executes it, `amount1` and `amount2` are debited from the actor's `TradingBalance` contracts and added to the pool, and the minted LP tokens are credited to an LP-token `TradingBalance` (instrument admin = vault, id = `ammId`). Both amounts must already exist as funded `TradingBalance` contracts for the actor (3.4.1). If the venue cannot execute the order (insufficient balance, ratio, or `minOut`), it cancels it and the balances are untouched. The actor can withdraw a pending order with `DepositOrder_Withdraw`.

**Choice Signature**

```daml
nonconsuming choice AMMRules_CreateDepositOrder : ContractId DepositOrder
  with
    actor : Party
    ammId : Text
    amount1 : Decimal
    amount2 : Decimal
    minOut : Optional Decimal
    changeAmounts : Optional [InstrumentAmount]
      -- ^ Vestigial: pass None (ignored)
```

**Resulting Template (Queryable)**

```daml
template DepositOrder
  with
    actor : Party
      -- ^ The party requesting the deposit
    venue : Party
    vault : Party
    ammId : Text
      -- ^ Identifies which AMM pool this deposit is for
    amount1 : Decimal
      -- ^ Amount of instrument1 to deposit (must be pre-funded in TradingBalance)
    amount2 : Decimal
      -- ^ Amount of instrument2 to deposit (must be pre-funded in TradingBalance)
    createdAt : Time
    minOut : Optional Decimal
```

{% hint style="warning" %}
**NOTE:** `amount1` and `amount2` are amounts of the pool's `instrument1` and `instrument2` (`token1` and `token2` in `/pools`), and _**must**_ match the pool's current ratio to within 0.01%. The ratio is checked when the venue executes the order, not when it is created, and an out-of-ratio deposit is cancelled rather than adjusted. Fetch the current price with `GET /ratio/{tokenA}/{tokenB}`, or let the API compute aligned amounts for you with `GET /quoteLPDeposit/{tokenA}/{tokenB}`.
{% endhint %}

**3.4.3 - Withdraw orders: _removing liquidity from a pool_ (\~5 kB)**

This choice creates a `WithdrawOrder`. When the venue executes it, the LP tokens are burned and the withdrawn amounts of both pool instruments are credited to the actor's `TradingBalance` contracts. The LP tokens must already exist as a funded `TradingBalance` for the actor. If the venue cannot execute the order (insufficient LP tokens or slippage), it cancels it. The actor can withdraw a pending order with `WithdrawOrder_Withdraw`.

**Choice Signature**

```daml
nonconsuming choice AMMRules_CreateWithdrawOrder : ContractId WithdrawOrder
  with
    actor : Party
    ammId : Text
    lpTokenAmount : Decimal
    minAmount1 : Optional Decimal
    minAmount2 : Optional Decimal
```

**Resulting Template (Queryable)**

```daml
template WithdrawOrder
  with
    actor : Party
      -- ^ The party requesting the withdrawal
    venue : Party
    vault : Party
    ammId : Text
      -- ^ Identifies which AMM pool this withdrawal is for
    lpTokenAmount : Decimal
      -- ^ Amount of LP tokens to burn (from TradingBalance)
    minAmount1 : Optional Decimal
      -- ^ Minimum amount of instrument1 to receive
    minAmount2 : Optional Decimal
      -- ^ Minimum amount of instrument2 to receive
    createdAt : Time
```

**3.4.4 - Withdraw `TradingBalance`**

When the user is ready to take their tokens back to their wallet, they request a withdrawal with `AMMRules_RequestTradingBalanceWithdrawal`. Similar to deposits, this is a two-call sequence: fetch the `AMMRules` disclosure (3.4.1), then exercise the choice. No vault holdings are needed; the venue supplies them when it delivers.

The withdrawal is queued, not immediate. The choice debits the `TradingBalance` contracts straight away and creates a **`PhysicalSettlement`**: an obligation for the venue to deliver the amount to the recipient, which it fulfills in a later transaction.

**Exercising `AMMRules_RequestTradingBalanceWithdrawal`**

The choice supports both full and partial withdrawals, and consolidates fragmented balances in a single transaction.

```daml
nonconsuming choice AMMRules_RequestTradingBalanceWithdrawal : ContractId PhysicalSettlement
  with
    actor : Party
      -- ^ The TradingBalance owner
    tradingBalanceCids : [ContractId TradingBalance]
      -- ^ TradingBalances to withdraw from. All will be archived;
      --   they must share the same actor and instrument.
      --   Multi-input is supported : fragmented balances are consolidated.
    amount : Optional Decimal
      -- ^ None = withdraw the full consolidated balance.
      --   Some x = partial withdrawal; remainder stays in a new TradingBalance.
    recipient : Party
      -- ^ Who the venue delivers the tokens to
  controller actor
```

**Resulting Template (Queryable)**

```daml
template PhysicalSettlement
  with
    actor : Party
      -- ^ The party that requested the withdrawal
    recipient : Party
      -- ^ Who the venue delivers the tokens to
    venue : Party
    vault : Party
    instrument : InstrumentId
      -- ^ Instrument to deliver
    amount : Decimal
      -- ^ Amount to deliver, already debited from the TradingBalances
    createdAt : Time
    releasedToActor : Optional Bool
    ammId : Optional Text
      -- ^ None for trading balance withdrawals
```

The venue delivers by exercising `PhysicalSettlement_Redeem`, which archives the `PhysicalSettlement` in the same transaction as the transfer to the recipient. Only the actor and the venue can see the `PhysicalSettlement`. A successful exercise of the request choice does not mean the recipient has been paid.

{% hint style="warning" %}
**NOTE:** The venue delivers through the recipient's transfer pre-approval for the instrument. If the recipient has **no pre-approval**, the venue cannot deliver and the `PhysicalSettlement` stays pending until the recipient sets one up.
{% endhint %}

#### 3.5 - Monitor for filled orders

When the venue processes an order created from holdings (3.2), the order's archival, the consumption of its input(s), and the transfer of its output(s) all appear in the _same_ transaction. Archival alone does not mean the order filled, so check the choice that archived it: `Order_Complete` means it filled, while `Order_Cancel` means it was cancelled, with a `reason` text saying why. If the swap failed at execution, the same transaction refunds the input.

**Recommended**\
Use **PQS** (Participant Query Store) to subscribe to the relevant template streams.

**Without PQS**\
Query the participant's contracts endpoint directly for active contracts filtered by your actor. When the contract disappears, look up the transaction that archived it to see whether it was completed or cancelled; the order's output or refund is in that same transaction.

### 4.0 - Type Reference

This is a consolidated list of every choice and template you'll touch as an integrator. Most live on or are produced by the singleton `AMMRules` contract.

**Choices on `AMMRules`**

* `AMMRules_CreateSwapOrderFromHoldingsV2` → `AMMRules_CreateSwapOrderFromHoldingsV2_Result`
  * contains `ContractId V2.SwapOrder` + `[ContractId Holding]`
* `AMMRules_CreateSwapOrderV2` → `ContractId V2.SwapOrder` (orders with an explicit settlement mode)
* `AMMRules_CreateSwapOrderBatcher` → `AMMRules_CreateSwapOrderBatcher_Result` (one-time setup for bulk orders; then exercise `SwapOrderBatcher_CreateOrdersV2` on the batcher contract)
* `AMMRules_CreateDepositOrder` → `ContractId DepositOrder`
* `AMMRules_CreateWithdrawOrder` → `ContractId WithdrawOrder`
* `AMMRules_AddTradingBalance` → `ContractId TradingBalance`
* `AMMRules_RequestTradingBalanceWithdrawal` → `ContractId PhysicalSettlement`
* `AMMRules_CreateSettlementAllocation` → `AMMRules_CreateSettlementAllocation_Result`
  * contains `ContractId AllocationV2.Allocation` + change holdings
* `AMMRules_FundSettlementAllocation` → `AMMRules_FundSettlementAllocation_Result`

**Choices on other templates**

* `SwapOrderBatcher_CreateOrdersV2` → `SwapOrderBatcher_CreateOrdersV2_Result` (actor; on the `SwapOrderBatcher`)
* `SwapOrder_WithdrawV3` → `RefundResult` (actor; withdraws a pending V2 swap order)
* `DepositOrder_Withdraw` / `WithdrawOrder_Withdraw` → `()` (actor; withdraws a pending LP order)
* `SettlementWithdrawalRequest_Cancel` → `()` (actor; takes back a withdrawal request)
* `Order_Complete` / `Order_Cancel` (`TC.OrderV1.Order` interface) : exercised by the venue to archive an order that filled or was cancelled; `Order_Cancel` carries `reason : Text`

**Templates Produced**

* `SwapOrder` (V1, `TC.V4.SwapOrder`) : legacy swap order; not produced by the flows in this guide
* `SwapOrder` (V2, `TC.V4.SwapOrderV2`) : pending swap; archived when filled, cancelled, or withdrawn
* `DepositOrder` : pending LP deposit; archived when filled, cancelled, or withdrawn
* `WithdrawOrder` : pending LP redemption; archived when filled, cancelled, or withdrawn
* `TradingBalance` : per user, per instrument, collateral for liquidity deposit and withdrawal orders
* `PhysicalSettlement` : queued delivery from the vault, such as a trading balance withdrawal; the venue delivers it with `PhysicalSettlement_Redeem`
* `SwapOrderBatcher` : per user helper for creating V2 swap orders in bulk
* `Allocation` (token-standard-v2) : committed settlement allocation; archived and recreated with updated funding on every settlement and top-up
* `SettlementWithdrawalRequest` : request to release a committed allocation; created directly by the actor, archived when the venue fulfills or rejects it or the actor cancels it

**Enums**

* `SwapDirection` : `Long` (buy fixed amount) or `Short` (sell fixed amount)
* `SettlementMode` : `SettlementMode_TBTB` (trading-balance), `SettlementMode_TBPhysical recipient` (physical delivery), or `SettlementMode_DVP` (settle through committed allocations)
* `CancellationReason` : `InsufficientBalance`, `SlippageExceeded`, `PoolNotFound`, `PoolMigration`, or `VenueDecision`. Only the venue's direct `SwapOrder_CancelV3` takes it; batch cancellations archive orders with `Order_Cancel` and a free-text `reason` instead
* `RejectReason` : `RejectReason_PendingSettlements`, `RejectReason_InsufficientFunds`, or `RejectReason_OtherReason reason`
