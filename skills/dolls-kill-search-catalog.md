---
name: Search and read the Dolls Kill catalog
description: Find products and resolve them to the purchasable variant ID, using the MCP catalog tools, the open Storefront GraphQL API, or the unauthenticated product JSON endpoints.
api: mcp/dolls-kill-mcp.yml
operations: [search_catalog, lookup_catalog, get_product]
graphql: [product, products, search, predictiveSearch, collection, collections, productRecommendations]
method: generated
generated: '2026-08-12'
---

# Search and read the Dolls Kill catalog

Three interchangeable read paths. Pick by what you already have.

## Path A — MCP tools (requires an agent profile)

Complete `dolls-kill-discover-agent-surface` first, then call with
`meta.ucp-agent.profile` set on every request.

- `search_catalog` — free-text product search. Pass buyer hints in
  `context.address_country` and `context.currency` so prices and availability come
  back correct.
- `lookup_catalog` — resolve several products or variants at once by identifier.
- `get_product` — full detail for one product.

## Path B — Storefront GraphQL (no credential required)

`POST https://www.dollskill.com/api/2026-04/graphql.json` with
`Content-Type: application/json`. Introspection and reads succeed with **no**
`X-Shopify-Storefront-Access-Token`.

Useful root fields: `product`, `products`, `search`, `predictiveSearch`,
`collection`, `collections`, `productRecommendations`, `productTags`, `productTypes`.

All list fields are Relay connections — page with `first`/`after` and read
`pageInfo.hasNextPage` and `pageInfo.endCursor`. Full SDL:
`graphql/dolls-kill-storefront.graphql`.

## Path C — Product JSON (no credential, no session)

Documented by Dolls Kill in `agents.md`:

- `GET /products.json`
- `GET /products/{handle}.json`
- `GET /collections/{handle}/products.json`
- `GET /search?q={query}&type=product`
- `GET /sitemap.xml`

Cheapest path for bulk browsing. Returns the whole product object with no field
selection.

## Resolve to a variant before you buy

**The purchasable unit is `ProductVariant`, not `Product`.** `create_checkout` and
`create_cart` require `line_items[].item.id` to be a Product Variant ID. From a
product, take `selectedOrFirstAvailableVariant`, or `variantBySelectedOptions` when
the buyer picked a size or colour. See `data-model/dolls-kill-data-model.yml`.

## Reading prices

Money is an integer in ISO 4217 **minor units** with a currency code —
`{"amount": 2500, "currency": "USD"}` is $25.00. Divide by 100 for two-decimal
currencies before quoting a buyer; JPY and other zero-decimal currencies are already
whole units. Dolls Kill presents 15 currencies and ships to 200+ countries.
