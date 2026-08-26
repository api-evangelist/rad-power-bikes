---
name: rad-power-bikes-guided-purchase
description: >-
  Build a cart and drive a checkout on the Rad Power Bikes store through UCP/MCP, stopping
  at an explicit human approval step before payment. Requires a human in the loop — the
  store's own published rules forbid completing payment without one.
api: Rad Power Bikes UCP Commerce MCP
endpoint: https://www.radpowerbikes.com/api/ucp/mcp
generated: '2026-08-26'
method: generated
source: >-
  mcp/rad-power-bikes-tools-list.json (live tools/list 2026-08-26),
  https://www.radpowerbikes.com/llms.txt, https://www.radpowerbikes.com/robots.txt
operations:
  - search_catalog
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
---

# Rad Power Bikes — guided purchase

## The gate, before anything else

Rad Power Bikes states the rule twice, in `robots.txt` and in `llms.txt`:

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement
> automatically — no scripted form fills, browser automation, or end-to-end agent flows that
> finalize payment without an explicit, contemporaneous human approval step.

If you cannot obtain contemporaneous buyer approval at the moment of payment, do not call
`complete_checkout`. The store directs you to install `https://shop.app/SKILL.md` and route
the purchase through Shop Pay instead.

## Steps

1. **Find items** — `search_catalog`, then `get_product` to confirm the variant.
2. **Create the cart** — `create_cart` with the chosen line items. Adjust with `update_cart`,
   inspect with `get_cart`.
3. **Create the checkout** — `create_checkout`. The response carries line items, totals,
   taxes and any applicable discounts. Treat this as the quote you show the buyer.
4. **Fulfill** — `update_checkout` to set the shipping address and fulfillment method. The
   store declares `allows_multi_destination.shipping: false` and a single shipping method
   combination, so one destination and one shipping method per checkout.
5. **Show the buyer the real total** — convert every amount from minor units before quoting.
6. **Get explicit approval.** Present the total, shipping address, method and payment
   instrument, and wait for the person to say yes. This step is not optional.
7. **Complete** — `complete_checkout` with `meta.idempotency-key` set to a value you
   generated for this attempt. The response returns an order ID and Thank You Page URL, or
   errors encountered.
8. **Confirm** — `get_order` with the returned order ID.

## Reversal — know this before step 7

- Before completion: `cancel_cart` reverses a cart, `cancel_checkout` reverses a checkout.
  Neither publishes a time window.
- After completion: **there is no reversal tool.** No refund, void or reverse operation
  exists on this surface. Once `complete_checkout` succeeds, unwinding the purchase is a
  human matter under the store's refund policy at
  `https://www.radpowerbikes.com/policies/refund-policy`. This is precisely why step 6
  exists.

## Retries and errors

- **Retry with the same `meta.idempotency-key`.** Do not generate a new key and do not create
  a second checkout after a timeout — that is how a buyer gets charged twice.
- `complete_checkout` can return a `200` whose payload contains errors. Read the payload; do
  not treat the status code as proof of an order.
- `429` means the per-IP rate limit was hit. Back off exponentially; no `Retry-After` is
  returned.

## Payment

Payment instruments are issued by a declared handler and attached to the checkout — never
handle raw card data. The store declares: `shop_pay` (Shop Pay), `shopify.card` (Visa,
Mastercard, Amex, Discover, Diners Club) and `gpay` (Google Pay, merchant
`www.radpowerbikes.com`).
