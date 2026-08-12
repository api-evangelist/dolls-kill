---
name: Discover the Dolls Kill agent commerce surface
description: Walk the Dolls Kill discovery chain from robots.txt to a live MCP session, confirm the UCP version and capabilities, and establish the agent profile required before any tool can be called.
api: mcp/dolls-kill-mcp.yml
operations: [initialize, tools/list]
method: generated
generated: '2026-08-12'
---

# Discover the Dolls Kill agent commerce surface

Dolls Kill has no developer portal and no OpenAPI. Discovery runs entirely through
well-known documents. Do this once per session before transacting.

## 1. Read the agent instructions

- `GET https://www.dollskill.com/robots.txt` — the header comments name every agent
  entry point.
- `GET https://www.dollskill.com/agents.md` — the canonical agent-facing description
  of the store. `https://www.dollskill.com/llms.txt` mirrors it byte for byte; read
  either, not both.

## 2. Read the UCP merchant profile

`GET https://www.dollskill.com/.well-known/ucp`

Confirm before you continue:

- `ucp.version` is `2026-04-08` (`2026-01-23` is also accepted).
- `services["dev.ucp.shopping"]` lists an `mcp` transport entry.
- `capabilities` includes the ones your flow needs: `dev.ucp.shopping.catalog.search`,
  `dev.ucp.shopping.catalog.lookup`, `dev.ucp.shopping.cart`,
  `dev.ucp.shopping.checkout`, `dev.ucp.shopping.fulfillment`,
  `dev.ucp.shopping.discount`, `dev.ucp.shopping.order`.
- `payment_handlers` — `com.google.pay`, `dev.shopify.card`, `dev.shopify.shop_pay`.

**Known discrepancy:** the discovery document advertises the MCP endpoint as
`https://dolls-test.myshopify.com/api/ucp/mcp`. Call
`https://www.dollskill.com/api/ucp/mcp` instead — that is the host that actually
answers. Do not follow the `myshopify.com` hostname.

## 3. Open the MCP session

`POST https://www.dollskill.com/api/ucp/mcp`
with `Content-Type: application/json` and
`Accept: application/json, text/event-stream`.

```json
{"jsonrpc":"2.0","id":1,"method":"initialize",
 "params":{"protocolVersion":"2025-06-18","capabilities":{},
           "clientInfo":{"name":"your-agent","version":"1.0"}}}
```

Expect `serverInfo` `{"name":"universal-commerce","version":"0.1.0"}` and
`protocolVersion` `2025-06-18`.

Then `tools/list` for the 13 tools and their full JSON Schema input contracts. Both
of these answer anonymously — no credential of any kind.

## 4. Stand up your agent profile — required before any tool call

`tools/call`, `prompts/list` and `resources/list` all fail anonymously with:

```json
{"code":-32001,"message":"UCP discovery failed",
 "data":{"code":"invalid_profile_url","content":"Unable to fetch agent profile: Missing profile uri"}}
```

Every tool's `inputSchema` requires `meta.ucp-agent.profile` — a URI Dolls Kill will
fetch to identify you. Serve that document at a public HTTPS URL returning 2xx, then
pass it on every call. A URI the merchant cannot fetch returns
`data.code: profile_unreachable`.

## Notes

- Version is echoed on `x-shopify-ucp-mcp-api-version`; correlate support issues with
  `x-request-id`.
- No `RateLimit-*` or `Retry-After` headers exist. You get
  `shopify-complexity-score` as a cost signal and a bare 429 on exhaustion — back off
  exponentially.
