# CHING API Reference

Complete endpoint catalog. Base URL: `https://api.ching.co.il/v1`. All endpoints require `Authorization: Bearer <API_KEY>` unless noted.

**Reminders:**
- Amounts are integer **agorot** (100 = 1.00 ILS).
- Currency is `ils` (only option).
- Test keys = `ck_test_*`, live keys = `ck_live_*`.
- For plan signup, always use `/checkout_sessions`; for saving a card, always use `/setup_sessions` (see SKILL.md Two Golden Rules).

## Response Envelope

All successful responses:
```json
{ "success": true, "data": { ... } }
```

Errors:
```json
{
  "success": false,
  "error": {
    "status": 400,
    "code": "price_archived",
    "message": "This price is archived and cannot be used..."
  }
}
```

Common error codes: `NO_ACCESS_ERROR` (401), `NOT_FOUND_ERROR` (404), `LIVE_KEY_INACTIVE_ERROR` (401), `price_archived` (400), validation codes per resource.

## Checkout Sessions

Hosted checkout - redirect the customer here to subscribe or pay.

### `POST /checkout_sessions`

Request:
| Field | Type | Required | Notes |
|---|---|---|---|
| `customer` | string | yes | `cus_...` |
| `price` | string | yes | `price_...` (determines amount, currency, interval) |
| `success_url` | string | yes | Where to redirect after success |
| `cancel_url` | string | yes | Where to redirect on cancel |

Response:
```json
{
  "id": "co_abc",
  "url": "https://secured.ching.co.il/checkout/co_abc",
  "expires_at": "2026-04-23T10:30:00Z"
}
```

Session TTL: 30 minutes. Listen to `checkout_session.completed` (or `.failed`/`.expired`) webhooks.

### `GET /checkout_sessions/{id}`

Returns the session with its current `status`: `pending` | `succeeded` | `failed` | `expired` | `canceled`.

## Setup Sessions

Hosted card collection - redirect the customer here to save a card without charging.

### `POST /setup_sessions`

Request:
| Field | Type | Required | Notes |
|---|---|---|---|
| `customer` | string | yes | `cus_...` |
| `success_url` | string | yes | |
| `cancel_url` | string | yes | |
| `metadata` | object | no | Arbitrary key/value |

Response:
```json
{
  "id": "seti_abc",
  "object": "setup_session",
  "customer": "cus_xxx",
  "status": "pending",
  "url": "https://secured.ching.co.il/setup/seti_abc",
  "success_url": "...",
  "cancel_url": "...",
  "metadata": null,
  "livemode": false,
  "expires_at": "2026-04-24T09:00:00Z",
  "created": "2026-04-23T09:00:00Z"
}
```

Session TTL: 24 hours. On success, a `payment_method` (`pm_*`) is attached to the customer. Listen for `setup_session.completed`.

### `GET /setup_sessions/{id}`

Same shape; current `status` of `pending` | `requires_action` | `succeeded` | `failed` | `canceled` | `expired`.

## Customers

### `POST /customers`

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | |
| `email` | string | no | |
| `phone` | string | no | E.164 or Israeli format; auto-normalized |
| `locale` | string | no | Defaults to `"he"` |
| `taxId` | string | no | Israeli ID / company number |
| `metadata` | object | no | |

Response: `{ id: "cus_...", object: "customer", ..., livemode, created }`

### `GET /customers/{id}`

Retrieve a customer.

### `GET /customers`

List. Supports pagination via `?limit=` and `?starting_after=`.

### `POST /customers/{id}` (update)

Same fields as create; all optional; unsupplied fields are unchanged.

### `GET /customers/{id}/payment_methods`

Returns `{ data: [ { id: "pm_...", type: "card", card: { brand, last4, exp_month, exp_year, ... }, is_default, status: "active" } ] }`.

### `GET /customers/{id}/payment_methods/inactive`

Same shape for detached cards.

## Products

### `POST /products`

| Field | Type | Required |
|---|---|---|
| `name` | string | yes |
| `description` | string | no |
| `image_url` | string | no |
| `metadata` | object | no |

### `GET /products/{id}`, `GET /products`, `POST /products/{id}` (update)

Update accepts `name`, `description`, `image_url`, `active`, `metadata`.

## Prices

### `POST /prices`

