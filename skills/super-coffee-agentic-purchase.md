---
name: Buy Super Coffee products as an agent via UCP/MCP
description: Search the Super Coffee storefront catalog, build a cart, create and fulfill a checkout, and complete payment with explicit human approval, using the store's Universal Commerce Protocol MCP endpoint.
api: mcp/super-coffee-mcp.yml
endpoint: https://www.drinksupercoffee.com/api/ucp/mcp
operations:
  - search_catalog
  - get_product
  - create_cart
  - update_cart
  - create_checkout
  - update_checkout
  - complete_checkout
  - get_order
generated: '2026-08-05'
method: generated
source: https://www.drinksupercoffee.com/llms.txt
---

# Buying from Super Coffee as an agent

Super Coffee's storefront (`www.drinksupercoffee.com`, a Shopify store) implements the
Universal Commerce Protocol (UCP) shopping service over MCP transport. Every tool below was
verified against a live anonymous `tools/list` on 2026-08-05 — see
`mcp/super-coffee-mcp-tools.json` for the exact input schemas.

## Before you start

- **Endpoint:** `POST https://www.drinksupercoffee.com/api/ucp/mcp`
- **Headers:** `Content-Type: application/json`, `Accept: application/json, text/event-stream`
- **No credentials are required** to discover tools, search the catalog, or build a cart.
- **Every** tool call requires a `meta.ucp-agent.profile` URI identifying you as the calling agent.
- Pass `context.address_country` (ISO 3166-1 alpha-2) and `context.currency` so pricing and
  availability are correct for your buyer.
- Confirm capabilities first with `GET https://www.drinksupercoffee.com/.well-known/ucp`.

## The hard rule: payment needs a human

`robots.txt` and `llms.txt` both state it plainly — **do not complete checkout, payment, or order
placement without an explicit, contemporaneous human approval step.** No scripted form fills, no
end-to-end flows that finalize payment on their own. If you cannot get approval at the moment of
payment, do not call `complete_checkout`; route the buyer through Shop Pay via
`https://shop.app/SKILL.md` instead.

## Steps

1. **Find the product.** Call `search_catalog` with a natural-language `query`, filter criteria, or
   both (at least one is required). Page with `catalog.pagination.cursor` and
   `catalog.pagination.limit` (default 10, minimum 1).
2. **Confirm the variant.** Call `get_product` with the identifier from step 1 to get exact pricing,
   real-time availability, and the variant set. Use `lookup_catalog` instead when you need to
   resolve several product or variant IDs in one request.
3. **Build the cart.** Call `create_cart` with the chosen variant and quantity; it returns a cart id
   in `gid://shopify/...` form. Adjust with `update_cart`, inspect with `get_cart`, and use
   `cancel_cart` if the buyer abandons.
4. **Open a checkout.** Call `create_checkout`. The response carries line items, totals, discounts
   and taxes.
5. **Fulfill.** Call `update_checkout` to set the shipping address and shipping method. This store
   allows a single shipping destination (`allows_multi_destination.shipping: false`) and one
   method combination (`[["shipping"]]`).
6. **Get approval, then complete.** Present the totals to your buyer and obtain explicit approval.
   Then call `complete_checkout` with a **required** `meta.idempotency-key` — a stable string you
   generate for this one purchase attempt. Reuse the same key on retry so a network failure never
   charges twice. The response returns the order ID and Thank You Page URL, or the errors hit.
7. **Confirm.** Call `get_order` with the returned order id to read back the completed order.

## Payment instruments

The store's UCP profile advertises three payment handlers: Google Pay (`gpay`), Shopify card
(`shopify.card`, accepting visa / master / american_express / discover / diners_club), and Shop Pay
(`shop_pay`). Pass instruments on `checkout.payment.instruments[]`, each with `id`, `handler_id`,
`type` (`card` for cards, `token` for wallets) and a `billing_address`.

## Failure handling

- **429** — the endpoint is rate limited per IP. Back off and retry.
- **Retrying `complete_checkout`** — always resend the identical `meta.idempotency-key`; do not mint
  a new one.
- The store publishes no enumerated error or decline-code reference, so treat JSON-RPC error objects
  as opaque and surface their message to the buyer rather than branching on codes.
