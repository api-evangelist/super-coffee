---
name: Read the Super Coffee catalog without transacting
description: Browse and query Super Coffee product, pricing and availability data using the anonymous UCP/MCP catalog tools or the store's published read-only JSON endpoints, without building a cart or touching checkout.
api: mcp/super-coffee-mcp.yml
endpoint: https://www.drinksupercoffee.com/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-08-05'
method: generated
source: https://www.drinksupercoffee.com/llms.txt
---

# Reading the Super Coffee catalog

Use this when you need product, price or availability data and are **not** transacting. Nothing here
requires credentials.

## Option A — the MCP catalog tools (preferred, structured)

`POST https://www.drinksupercoffee.com/api/ucp/mcp` with `Content-Type: application/json` and
`Accept: application/json, text/event-stream`. Every call needs `meta.ucp-agent.profile`.

- **`search_catalog`** — natural-language `query`, filter criteria, or both; at least one is
  required. Page with `catalog.pagination.cursor` and `catalog.pagination.limit` (default 10,
  minimum 1). A boolean availability filter defaults to `true`, returning only sale-ready items —
  set it false if you want out-of-stock products too.
- **`lookup_catalog`** — resolve several product or variant IDs (`gid://shopify/Product/...`) in a
  single round trip.
- **`get_product`** — full detail for one identifier: variants, exact pricing, real-time
  availability, and interactive option selection.

Pass `context.address_country` and `context.currency` on every call; pricing and availability are
region-dependent.

## Option B — the published read-only HTTP endpoints

`llms.txt` documents these for agents that only need to read:

| Purpose | Request |
|---|---|
| All products | `GET /collections/all` |
| Product page | `GET /products/{handle}` |
| Product JSON | `GET /products/{handle}.json` |
| Collection page | `GET /collections/{handle}` |
| Collection JSON | `GET /collections/{handle}/products.json` |
| Search | `GET /search?q={query}&type=product` |
| Sitemap | `GET /sitemap.xml` |

## Boundaries to respect

- `robots.txt` **disallows** `/cart.js` and `/recommendations/products` for agents and directs you to
  UCP/MCP for anything cart-shaped. Do not scrape those.
- Do not enter the checkout flow from this skill. If the buyer wants to purchase, switch to
  `super-coffee-agentic-purchase.md`, which carries the human-approval and idempotency rules.
- Back off on `429`; the MCP endpoint is rate limited per IP.