| Field | Type | Required | Notes |
|---|---|---|---|
| `product` | string | yes | `prod_...` |
| `unit_amount` | integer | yes | Agorot |
| `currency` | string | no | Defaults to `"ils"` |
| `type` | string | yes | `"recurring"` or `"one_time"` |
| `tax_mode` | string | no | `"inclusive"` (default) or `"exclusive"` |
| `recurring` | object | if `type=recurring` | `{ interval: "month"|"year", interval_count?: number, trial_period_days?: number }` |
| `metadata` | object | no | |

Once created, price `unit_amount` is immutable for billing - create a new price to change it (use `apply_mode` below for migration).

### `GET /prices/{id}`, `GET /prices?product=prod_...`

### `POST /prices/{id}` (update)

| Field | Type | Notes |
|---|---|---|
| `unit_amount` | integer | New amount |
| `apply_mode` | string | **Required:** `"now"` \| `"new_subscribers_only"` \| `"scheduled"` |
| `effective_date` | string | Required when `apply_mode=scheduled` (ISO timestamp) |
| `trial_period_days` | integer | |
| `tax_mode` | string | |
| `metadata` | object | |

### `DELETE /prices/{id}/pending_migration`

Cancel a scheduled price migration.

## Subscriptions

### `POST /subscriptions`

| Field | Type | Required | Notes |
|---|---|---|---|
| `customer` | string | yes | |
| `payment_method` | string | yes | Must be active `pm_*` on this customer |
| `price` | string | yes | |
| `metadata` | object | no | |

Response:
```json
{
  "id": "sub_xxx",
  "object": "subscription",
  "customer": "cus_xxx",
  "default_payment_method": "pm_xxx",
  "status": "active",
  "currency": "ils",
  "items": [{ "id": "si_xxx", "price": "price_xxx", "quantity": 1 }],
  "billing_cycle_anchor": "2026-04-23T09:00:00Z",
  "current_period_start": "2026-04-23T09:00:00Z",
  "current_period_end": "2026-05-23T09:00:00Z",
  "cancel_at_period_end": false,
  "latest_charge": "ch_xxx",
  "metadata": null,
  "livemode": false,
  "created": "2026-04-23T09:00:00Z"
}
```

Statuses: `active` | `trialing` | `past_due` | `canceled` | `incomplete` | `incomplete_expired`.

### `POST /subscriptions/{id}/cancel`

```json
{ "cancel_at_period_end": false }
```

`true` = stays `active` until `current_period_end` then transitions to `canceled`.

### `GET /subscriptions/{id}`, `GET /subscriptions`

## Charges

One-time charge on a saved payment method.

### `POST /charges`

| Field | Type | Required | Notes |
|---|---|---|---|
| `customer` | string | yes | |
| `payment_method` | string | yes | Must be active card |
| `amount` | integer | yes | Agorot |
| `currency` | string | no | Defaults `"ils"` |
| `description` | string | no | |
| `installments` | object | no | `{ count: 1-12 }` |
| `metadata` | object | no | |

Response:
```json
{
  "id": "ch_xxx",
  "object": "charge",
  "status": "succeeded",
  "amount": 5000,
  "currency": "ils",
  "installments": { "count": 3 },
  "captured": true,
  "refunded_amount": 0,
  "failure_code": null,
  "failure_message": null,
  "document": null,
  "livemode": false,
  "created": "2026-04-23T09:00:00Z"
}
```

Statuses: `succeeded` | `failed` | `pending` | `processing` | `canceled`.

### `GET /charges/{id}`, `GET /charges`

## Refunds

### `POST /refunds`

| Field | Type | Required | Notes |
|---|---|---|---|
| `charge` | string | yes | |
| `amount` | integer | yes | Agorot, <= remaining refundable |
| `reason` | string | no | `"requested_by_customer"` \| `"duplicate"` \| `"fraudulent"` |
| `metadata` | object | no | |

Response statuses: `succeeded` | `failed` | `pending`.

### `GET /refunds/{id}`, `GET /refunds`

## Payment Methods

Payment methods are created indirectly via setup sessions or checkout sessions. There is no direct `POST /payment_methods` on the public API.

### `GET /customers/{id}/payment_methods`, `GET /customers/{id}/payment_methods/inactive`

### `POST /payment_methods/{id}/detach`

Removes a card from a customer; it moves to the inactive list.

### `POST /payment_methods/{id}/set_default`

