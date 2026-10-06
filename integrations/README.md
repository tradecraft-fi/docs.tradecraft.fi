---
description: >-
  Build apps that surface trading and LP features, or use Tradecraft liquidity
  as a building block for more complex products.
icon: object-union
metaLinks: {}
---

# Integration

Tradecraft is a permissionless decentralized exchange, open for integration by wallets and other dApps.

{% hint style="warning" %}
## Dependencies

Both of the integration options below require the DA Utility package version 0.14.4. If you do not have it installed when making trades you will be unable to receive the incoming tokens. Additionally you may be automatically put on a temporary blacklist to prevent further trades from being initiated. If you think you may have been blacklisted, [contact us](https://docs.tradecraft.fi/support/get-help).
{% endhint %}

We offer two integration options, with the tradeoffs shown below:

<table data-card-size="large" data-view="cards"><thead><tr><th></th><th></th><th></th><th></th></tr></thead><tbody><tr><td><h3>Pool Addresses</h3></td><td>Interact using standard Daml <em>TransferOffers</em> and <em>Preapprovals</em>.</td><td><p><i class="fa-check" style="color:$warning;">:check:</i> Trade.</p><p><i class="fa-check" style="color:$warning;">:check:</i> Deposit and withdraw liquidity.</p><p><i class="fa-x" style="color:$danger;">:x:</i> More expensive.</p><p><i class="fa-x" style="color:$danger;">:x:</i> Slower transactions.</p><p><i class="fa-x" style="color:$danger;">:x:</i> Lower TPS capacity.</p><p><i class="fa-x" style="color:$danger;">:x:</i> No flash accounting features.</p><p><i class="fa-x" style="color:$danger;">:x:</i> No built-in leg matching.</p><p><i class="fa-x" style="color:$danger;">:x:</i> No transaction failure insight.</p><p><i class="fa-check" style="color:$warning;">:check:</i> Usable with basic transfer API access (no validator node required).</p></td><td><a href="https://app.gitbook.com/o/yCv1wBUdpK0UqbPlBfEl/s/dF47DwfCAGYAD1QW2BrD/">Documentation</a></td></tr><tr><td><h3>Daml Package</h3></td><td>Interact using custom choices from our Tradecraft Daml package.</td><td><p><i class="fa-check" style="color:$warning;">:check:</i> Trade.</p><p><i class="fa-check" style="color:$warning;">:check:</i> Deposit and withdraw liquidity.</p><p><i class="fa-check" style="color:$warning;">:check:</i> Cheaper transactions.</p><p><i class="fa-check" style="color:$warning;">:check:</i> Faster transactions.</p><p><i class="fa-check" style="color:$warning;">:check:</i> Higher TPS capacity.</p><p><i class="fa-check" style="color:$warning;">:check:</i> Flexible flash accounting features.</p><p><i class="fa-check" style="color:$warning;">:check:</i> Leg matching built in.</p><p><i class="fa-check" style="color:$warning;">:check:</i> Transaction failure insight.</p><p><i class="fa-x" style="color:$danger;">:x:</i> Requires a validator node to use.</p></td><td><a href="integrate-tradecraft/guide-dar-integration/">Documentation</a></td></tr></tbody></table>

Aside from the above documentation, other helpful documentation includes:

[guide-building-a-trading-interface.md](integrate-tradecraft/guide-building-a-trading-interface.md "mention")

[HTTP API Docs](https://app.gitbook.com/o/yCv1wBUdpK0UqbPlBfEl/s/CCRi0UxKmgIrQPvrXtbk/ "mention")
