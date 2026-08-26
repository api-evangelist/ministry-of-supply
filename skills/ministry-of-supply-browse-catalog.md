---
name: Browse the Ministry of Supply catalog
description: >-
  Search, filter and read Ministry of Supply products through the store's anonymous UCP/MCP endpoint,
  with the read-only storefront JSON as a fallback. No credentials, no purchase, no side effects.
api: mcp/ministry-of-supply-mcp.yml
endpoint: https://www.ministryofsupply.com/api/ucp/mcp
operations: [search_catalog, lookup_catalog, get_product]
generated: '2026-08-25'
method: generated
source: mcp/ministry-of-supply-mcp-tools-list.json
---

# Browse the Ministry of Supply catalog

Read-only. Every tool below is available anonymously — a `tools/list` POST to the endpoint returned
HTTP 200 with no credential on 2026-08-25.

## Before you call anything

Every tool requires a `meta` object carrying your agent identity:

```json
{"meta": {"ucp-agent": {"profile": "https://your-agent.example/ucp-profile"}}}
```

Pass buyer context on catalog calls so prices and availability are correct for the shopper:
`context.address_country` (ISO 3166-1 alpha-2) and `context.currency` (ISO 4217).

## Steps

1. **`search_catalog`** — pass `catalog.query` with the shopper's intent in their own words
   ("merino travel blazer", "wrinkle-free dress shirt"). Narrow with `catalog.filters.categories`,
   `catalog.filters.price.min` / `.max` (integers, minor units) and `catalog.filters.available`
   (defaults to `true`, i.e. sale-ready items only).
2. **Page through** with `catalog.pagination.cursor` and `catalog.pagination.limit`. The limit defaults
   to `10` and has a minimum of `1`; no maximum is published, so do not assume one.
3. **`get_product`** — pass `catalog.id` for full detail on one product. Use `catalog.selected` to pin
   variant options (name/label pairs) when the shopper has already chosen a size or colour.
4. **`lookup_catalog`** — resolve several products or variants at once by identifier. Use this instead of
   looping `get_product` when you already hold IDs.

## Reading prices correctly

Prices are **integers in ISO 4217 minor units** paired with a currency code.
`{"amount": 2500, "currency": "USD"}` is **$25.00**. Divide by 100 for two-decimal currencies before
quoting anything to a shopper; zero-decimal currencies such as JPY are already whole units. Quoting the
raw integer is the single most likely way to mislead a buyer here.

## Fallback (no MCP)

The merchant's own `agents.md` publishes unauthenticated read-only JSON on the storefront host:

- `GET https://www.ministryofsupply.com/products.json`
- `GET https://www.ministryofsupply.com/collections.json`
- `GET https://www.ministryofsupply.com/products/{handle}.json`
- `GET https://www.ministryofsupply.com/collections/{handle}/products.json`
- `GET https://www.ministryofsupply.com/search?q={query}&type=product`

The merchant's `robots.txt` explicitly **disallows** `/cart.js` and `/recommendations/products` for
agents and directs you to UCP/MCP instead. Honour that.

## Rate limits

The endpoint is rate-limited per IP and no quota is published. Back off on `429`. No `RateLimit-*` or
`Retry-After` headers were observed on a live 200, so treat the status code as your only signal.
