---
name: shopify-checkout-recs-extension
description: Use when building a Shopify Checkout UI Extension that shows product recommendations or upsells from a third-party API (Rebuy, Nosto, Algolia, a custom recs service) inside checkout, especially when the extension must call an external service, keep an API key secret, or add a recommended product to the cart.
---

# Shopify Checkout Recommendations via Backend Proxy

## Overview

A checkout UI extension runs in a **sandboxed web worker with a public bundle** — it can hold no secrets and should never call a third-party recs API directly. The reusable pattern: the extension talks ONLY to a **backend proxy** you control (a Remix/Node app route). The proxy is the trust boundary that holds the API key, calls the third party, and returns a tiny card model. The add-to-cart happens client-side via `applyCartLinesChange` — no backend call.

```
Checkout UI Extension (React, network_access=true)
  → POST <backend>/api/recs   { productIds, country, maxCards }
       → third-party recs API   (server-side, API key from env)
       → normalize → validate variants on this store (Storefront API)
  ← [{ title, image, price, currencyCode, variantGid }]
  → render cards → applyCartLinesChange(addCartLine, variantGid)
```

## When to use

- Checkout/post-purchase extension needs data from any external service.
- The service requires a secret key (must stay server-side).
- You add recommended products to the cart from inside checkout.

Not for: theme/storefront recs (use the storefront directly); recs that need no external call.

## Non-obvious gotchas (the whole point of this skill)

| Gotcha | What to do |
|--------|-----------|
| Extension can't hold the key | Proxy holds it (`process.env`); extension only knows the proxy URL. |
| `network_access` | Set `network_access = true` in the extension TOML capabilities. **Do NOT allowlist domains in the TOML** — the capability alone is sufficient. |
| CORS **preflight** | Checkout calls cross-origin. The proxy must answer `OPTIONS` with `204` + CORS headers AND put CORS headers on the POST response. In Remix: a `loader()` returns 204+CORS, and the `action` short-circuits `OPTIONS`. |
| ID direction | Cart lines give product **GIDs** → strip to numeric to seed the recs API. Recs APIs return **numeric REST variant ids** → you build the GID yourself: `gid://shopify/ProductVariant/${id}`. `applyCartLinesChange` needs the **variant** GID, not product. |
| Stale / wrong-store variants | Recs payloads can include variants not published on this store or sold out. Validate every GID via Storefront `nodes(ids:)` + `availableForSale`, drop failures. |
| Currency | Use `@inContext(country:)` in the Storefront query to get presentment price in the buyer's currency; format with `i18n.formatCurrency`. |
| Accelerated checkout | Guard with `useInstructions().lines.canAddCartLine` — return `null` when false (Shop/Apple Pay can't add lines). |
| Refetch storms / races | Build a stable dep key from sorted productIds (`ids.sort().join(',')`); abort in-flight fetches with `AbortController`. |
| Fail soft | Proxy returns `[]` (HTTP 200) on any error; extension renders nothing rather than a broken UI. Never throw a 500 into checkout. |
| Don't recommend cart items | Filter recs whose productId is already in the cart. |
| Bundle ≤ 64 KB | No third-party SDK, no carousel libs in the extension — hand-roll minimal cards. |
| Dev tunnel rotates | ngrok/cloudflare URLs change each restart. Keep the backend URL in ONE `config.ts` constant, overridable by an extension setting. |
| Checkout-step placement | Requires Shopify **Plus** on the live store (Partner dev stores have it for previews). |

## Key code

**Proxy CORS + fail-soft (Remix route):**
```ts
const CORS = { "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "POST, OPTIONS", "Access-Control-Allow-Headers": "Content-Type" };
export function loader() { return new Response(null, { status: 204, headers: CORS }); } // preflight
export async function action({ request }: ActionFunctionArgs) {
  if (request.method === "OPTIONS") return new Response(null, { status: 204, headers: CORS });
  try {
    const { productIds = [] } = await request.json();
    const inCart = new Set(productIds);
    const cards = normalize(await fetchRecs(productIds)).filter(c => !inCart.has(c.productId));
    return json(await validateVariants(cards), { headers: CORS });
  } catch (e) { console.error(e); return json([], { headers: CORS }); } // never 500 into checkout
}
```

**ID conversions:**
```ts
const numericId = (gid: string) => Number(gid.split("/").pop());      // GID → seed recs API
const variantGid = `gid://shopify/ProductVariant/${variant.id}`;     // REST id → GID for cart
```

**Storefront variant validation (drop unbuyable, fix currency):**
```ts
// query ValidateVariants($ids:[ID!]!, $country:CountryCode!) @inContext(country:$country) {
//   nodes(ids:$ids){ ... on ProductVariant { id availableForSale price{ amount currencyCode } } } }
const buyable = new Map(nodes.filter(n => n?.availableForSale).map(n => [n.id, n]));
return cards.filter(c => buyable.has(c.variantGid))
  .map(c => ({ ...c, price: buyable.get(c.variantGid).price.amount }));
// If Storefront creds are absent, log a warning and trust the datasource (it's store-scoped).
```

**Extension shell:**
```tsx
if (!useInstructions().lines.canAddCartLine) return null;     // accelerated checkout
// useEffect: fetch(backendUrl, {POST, body:{productIds,maxCards,country}, signal}); key = sorted ids
const r = await applyCartLinesChange({ type: "addCartLine", merchandiseId: card.variantGid, quantity: 1 });
if (r.type === "error") setError(r.message);                  // applyCartLinesChange can fail
```

## File layout

| File | Role |
|------|------|
| `app/routes/api.<recs>.tsx` | Proxy: CORS, POST, orchestration, fail-soft |
| `app/rebuy/client.ts` | Server call to recs API (key from env) |
| `app/rebuy/normalize.ts` | Recs product → card model + variant GID |
| `app/rebuy/validateVariants.ts` | Storefront `availableForSale` check + presentment price |
| `extensions/<name>/shopify.extension.toml` | Target + `network_access = true` + settings |
| `extensions/<name>/src/Checkout.tsx` | Read lines → fetch → render → add |
| `extensions/<name>/src/useUpsells.ts` | Fetch + status machine + AbortController |
| `extensions/<name>/src/config.ts` | Single `BACKEND_URL` constant |

## Local dev

```bash
npm run dev                  # shopify app dev (note the local port)
ngrok http 3000             # tunnel; paste URL into config.ts BACKEND_URL (or the extension setting)
# Backend sanity check, no checkout needed:
curl -sX POST <tunnel>/api/recs -H 'Content-Type: application/json' -d '{"productIds":[],"maxCards":4}' | jq
```

Then open the dev-store checkout preview and drop the recommendation block into the order summary.

## Common mistakes

- Allowlisting the proxy domain in the TOML — unnecessary; `network_access = true` is enough.
- Calling the recs API straight from the extension (leaks the key, blocked by sandbox).
- Returning a 500 from the proxy on error — breaks checkout; return `[]` instead.
- Passing a product GID to `applyCartLinesChange` — it needs the **variant** GID.
- Trusting recs variant ids without Storefront validation — stale/cross-store ids fail silently at add time.
