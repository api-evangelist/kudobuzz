---
name: kudobuzz-sync-orders-and-customers
description: Sync commerce customers and orders into Kudobuzz APM so post-purchase review requests can be segmented and scheduled.
api: Kudobuzz Developer API
version: v1
base_url: https://api.kudobuzz.com/v1
package: "@kudobuzz/kbclient"
operations:
  - client.apm.customers.createOrUpdate
  - client.apm.orders.createOrUpdate
  - client.apm.orders.fetchOrders
  - client.apm.orders.getOrderById
generated: '2026-07-19'
method: generated
source: https://docs.kudobuzz.com/apm.md
---

# Sync customers and orders into Kudobuzz APM

APM is Kudobuzz's After Purchase Mail product. Pushing customers and orders into it is
what lets Kudobuzz segment an audience and time review requests after a purchase. All
four operations below are documented at https://docs.kudobuzz.com/apm.md — Kudobuzz
publishes no OpenAPI definition and no other APM methods.

## Before you start

- Request Developer API access at https://kudobuzz.com/developer-api (Buffet plan
  merchants, or free for integration developers).
- Credentials live at https://dashboard.kudobuzz.com/settings. This is a server-to-server
  sync, so use `accessToken`, never the browser `clientId`.
- You need the `business_id` for the Kudobuzz merchant account you are writing into. It
  is required on every APM payload.

## Initialize

```javascript
import makeClient from '@kudobuzz/kbclient'
const client = makeClient({
    accessToken: process.env.KUDOBUZZ_ACCESS_TOKEN,
    clientId: process.env.KUDOBUZZ_CLIENT_ID
})
```

## Step 1 — Upsert the customer

`createOrUpdate` creates when `body.id` is absent and updates when you supply it. The
payload is wrapped: an `apmId` alongside a `body`. Required inside `body`:
`business_id` and `last_name`.

```javascript
await client.apm.customers.createOrUpdate({
  apmId: 'your-apm-id',
  body: {
    business_id: 'your-business-id',
    email: 'buyer@example.com',
    first_name: 'Ama',
    last_name: 'Boateng',
    external_customer_id: 'your-platform-customer-id',
    order_id: 'your-platform-order-id',
    orders_count: 3,
    total_spent: '240.00',
    accepts_marketing: true,
    created_at_platform: new Date(),
    tags: ['vip']
  }
})
```

Carry `external_customer_id` and the `*_at_platform` timestamps from your commerce
platform. Kudobuzz is designed to mirror your platform as source of truth, not to own
these identifiers.

**Check `accepts_marketing` honestly.** This write puts PII into a marketing-campaign
audience. Kudobuzz is the GDPR Data Processor; the merchant is the Controller and must
have a lawful basis. Do not sync customers who have not consented.

## Step 2 — Upsert the order

Required: `business_id`, `order_id`, `order_number`, `number`, `name`, `platform`,
`source`, `currency`, `subtotal_price`, `total_line_items_price` and `line_items[]`.
Each line item requires `title`, `name`, `price` and `quantity`.

```javascript
await client.apm.orders.createOrUpdate({
  business_id: 'your-business-id',
  order_id: 'your-platform-order-id',
  order_number: 1042,
  number: 1042,
  name: '#1042',
  platform: /* Platforms */,
  source: 'shopify',
  currency: /* number, per the documented payload */,
  subtotal_price: '180.00',
  total_line_items_price: 180,
  total_price: 195,
  email: 'buyer@example.com',
  financial_status: 'paid',
  fulfillment_status: 'fulfilled',
  processed_at: new Date(),
  created_at_platform: new Date(),
  customer: { first_name: 'Ama', last_name: 'Boateng', orders_count: '3' },
  line_items: [{
    title: 'Kente Tote',
    name: 'Kente Tote',
    price: 90,
    quantity: 2,
    sku: 'KT-01',
    product_id: 'your-product-id',
    image: 'https://…'
  }]
})
```

Keep `line_items[].product_id` aligned with the `external_unique_id` you use when
creating reviews — that is what ties a review back to the product a buyer actually
purchased.

## Step 3 — Reconcile with a paged read

Kudobuzz paginates with a cursor, not offsets. `cursor` means "everything greater than
this id".

```javascript
let cursor
do {
  const page = await client.apm.orders.fetchOrders({
    business_id: 'your-business-id',
    limit: 25,
    cursor,
    sort: '-created_at'
  })
  // page.data is the array; page.metadata.count is the total available
  cursor = /* id of the last item in page.data */
} while (/* page.data.length === 25 */)
```

Fetch a single order when you need to verify one write:

```javascript
await client.apm.orders.getOrderById(orderId)
```

Use `fetchOrders`/`getOrderById` as the reconciliation path after any ambiguous write —
see the retry rule below.

## Retry rules

Kudobuzz's own API standard guarantees **GET, PUT and DELETE are idempotent**, so reads
are always safe to retry. But Kudobuzz publishes **no idempotency key**, so both
`createOrUpdate` calls are non-idempotent POSTs:

- On a clean 4xx, fix and re-send — nothing was written.
- On **429**, back off until `X-Rate-Limit-Reset` and retry; monitor
  `X-Rate-Limit-Remaining` and throttle proactively. Numeric quotas are not published,
  so drive your pacing off the headers rather than an assumed ceiling.
- On a **timeout with no response**, call `getOrderById` or `fetchOrders` to check
  whether the write landed before re-sending. Blind retries duplicate records.

Prefer supplying stable `external_customer_id` / `order_id` values so Kudobuzz's own
upsert semantics absorb a duplicate rather than creating one.

## Error envelope

```json
{ "error": { "message": "…", "code": "444444", "details": [{ "param": "…", "message": "…", "value": "…" }] } }
```

Not RFC 9457. On 422, `error.details[]` names the offending fields. `error.code` is an
internal code, not the HTTP status.

## Related

- `conventions/kudobuzz-conventions.yml`
- `errors/kudobuzz-error-codes.yml`
- `rate-limits/kudobuzz-rate-limits.yml`
- `data-model/kudobuzz-data-model.yml` — full Customer, Order and LineItem field lists