Marks this card as the default for future charges/subscriptions.

## Documents (Israeli Tax Invoices)

Auto-generated when a charge succeeds and invoicing is enabled in the project.

### `GET /documents`, `GET /documents/{id}`

Returns invoice/receipt metadata (type: `invoice_receipt` | `receipt` | `credit_note`), amount, status, and `pdf_url` when available.

## Webhooks

### `POST /webhooks`

| Field | Type | Required | Notes |
|---|---|---|---|
| `url` | string | yes | HTTPS URL |
| `events` | string[] | yes | Specific events or `["*"]` |

Response includes `secret` (`whsec_*`) - store it, it's shown only on creation.

### `GET /webhooks`, `DELETE /webhooks/{id}`

### Event payload

```json
{
  "id": "evt_xxx",
  "type": "charge.succeeded",
  "data": { "...object matching the event's resource..." },
  "livemode": false,
  "created": "2026-04-23T09:00:00Z"
}
```

### Signature verification

Header: `Ching-Signature: <hex>`. Verify HMAC-SHA256 of the **raw request body** against the webhook secret:

```js
const expected = crypto.createHmac('sha256', webhook_secret).update(rawBody).digest('hex')
crypto.timingSafeEqual(Buffer.from(expected, 'hex'), Buffer.from(headerValue, 'hex'))
```

Never compute over the re-serialized JSON - use the exact bytes you received.

### Available event types

- `checkout_session.completed` / `.failed` / `.expired`
- `setup_session.completed` / `.failed` / `.expired`
- `customer.created` / `.updated` / `.deleted`
- `payment_method.attached` / `.detached` / `.updated`
- `charge.succeeded` / `.failed`
- `refund.created` / `.failed`
- `subscription.created` / `.updated` / `.canceled` / `.past_due` / `.trial_will_end`
- `document.issued`

## Billing Portal

Customer-facing self-service. You create a session, the customer gets a URL.

### `POST /billing_portal_sessions`

```json
{ "customer": "cus_xxx" }
```

Returns `{ url, token, expires_at }`. Redirect the customer to `url`.

### Customer-scoped portal endpoints (use returned `token`)

These endpoints are used inside the hosted portal, but are documented for completeness:

- `GET /billing_portal/me`
- `GET /billing_portal/me/subscriptions`
- `POST /billing_portal/me/subscriptions/{id}/cancel`
- `POST /billing_portal/me/subscriptions/{id}/resume`
- `GET /billing_portal/me/payment_methods`
- `POST /billing_portal/me/payment_methods/setup` - initiates a setup session
- `GET /billing_portal/me/documents`

## Pagination

List endpoints use cursor pagination:

```
GET /customers?limit=25&starting_after=cus_xyz
```

Response includes `has_more: boolean` and the last item's `id` is used as the next `starting_after`.

## Idempotency

Include an `Idempotency-Key: <any-unique-string>` header on mutating requests to safely retry. CHING replays the original response for up to 24 hours.

## Rate Limits

Documented per key in the dashboard. 429 responses include `Retry-After` seconds - back off and retry.

## Quick Endpoint Map

```
Auth
  (header)   Authorization: Bearer ck_(test|live)_...

Checkout / Setup (hosted, redirect-based)
  POST   /checkout_sessions
  GET    /checkout_sessions/{id}
  POST   /setup_sessions
  GET    /setup_sessions/{id}

Customers
  POST   /customers
  GET    /customers
  GET    /customers/{id}
  POST   /customers/{id}
  GET    /customers/{id}/payment_methods
  GET    /customers/{id}/payment_methods/inactive

Catalog
  POST   /products
  GET    /products[/{id}]
  POST   /products/{id}
  POST   /prices
  GET    /prices[/{id}]
  POST   /prices/{id}
  DELETE /prices/{id}/pending_migration

Billing
  POST   /subscriptions
  GET    /subscriptions[/{id}]
  POST   /subscriptions/{id}/cancel
  POST   /charges
  GET    /charges[/{id}]
  POST   /refunds
  GET    /refunds[/{id}]

Payment methods
  POST   /payment_methods/{id}/detach
  POST   /payment_methods/{id}/set_default

Webhooks
  POST   /webhooks
  GET    /webhooks
  DELETE /webhooks/{id}

Portal
  POST   /billing_portal_sessions
```
