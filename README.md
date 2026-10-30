# shopify-checkout-extensions

Shopify **Checkout UI Extensions** that show third-party recommendations and upsells at
checkout, plus the Claude Code skill that captures how to build them.

Two parts, one repo:

| Part | Where | What it is |
|---|---|---|
| Shopify app + extension | `app/`, `extensions/rebuy-checkout-recs/` | Remix app (Shopify's app template) with a backend proxy for Rebuy, and the checkout extension that renders the cards |
| Claude Code plugin | `.claude-plugin/`, `skills/shopify-checkout-recs-extension/` | Skill with the architecture and gotchas, reusable for Rebuy, Nosto, Algolia or a custom recs service |

The extension runs in a sandboxed public bundle, so it only talks to the backend proxy. The
proxy holds the API key and validates variants. See [docs/REBUY_CHECKOUT_UPSELLS.md](docs/REBUY_CHECKOUT_UPSELLS.md).

## App and extension

Setup, `shopify app dev` and deploy steps are in Shopify's template guide, kept unchanged in
[docs/SHOPIFY_APP_TEMPLATE.md](docs/SHOPIFY_APP_TEMPLATE.md).

```bash
npm install
npm run dev      # shopify app dev
npm run deploy   # shopify app deploy
```

## Claude Code plugin

```
/plugin marketplace add Andrewrico/shopify-checkout-extensions
/plugin install shopify-checkout-recs
```

Once installed, the skill triggers on checkout-extension recommendation work, or run
`/shopify-checkout-recs-extension`.

## Automatic Shopify template updates

This repo tracks [Shopify/shopify-app-template-remix](https://github.com/Shopify/shopify-app-template-remix).
[`sync-shopify-template.yml`](.github/workflows/sync-shopify-template.yml) runs every Monday
(and on demand from the Actions tab):

- Nothing new upstream: it does nothing.
- New upstream commits that merge cleanly: it opens a pull request `sync/shopify-template`.
- Conflicts with our changes: it opens an issue listing the conflicting files.

It never pushes to `main`; you review and merge. This README is never overwritten by the sync.
Shopify has deprecated the Remix template in favor of
[React Router](https://github.com/Shopify/shopify-app-template-react-router), so updates may be sparse.

## License

MIT
