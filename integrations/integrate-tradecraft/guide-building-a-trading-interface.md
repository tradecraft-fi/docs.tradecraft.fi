---
icon: arrow-right-arrow-left
metaLinks:
  alternates:
    - /broken/spaces/dF47DwfCAGYAD1QW2BrD/pages/052OQCkPZuYJUDaAVK2G
---

# Guide: Building a Trading Interface

## Token Support

If your wallet fully supports CIP-56 tokens you can freely integrate all Tradecraft pools for trading, along with add and remove liquidity options (which return CIP-56 compliant Tradecraft LP tokens).

If your wallet supports only a whitelist of Canton tokens you will need to manually limit your interface to only allow trading of your wallet's supported tokens.

## Pools API

To build a trading interface that supports all pools listed on Tradecraft, start by using the [Broken link](/broken/spaces/CCRi0UxKmgIrQPvrXtbk/pages/7c306ffd23689acb6bf8bb749468518a6ee2dbe1#get-pools "mention") API to fetch a list of all pools. Note that multi-hop trades are not yet supported.

To get a list of tradable token pairs for a given token, run a reduce function on the list of pools and construct a mapping of each unique token to a list of other tokens. Every time a token appears in a pool entry, add its paired token to its list of tradable tokens. With the two pools we have online now, your output would look like this:

```javascript
const tradableTokens = {
    'CC': ['USDCx', 'CBTC'],
    'USDCx': ['CC'],
    'CBTC': ['CC']
}
```

Now you can create a widget with token drop-downs similar to a standard swap widget that looks something like the below image. Each entry in `tradableTokens` can be added to the first drop-down, and the second drop-down can be populated with the array value for that token.

<figure><img src="../.gitbook/assets/Screenshot 2026-01-29 at 11.29.19 AM.png" alt=""><figcaption></figcaption></figure>

## Quote API

{% hint style="warning" %}
**Important**

While the API is currently named "quote", these quotes should be presented to users as _estimates_. Since the on-chain portion of a trade is only initiated after the user sends funds and after Tradecraft gets to their order and begins execution, the price returned from the quote API may not be the price the user receives, unless a limit is specified for the trade.
{% endhint %}

There are two API's for getting trade quotes–one to specify amount in, and one to specify amount out. Use either or both as your interface requires. For example in the widget above, the API for amount in would be used when the user types in the first input, showing response in the second, and when the user types in the second input, the response would be shown in the first input. Both APIs can be found on the [Broken link](/broken/spaces/CCRi0UxKmgIrQPvrXtbk/pages/93773e026af2d87558fcac79bf11a46fe307ab44 "mention") API page.

## Initiating a Trade

There are two options for executing trades, both introduced on the [..](../ "mention") page.

For the Daml package integration option, see [guide-dar-integration](guide-dar-integration/ "mention").

For Pool Addresses, see [Pool Addresses](https://app.gitbook.com/o/yCv1wBUdpK0UqbPlBfEl/s/dF47DwfCAGYAD1QW2BrD/ "mention").
