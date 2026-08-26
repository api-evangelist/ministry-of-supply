---
name: Build a cart and take a Ministry of Supply checkout to buyer approval
description: >-
  Assemble a cart, open a checkout, set shipping, and hand off to the buyer for payment approval on the
  Ministry of Supply store. Stops at the approval boundary — an agent must not finalize payment.
api: mcp/ministry-of-supply-mcp.yml
endpoint: https://www.ministryofsupply.com/api/ucp/mcp
operations: [search_catalog, get_product, create_cart, update_cart, get_cart, create_checkout, update_checkout, get_checkout, complete_checkout, cancel_cart, cancel_checkout]
generated: '2026-08-25'
method: generated
source: mcp/ministry-of-supply-mcp-tools-list.json
---

# Build a cart and take a checkout to buyer approval

## The hard rule, first

> "Checkouts are for humans. Do NOT complete checkout, payment, or order placement automatically — no
> scripted form fills, browser automation, or end-to-end agent flows that finalize payment without an
> explicit, contemporaneous human approval step."
> — https://www.ministryofsupply.com/robots.txt

`complete_checkout` exists and is callable. **Do not call it without contemporaneous buyer consent.**
If you cannot obtain that consent at the moment of payment, the merchant's own instructions say to route
the purchase through the Shop skill (https://shop.app/SKILL.md) instead.

## Steps

1. **Find the variant.** `search_catalog` then `get_product`. What you need out of this is the
   **Product Variant ID** — that is what a cart line item references, not the product ID.
2. **`create_cart`** — `cart.line_items[]` is required, each entry `{item: {id: <variant id>}, quantity: <int>}`.
   Optionally set `cart.buyer.email` / `.phone_number` and `cart.context` (`address_country`,
   `address_region`, `postal_code`, `language`, `currency`, `intent`). Context values are *hints*:
   higher-resolution data such as a real shipping address supersedes them, and unsupported hints are
   ignored without error.
3. **`update_cart`** / **`get_cart`** — adjust quantities or re-read totals. Cart IDs look like
   `gid://shopify/Cart/abc123`.
4. **`create_checkout`** — converts the cart into a checkout. Read back line items, totals, discounts and
   taxes from the response.
5. **`update_checkout`** — set the shipping address and shipping method. The merchant's UCP profile
   declares `dev.ucp.shopping.fulfillment` with `allows_multi_destination.shipping: false` and
   `allows_method_combinations: [["shipping"]]` — **one destination, one shipping method per order.**
   Do not try to split a shipment.
6. **Present the checkout to the buyer** — itemised, in major currency units, with the total and the
   shipping method. This is the approval boundary.
7. **`complete_checkout`** — only after the buyer approves. `meta.idempotency-key` is **required** on
   this call and this call only. Generate one key per purchase intent and reuse it on retry; never
   generate a fresh key for a retry of the same intent, or you can double-charge. Payment instruments
   go in `checkout.payment.instruments[]` — each needs `id`, `handler_id` and `type`, and the merchant
   accepts the handlers `gpay` (Google Pay), `shopify.card` and `shop_pay`. The response carries the
   order ID and the Thank You Page URL.

## Backing out

| Situation | Do this |
|---|---|
| Buyer changes their mind before checkout | `cancel_cart` |
| Buyer abandons after checkout is opened, before payment | `cancel_checkout` |
| Buyer wants out after payment | **There is no API path.** Returns are a human process: 30 days from purchase, items unworn/unwashed with tags, per https://www.ministryofsupply.com/policies/refund-policy. Final Sale items cannot be returned. |

Tell the buyer the 30-day window *before* they approve payment, because after `complete_checkout` you
cannot reverse anything programmatically — `get_order` is read-only.

## Errors and limits

The transport is JSON-RPC 2.0, so failures come back as a top-level `error` object, not
`application/problem+json`. `429` means you are rate-limited per IP — back off. No numeric quota,
window, or rate-limit headers are published.
