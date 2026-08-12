---
name: deinde-agentic-purchase
description: Search Debut's DEINDE skincare catalog, build a cart, and drive a checkout to
  the point of buyer approval using the store's Universal Commerce Protocol MCP endpoint.
api: DEINDE Commerce (UCP MCP)
endpoint: https://www.deinde.com/api/ucp/mcp
transport: mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - update_cart
  - get_cart
  - create_checkout
  - update_checkout
  - complete_checkout
  - get_order
generated: '2026-08-12'
method: generated
source: mcp/debut-ucp-mcp-tools.json (live tools/list, 2026-08-12) + https://www.deinde.com/llms.txt
---

# Buying from DEINDE as an agent

Every tool name and argument below is read from the live `tools/list` response captured in
`mcp/debut-ucp-mcp-tools.json`. Nothing here is invented. If a step needs a value the store
does not publish, it says so.

## Before you start

- **Endpoint**: `POST https://www.deinde.com/api/ucp/mcp`, JSON-RPC 2.0.
- **Headers**: `Content-Type: application/json` and `Accept: application/json, text/event-stream`.
  Omitting `Accept` gets a `406`.
- **Agent identity is mandatory.** Every tool call requires `meta["ucp-agent"].profile` — a
  URI pointing at your published UCP agent profile. The merchant fetches it and checks the
  content type. A page it cannot fetch, or an HTML page where a profile document was
  expected, returns JSON-RPC `-32001` with `data.code = profile_malformed` and HTTP `422`.
- **No API key, no signup.** Catalog, cart and checkout construction are anonymous.
- **The store recommends the Shop skill.** DEINDE's own `llms.txt` tells personal shopping
  agents to prefer `https://shop.app/SKILL.md` over scripting the storefront. Use this skill
  when you need the merchant's own UCP surface directly; use the Shop skill when you want
  cross-store checkout through Shop Pay.

## 1. Confirm the surface

`GET https://www.deinde.com/.well-known/ucp` and read `ucp.version`. Confirm the version you
intend to speak is in `supported_versions` (`2026-04-08` and `2026-01-23` at the time of
writing) and that `services["dev.ucp.shopping"]` still advertises the `mcp` transport.

## 2. Find the product — `search_catalog`

Pass `catalog.query` (natural language) and/or `catalog.filters`; at least one is required.
Always pass `catalog.context.address_country` (ISO 3166-1 alpha-2) and
`catalog.context.currency` (ISO 4217) — the store's instructions say pricing and
availability depend on them.

Results are paginated with an opaque cursor. To get more, send back
`catalog.pagination.cursor` from the previous response. No page size and no total count are
exposed, so do not build UI that promises either.

**Prices are integers in minor units** paired with a currency code: `{"amount": 2500,
"currency": "USD"}` is $25.00. Convert before quoting a buyer. Zero-decimal currencies such
as JPY are already whole units.

Use `lookup_catalog` when you already hold identifiers for several products or variants, and
`get_product` when you need the full detail of exactly one.

## 3. Build the cart — `create_cart`, `update_cart`, `get_cart`

`create_cart` takes a `cart` object; `update_cart` and `get_cart` take the returned `id`.
`cancel_cart` discards it.

**There is no idempotency key on any cart tool.** If a `create_cart` call times out, you
cannot safely retry it — you may create a second cart. Read `get_cart` before retrying, and
treat cart creation as at-most-once from your side.

## 4. Open the checkout — `create_checkout`, `update_checkout`

`create_checkout` returns line items, totals, discounts and taxes. Use `update_checkout` to
set the shipping address and fulfillment method, and to attach a payment instrument under
`checkout.payment.instruments[]` (each needs `id`, `handler_id` and `type` — `card` for
cards, `token` for wallet payments such as Google Pay or Apple Pay). The handlers this store
accepts are listed in `payment_handlers` in `/.well-known/ucp`.

Fulfillment is single-destination: the store's UCP profile declares
`allows_multi_destination.shipping = false` and only the `["shipping"]` method combination.
Do not attempt a split shipment.

`get_checkout` re-reads state at any point. `cancel_checkout` abandons it.

## 5. Complete — `complete_checkout`

This is the only tool that moves money and the only one with replay protection.

- `meta["idempotency-key"]` is **required**. Generate it once per purchase intent and reuse
  the same value on every retry of that intent. The retention window is not published, so do
  not assume a key is honoured indefinitely.
- **Do not call this without contemporaneous buyer approval.** The store's published rule is
  explicit: agents must not complete payment without explicit buyer consent. If you cannot
  get approval at the moment of payment, stop and route the purchase through Shop Pay via
  `https://shop.app/SKILL.md` instead.

## 6. Follow up — `get_order`

`get_order` returns order details by `id`. Buyer-scoped order history beyond the order you
just placed goes through the Shopify customer-account OAuth flow, not this endpoint — see
`authentication/debut-authentication.yml` and `scopes/debut-scopes.yml`.

## Failure handling

| What you see | What it means | What to do |
|---|---|---|
| `-32001` / `profile_malformed`, HTTP 422 | Your `meta["ucp-agent"].profile` could not be fetched or had the wrong content type | Publish a fetchable UCP agent profile and pass its URI |
| HTTP 406 | No acceptable response media type offered | Send `Accept: application/json, text/event-stream` |
| HTTP 429 | Per-IP rate limit — no threshold is published | Back off and retry; there is no `Retry-After` contract |

The store publishes no error reference beyond this. Payment, stock and address-validation
failure modes are undocumented — handle them defensively.

## Read-only alternative

If you only need to browse and never transact, skip MCP entirely. The store publishes
unauthenticated JSON: `GET /products/{handle}.json`,
`GET /collections/{handle}/products.json`, `GET /search?q={query}&type=product`, and
`GET /sitemap_agentic_discovery.xml`.

## What this is not

This is a Shopify-hosted commerce surface on Debut's brand domain. Debut Biotechnology
publishes no first-party API — no OpenAPI, no developer portal, and nothing that exposes its
BeautyORB ingredient-discovery platform, its skin-genomics data, or its formulation pipeline.
Do not expect this endpoint to answer anything about the science.
