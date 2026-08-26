---
name: rad-power-bikes-catalog-research
description: >-
  Search and read the Rad Power Bikes product catalog through the store's anonymous UCP/MCP
  endpoint, without creating a cart or touching checkout. Read-only and safe to run
  unattended.
api: Rad Power Bikes UCP Commerce MCP
endpoint: https://www.radpowerbikes.com/api/ucp/mcp
generated: '2026-08-26'
method: generated
source: mcp/rad-power-bikes-tools-list.json (live tools/list 2026-08-26)
operations:
  - search_catalog
  - lookup_catalog
  - get_product
---

# Rad Power Bikes — catalog research

Read-only research over the Rad Power Bikes store. No credential is required. Every tool
call needs `meta.ucp-agent.profile` — a URI identifying your agent's UCP profile.

## Steps

1. **Confirm the surface.** `GET https://www.radpowerbikes.com/.well-known/ucp` and check
   `ucp.version`. Two protocol versions were live on 2026-08-26: `2026-04-08` (latest
   stable) and `2026-01-23`.
2. **Search.** Call `search_catalog` with a natural-language `query`, `filters`, or both —
   at least one is required. Pass `context.address_country` and `context.currency` so
   pricing and availability are correct for your buyer.
3. **Page.** Initial results are deliberately limited. Take `pagination.cursor` from the
   response and pass it back on the next `search_catalog` call only when the user asks for
   more.
4. **Resolve identifiers.** Use `lookup_catalog` to fetch several products or variants by
   identifier in one call, or `get_product` for complete detail on one.

## Rules

- **Prices are integers in ISO 4217 minor units** paired with a currency code.
  `{"amount": 2500, "currency": "USD"}` is $25.00. Divide by 100 for two-decimal currencies
  before quoting a price to a person. JPY and other zero-decimal currencies are already
  whole units. Every tool in this API repeats this rule; getting it wrong quotes a buyer a
  price 100x off.
- **Rate limits are per IP** and undocumented in size. On `429`, back off exponentially.
  No `Retry-After` header is returned.
- **Stay read-only.** Nothing in this skill creates a cart or a checkout. If the user wants
  to buy, switch to `rad-power-bikes-guided-purchase` — and read its approval rules first.
- The same product data is also public as plain JSON at `/products/{handle}.json` and
  `/collections/{handle}/products.json`, and as schema.org `Product` JSON-LD on each product
  page, if you only need to read and do not want to speak MCP.
