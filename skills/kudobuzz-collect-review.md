---
name: kudobuzz-collect-review
description: Create a customer review in Kudobuzz, attached to a product and reviewer, using the official Kudobuzz client wrapper.
api: Kudobuzz Developer API
version: v1
base_url: https://api.kudobuzz.com/v1
package: "@kudobuzz/kbclient"
operations:
  - client.core.reviews.createReview
generated: '2026-07-19'
method: generated
source: https://docs.kudobuzz.com/core.md
---

# Collect a review into Kudobuzz

Kudobuzz publishes no OpenAPI definition, so this skill is grounded in the one review
operation documented at https://docs.kudobuzz.com/core.md. Do not invent additional
operations — `createReview` is the only review method Kudobuzz documents.

## Before you start

- You need Developer API access. It is not self-serve: it is granted to merchants on the
  Buffet plan and free to app developers building integrations, by request at
  https://kudobuzz.com/developer-api.
- Get `accessToken` and `clientId` from https://dashboard.kudobuzz.com/settings.
- Use `accessToken` **only server-side**. It is a long-lived static secret with no scopes —
  it grants full account access. Use `clientId` in the browser.

## Steps

1. Install and initialize the client.

   ```bash
   npm i @kudobuzz/kbclient --save-exact
   ```

   ```javascript
   import makeClient from '@kudobuzz/kbclient'
   const client = makeClient({
       accessToken: process.env.KUDOBUZZ_ACCESS_TOKEN,
       clientId: process.env.KUDOBUZZ_CLIENT_ID
   })
   ```

2. Build the review payload. These fields are required: `platform`, `source`,
   `created_at_platform`, `reviewer`, `message`, and `rating` (an integer 1–5).
   `reviewer` itself requires `external_reviewer_id`, `display_name` and `channel`.

   ```javascript
   const reviewPayload = {
     platform: ReviewPlatform.KUDOBUZZ,
     source: /* Sources */,
     created_at_platform: new Date(),
     reviewer: {
       external_reviewer_id: 'your-system-reviewer-id',
       display_name: 'Ama B.',
       channel: /* Channels */,
       email: 'optional@example.com'
     },
     rating: 5,
     title: 'Optional title',
     message: 'Required review body',
     external_unique_id: ['your-product-id'],
     media: []
   }
   ```

3. Check for an existing review **before** you write. Kudobuzz documents no
   idempotency key, so `createReview` is not safe to retry — see step 5.

4. Create the review.

   ```javascript
   await client.core.reviews.createReview(reviewPayload)
   ```

5. Handle failure correctly.

   - **422** — the body is well formed but semantically invalid. Read
     `error.details[]`, which carries `{ param, message, value }` for each offending
     field, and correct the payload. Do not retry unchanged.
   - **400** — malformed request. Deterministic; do not retry.
   - **401 / 403** — missing or unauthorized credentials. Confirm the account actually
     has Developer API access.
   - **429** — rate limited. Back off until the time in `X-Rate-Limit-Reset`, then retry.
     Watch `X-Rate-Limit-Remaining` and throttle before you exhaust it.
   - **Network timeout with no response** — do **not** blindly retry. `createReview` is a
     non-idempotent POST with no idempotency key; a retry can publish a duplicate review.
     Reconcile by reading the review back first.

## Error envelope

Kudobuzz does not use RFC 9457 problem details. Parse this shape instead:

```json
{
  "error": {
    "message": "plain-language description with remediation guidance",
    "code": "444444",
    "details": [{ "param": "rating", "message": "...", "value": "9" }]
  }
}
```

`error.code` is an internal Kudobuzz code and is explicitly **not** the HTTP status code.

## Consequence

Creating a review publishes merchant-visible content and may include reviewer PII
(`email`, `display_name`). Kudobuzz is a GDPR Data Processor; the merchant is the
Controller and must hold a lawful basis for the reviewer data being pushed. Treat this
as a confirm-before-write operation in any autonomous agent.

## Related

- `conventions/kudobuzz-conventions.yml` — pagination, idempotency, rate-limit signalling
- `errors/kudobuzz-error-codes.yml` — full status-code catalog and retry guidance
- `authentication/kudobuzz-authentication.yml` — the token / client-id split
- `data-model/kudobuzz-data-model.yml` — Review and Reviewer field reference
