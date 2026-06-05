# shopify-checkout-recs

A Claude Code plugin that packages one skill, **`shopify-checkout-recs-extension`**, for
building Shopify **Checkout UI Extensions** that render third-party product
recommendations / upsells (Rebuy, Nosto, Algolia, or a custom recs service) inside
checkout.

The skill captures the reusable architecture and the non-obvious gotchas:

- The extension runs in a **sandboxed public bundle**, so it talks only to a **backend
  proxy** you control — the proxy holds the API key and is the trust boundary.
- CORS **preflight** (`OPTIONS` → 204) plus CORS headers on the POST response.
- Converting between Shopify **GIDs** and the **numeric REST ids** recs APIs return, in
  both directions.
- Validating recommended variants against the **Storefront API** (`availableForSale`) and
  fixing presentment currency with `@inContext`.
- `useInstructions().lines.canAddCartLine` guard for accelerated checkouts.
- Fail-soft (`[]` + HTTP 200) so a recs outage never breaks checkout.
- `network_access = true` (no domain allowlist needed), 64 KB bundle limit, tunnel-URL
  rotation in dev.

## Install

```bash
# Add this repo as a marketplace, then install the plugin:
/plugin marketplace add Andrewrico/shopify-checkout-recs-plugin
/plugin install shopify-checkout-recs
```

(Replace the GitHub path with wherever you host this repo.)

## Use

Once installed, the skill auto-triggers when you start a task about building a checkout
extension that pulls third-party recommendations. You can also invoke it explicitly:

```
/shopify-checkout-recs-extension
```

## Contents

```
.claude-plugin/
  plugin.json        # plugin manifest
  marketplace.json   # marketplace manifest (for `/plugin marketplace add`)
skills/
  shopify-checkout-recs-extension/
    SKILL.md         # the skill
```

## License

MIT
