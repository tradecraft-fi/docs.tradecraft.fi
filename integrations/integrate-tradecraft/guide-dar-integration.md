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
The protocol contract that exposes every order-creation choice. All choices on it are _nonconsuming_, so the same contract is reused across every interaction. You will need its disclosure once per pool.

**`SwapOrder` - `DepositOrder` - `WithdrawOrder`** : _the order types_\
Each is created by exercising a choice on AMMRules. Swap orders either settle immediately from wallet holdings (3.2) or queue against committed allocations and settle when the venue fills them (3.3).

**`Allocation`** ([token-standard-v2, committed](https://github.com/canton-foundation/cips/blob/main/cip-0112/cip-0112.md#416-committed-allocations-for-prefunded-trading-and-iterated-settlement)) : _per user, per pool, per instrument_\
A committed allocation that pre-funds swap order queueing. On every fill, settlement moves tokens between the user's and the vault's allocations. Each fill archives the allocation contract and recreates it with updated funding, so track allocations by their metadata rather than by contract ID. See section 3.3.

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
**NOTE:** An [API endpoint for the token allocation factory](https://docs.tradecraft.fi/api/routes/tokens#get-allocation-factory-token) may be required for some of the steps described below. See [https://docs.tradecraft.fi/api](https://docs.tradecraft.fi/api) for more information.
{% endhint %}

#### 3.1 - Discover trading pairs

Fetch the list of every active AMM pool. The response gives you the `ammId` you'll need for every subsequent order.

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

Without it, **the swap still executes**: the input is consumed on fill, but **NO funds are returned to the wallet**. If the order fails or is cancelled, pre-approval is required to receive the input tokens back. In both scenarios, missing pre-approval = **loss of funds**.
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
**Reminder: no pre-approval on the output token = the swap will execute but no funds will be returned.**
{% endhint %}

#### 3.3 - "Queued" swap orders: _committed allocations for regular trading_

**Best for inventory providers and market makers who anticipate making regular trades against the same pool.**

The actor pre-funds a **committed allocation** for each pool instrument, then queues swap orders against them. Orders settle through the allocations trade after trade, and no transfer pre-approval is required.

**3.3.1 - Committed allocations**

A committed allocation is a token-standard-v2 **`Allocation`** that locks a user's holdings for settlement by the venue. On every fill, settlement moves tokens between the user's and the vault's allocations.

Tradecraft settlement allocations always have the same shape:

* `committed = True` - funds stay locked until the venue settles or cancels the allocation.
* `nextIterationFunding` enabled - the same allocation serves fill after fill. Every fill archives the allocation contract and recreates it with updated funding, so track allocations by their metadata rather than by contract ID.
* `settlementDeadline = None` and `executors = [venue]` - the venue is the sole settlement executor.
* Metadata binds the allocation to one pool (`tradecraft.fi/amm-id`) and one instrument (`tradecraft.fi/instrument-id`).

{% hint style="danger" %}
**Critical: a committed allocation with no settlement deadline cannot be withdrawn by its owner.** Only the venue (the sole executor) can settle or cancel it. Funds leave a committed allocation only through swap settlement or a venue-side cancellation - treat allocations as working capital, not storage, and coordinate with the venue before committing large balances.\
\
In a future release, we will add a Daml choice to `AMMRules` that allows a user to cancel an allocation, provided they have no executed-but-unsettled trades pending.
{% endhint %}

{% hint style="warning" %}
**Keep exactly one allocation per pool and instrument.** The venue selects at most one allocation per pool and instrument when settling your fills. Create the allocation once and top it up with `AMMRules_FundSettlementAllocation` (3.3.3) rather than creating additional shards.
{% endhint %}

**3.3.2 - Create a committed allocation**

A queued trade requires **two allocations: one for the input instrument and one for the output instrument**. Both must exist before you submit an order (3.3.4); the venue cannot settle a fill without both.

For the instrument you intend to _sell_, pass `initialFunding` and the holdings that fund it. For the instrument you intend to _receive_, the allocation may start empty (`initialFunding = None`, no holdings) or funded, as you prefer. Either way it is **required**, since it is what receives the settlement proceeds.

You will need:

* `ammCid` - the pool's V4 AMM contract ID, from the `amm` disclosure returned by `GET /disclosures/{tokenA}/{tokenB}`.
* `authorizerAccount` - the actor's token-standard (HoldingV2) account. Its owner must be the actor.
* `allocationFactoryCid` + `extraArgs` - the instrument's V2 allocation factory and its choice context, served by the token registry's utility service at `POST /registry/allocation-instruction/v2/allocation-factory`. The request body wraps the allocate choice you intend to exercise: `{"choiceArguments": <AllocationFactory_Allocate arguments>, "excludeDebugFields": false}`. The response is `{"factoryId", "choiceContext"}` - pass `factoryId` as `allocationFactoryCid` and build `extraArgs` from `choiceContext`.

{% hint style="warning" %}
**NOTE:** The [`GET /allocation-factory/{token}`](https://docs.tradecraft.fi/api/routes/tokens#get-allocation-factory-token) endpoint referenced in 3.2 serves the V1 allocation factory used by immediate swaps. It does not return the V2 factory this flow requires.
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
      -- ^ Some x = fund the allocation with x from inputHoldingCids.
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

This choice adds funds to a live allocation from the actor's spare holdings in a single atomic transaction. You will additionally need the `SettlementFactory` contract and its choice context for the instrument's admin, served by the same utility service at `POST /registry/allocation/v2/settlement-factory`. The request has the same shape as the allocation-factory call: the body wraps the `SettlementFactory_SettleBatch` choice arguments, and the response's `factoryId` and `choiceContext` provide `settlementFactoryCid` and `settlementExtraArgs`.

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
      -- ^ Includes the topped-up allocation's successor contract ID
```

**3.3.4 - Submit a queued swap order** **(\~4.5 kB)**

A queued V2 swap order is pure intent; no tokens move at submission. When the venue fills the order, settlement debits the actor's input allocation and credits their output allocation. No wallet transfer pre-approval is needed. The output lands in the actor's allocation, not their wallet.

{% hint style="warning" %}
**Prerequisite:** Before submitting a queued swap, the actor must hold a committed allocation for the _input_ instrument funded with at least the swap input amount, and a committed allocation for the _output_ instrument (an empty receiving allocation is fine). If funding is insufficient or an allocation is missing at execution time, the order fails without touching pool state.
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
      --   allocations. A bare None with the three fields above unset
      --   selects the same mode, but prefer being explicit.
  controller actor
```

**Lifecycle**

1. The order is created (template `TC.V4.SwapOrderV2.SwapOrder`, signatories `vault` + `actor`) and is queryable like any other order.
2. When the venue fills the order, it is archived - monitor for archival as in 3.5. An order that cannot be filled (slippage, insufficient allocation funding) is cancelled with a structured `CancellationReason` on-chain.
3. Settlement moves tokens between the actor's and the vault's committed allocations. The actor's allocations are archived and recreated with updated funding - this is why tracking them by metadata matters.

To abandon a pending order, the actor exercises `SwapOrder_WithdrawV3`. Because no funds have moved yet, there is nothing to refund - the input stays in the actor's committed allocation throughout.

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

{% hint style="info" %}
**NOTE:** `CreateOrdersV2` creates `Short` (exact-input) orders only, and every order in the batch sells the same instrument. The orders carry no attached allocation and no explicit settlement mode, so they queue against the actor's committed allocations exactly like the orders of 3.3.4 - the same funding prerequisites apply.
{% endhint %}

#### 3.4 - Liquidity operations: _adding and removing pool liquidity_

**Best for liquidity providers who want to supply capital to a pool and earn fees.**

Deposit and withdrawal orders draw from and settle into `TradingBalance` contracts (vault-held collateral of a single token, owned by the user), so funding and managing `TradingBalance` contracts is part of every LP workflow.

**3.4.1 - Add `TradingBalance` (\~10 kB)**

Depositing tokens into a `TradingBalance` is a two-call sequence: fetch the disclosure for the `AMMRules` contract, then exercise the deposit choice.

**Fetching the `AMMRules` disclosure**

The path segments are the two instruments of the pool you intend to deposit into. The example below targets the CBTC/CC pool.

```shell
$ curl https://api.tradecraft.fi/v1/disclosures/CBTC/CC | jq ".amm_rules"
```

**Exercising `AMMRules_AddTradingBalance`**

With the disclosure in hand, exercise the add choice. The user is the actor; their wallet's allocation context provides the tokens.

**Choice Signature**

```daml
nonconsuming choice AMMRules_AddTradingBalance : ContractId TradingBalance
  with
    actor : Party
      -- ^ The user depositing tokens
    allocationContext : (ContractId Allocation, ExtraArgs)
      -- ^ Allocation of tokens the actor is transferring to the vault
    existingBalances : [ContractId TradingBalance]
      -- ^ Existing balances to consolidate with this one
  controller actor
```

{% hint style="info" %}
**TIP:** Avoid UTXO fragmentation. Pass _**every**_ known `TradingBalance` contract ID for the given asset into `existingBalances` on every call. They will be consolidated atomically into a single new balance, keeping your contract set tidy and reducing downstream gas costs.
{% endhint %}

**3.4.2 - Deposit orders: _adding liquidity to a pool_ (\~5 kB)**

This choice deposits liquidity into a pool and mints LP tokens into a `TradingBalance`. Both `amount1` and `amount2` must already exist as funded `TradingBalance` contracts for the actor (3.4.1).

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
**NOTE:** `amount1` and `amount2` _**must**_ be aligned with the current ratio of the pool. Fetch the current price with `GET /ratio/{tokenA}/{tokenB}`, or let the API compute aligned amounts for you with `GET /quoteLPDeposit/{tokenA}/{tokenB}`.
{% endhint %}

**3.4.3 - Withdraw orders: _removing liquidity from a pool_ (\~5 kB)**

This choice removes liquidity from a pool and puts the withdrawn tokens into a `TradingBalance`. The LP tokens must already exist as a funded `TradingBalance` for the actor.

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

**3.4.4 - Withdraw `TradingBalance` (\~10 kB)**

When the user is ready to take their tokens back to their wallet, return them using `AMMRules_WithdrawTradingBalance`. Similar to deposits, this is a two-call sequence.

**Fetching the vault holdings**

The withdrawal choice needs a list of holding contract IDs from the vault. Request them for the specific token, amount, and recipient.

```shell
$ curl -s -X POST \
    https://api.tradecraft.fi/v1/vault-holdings \
    -H 'Content-Type: application/json' \
    -d '{
      "token": "CC",
      "amount": 1.0,
      "vault": "cs-vault::1220b4cd6098eebafd4c88efd2b3986e86542bdc391060675432cd195ad26bcf013b",
      "receiver": "YourParty::YourNode"
    }' | jq
```

**Exercising `AMMRules_WithdrawTradingBalance`**

The choice supports both full and partial withdrawals, and consolidates fragmented balances in a single transaction.

```daml
nonconsuming choice AMMRules_WithdrawTradingBalance : TransferInstructionResult
  with
    actor : Party
    tradingBalanceCids : [ContractId TradingBalance]
      -- ^ TradingBalances to withdraw from. All will be archived;
      --   they must share the same actor and instrument.
      --   Multi-input is supported : fragmented balances are consolidated.
    amount : Optional Decimal
      -- ^ None = withdraw the full consolidated balance.
      --   Some x = partial withdrawal; remainder stays in a new TradingBalance.
    recipient : Party
      -- ^ Who to send the tokens to
    instrumentTransferInfo : InstrumentTransferInfo
      -- ^ Transfer infrastructure (TransferFactory, expectedAdmin, etc.)
    holdingCids : [ContractId Holding]
      -- ^ The holdings returned by /vault-holdings
```

{% hint style="info" %}
**NOTE:** If the recipient has a transfer pre-approval for the output instrument, the tokens arrive in the recipient's wallet with no further action. If the recipient has **no pre-approval**, the tokens will appear as a transfer offer which the recipient must accept.
{% endhint %}

#### 3.5 - Monitor for filled orders

When an order created from holdings (3.2) is filled by the venue, the order's archival, the consumption of its input(s), and the creation of its output(s) all appear in the _same_ transaction. Detection is therefore as simple as watching for the order's archival.

**Recommended**\
Use **PQS** (Participant Query Store) to subscribe to the relevant template streams.

**Without PQS**\
Query the participant's contracts endpoint directly for active contracts filtered by your actor. When the contract disappears, the fill has occurred; the order's outputs are created in the same transaction.

### 4.0 - Type Reference

This is a consolidated list of every choice and template you'll touch as an integrator. Each lives on or is produced by the singleton `AMMRules` contract.

**Choices on `AMMRules`**

* `AMMRules_CreateSwapOrderFromHoldingsV2` → `AMMRules_CreateSwapOrderFromHoldingsV2_Result`
  * contains `ContractId V2.SwapOrder` + `[ContractId Holding]`
* `AMMRules_CreateSwapOrderV2` → `ContractId V2.SwapOrder` (orders with an explicit settlement mode)
* `AMMRules_CreateSwapOrderBatcher` → `AMMRules_CreateSwapOrderBatcher_Result` (one-time setup for bulk orders; then exercise `SwapOrderBatcher_CreateOrdersV2` on the batcher contract)
* `AMMRules_CreateDepositOrder` → `ContractId DepositOrder`
* `AMMRules_CreateWithdrawOrder` → `ContractId WithdrawOrder`
* `AMMRules_AddTradingBalance` → `ContractId TradingBalance`
* `AMMRules_WithdrawTradingBalance` → `TransferInstructionResult`
* `AMMRules_CreateSettlementAllocation` → `AMMRules_CreateSettlementAllocation_Result`
  * contains `ContractId AllocationV2.Allocation` + change holdings
* `AMMRules_FundSettlementAllocation` → `AMMRules_FundSettlementAllocation_Result`

**Templates Produced**

* `SwapOrder` : pending swap; archived on fill
* `SwapOrder` (V2, `TC.V4.SwapOrderV2`) : pending swap with an explicit settlement mode; archived on fill
* `DepositOrder` : pending LP deposit; archived on fill
* `WithdrawOrder` : pending LP redemption; archived on fill
* `TradingBalance` : per user, per instrument, collateral for liquidity deposit and withdrawal orders
* `SwapOrderBatcher` : per user helper for creating V2 swap orders in bulk
* `Allocation` (token-standard-v2) : committed settlement allocation; archived and recreated with updated funding on every fill

**Enums**

* `SwapDirection` : `Long` (buy fixed amount) or `Short` (sell fixed amount)
* `SettlementMode` : `SettlementMode_TBTB` (trading-balance), `SettlementMode_TBPhysical recipient` (physical delivery), or `SettlementMode_DVP` (settle through committed allocations)
* `CancellationReason` : `InsufficientBalance`, `SlippageExceeded`, `PoolNotFound`, `PoolMigration`, or `VenueDecision`
