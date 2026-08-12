---
name: Build a Dolls Kill cart and complete checkout with buyer approval
description: Take a resolved variant through cart, checkout, fulfillment and payment on the Dolls Kill UCP/MCP endpoint, honoring the idempotency key and the mandatory human-approval-before-payment rule.
api: mcp/dolls-kill-mcp.yml
operations: [create_cart, get_cart, update_cart, cancel_cart, create_checkout, get_checkout, update_checkout, complete_checkout, cancel_checkout, get_order]
method: generated
generated: '2026-08-12'
---

# Build a Dolls Kill cart and complete checkout

Prerequisites: a resolved **Product Variant ID**
(`dolls-kill-search-catalog`) and a reachable UCP agent profile URI
(`dolls-kill-discover-agent-surface`). Every call below carries
`meta.ucp-agent.profile`.

## The rule that governs this entire flow

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement
> automatically — no scripted form fills, browser automation, or end-to-end agent
> flows that finalize payment without an explicit, contemporaneous human approval
> step.

Dolls Kill publishes this in `robots.txt`, `agents.md` **and** `llms.txt`. It applies
to exactly one tool: `complete_checkout`. Everything before it is yours to automate.
If you cannot obtain contemporaneous buyer approval at the moment of payment, stop —
Dolls Kill's own instruction is to route the purchase through the Shopify shopping
skill at `https://shop.app/SKILL.md` instead.

## 1. Cart

- `create_cart` — build the cart. Pass `context.address_country` and
  `context.currency` up front; pricing and availability depend on them.
- `get_cart` / `update_cart` — read and revise. `update_cart` collapses eight
  underlying GraphQL mutations (line add/update/remove, note, attributes, buyer
  identity, discount codes, gift cards) into one call that takes the whole cart
  object.
- `cancel_cart` — abandon it.

Cart IDs are `gid://shopify/Cart/abc123?key=secret`. **The `?key=` fragment is a
capability secret — it is part of the identifier and must be kept with it, and out of
logs.**

## 2. Checkout

- `create_checkout` — either pass `line_items` directly, or pass `cart_id` to convert
  the cart you already built. When `cart_id` is present it wins: the merchant uses the
  cart's line items, context and buyer, and ignores those fields on the checkout
  payload.
- `update_checkout` — set `buyer.email` / `buyer.phone_number`,
  `fulfillment.methods[].destinations[]` for the shipping address, then
  `selected_destination_id` and `groups[].selected_option_id` for the chosen shipping
  method.
- `get_checkout` — re-read totals, taxes and discounts after every change.

Only prompt for a discount code if the buyer mentions having one; then set
`discounts.codes[]`. Codes are case-insensitive and each submission **replaces** the
previous set — send an empty array to clear.

**Fulfillment constraints, from the UCP profile:**
`allows_multi_destination.shipping` is `false` and the only allowed method
combination is `[["shipping"]]`. One destination, shipping only. Do not attempt a
split-shipment or pickup flow.

## 3. Payment — stop and ask

Attach a payment instrument under `checkout.payment.instruments[]`
(`id`, `handler_id`, `type`). Accepted handlers, from `/.well-known/ucp`:

| handler_id | type | notes |
|---|---|---|
| `gpay` | token | Google Pay; VISA, MASTERCARD, AMEX, DISCOVER; full billing address + phone required |
| `shopify.card` | card | visa, master, american_express, discover, diners_club |
| `shop_pay` | token | Shop Pay |

Present the final total, the shipping method and the last four digits to the buyer.
**Get an explicit yes, at this moment, for this amount.**

## 4. Complete

`complete_checkout` is the only irreversible tool and the only one that requires
`meta.idempotency-key`. Generate one key per completion attempt and **reuse the same
key on every retry** — a fresh key on a retry risks a duplicate order.

```json
{"jsonrpc":"2.0","id":9,"method":"tools/call",
 "params":{"name":"complete_checkout",
   "arguments":{"meta":{"ucp-agent":{"profile":"https://your-agent.example/profile"},
                        "idempotency-key":"<stable-uuid-per-attempt>"},
                "id":"gid://shopify/Checkout/...",
                "checkout":{...}}}}
```

Success returns the order ID and a Thank You Page URL — hand that URL to the buyer.

## 5. After the order

`get_order` reads order detail. Note that the Storefront GraphQL API has **no** order
query; order history for a signed-in customer lives behind the
`customer-account-api:full` / `customer-account-mcp-api:full` OAuth scopes at
`https://account.dollskill.com` (see `scopes/dolls-kill-scopes.yml`).

Returns are handled outside the API at `https://returns.dollskill.com/`; support is
at `https://help.dollskill.com/`.

## Error handling

Errors are JSON-RPC 2.0, **not** RFC 9457:
`{"error":{"code":-32001,"message":...,"data":{"code":...,"content":...,"continue_url":...}}}`.
`data.code` is the machine-readable reason; `continue_url` is a human-usable
storefront URL to hand the buyer when your agent is stuck. On HTTP 429, back off
exponentially — there is no `Retry-After`. Full catalog:
`errors/dolls-kill-problem-types.yml`.
