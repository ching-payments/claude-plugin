---
name: ching-integration
description: Use when integrating CHING into a SaaS product - pulling products and prices, redirecting users to the hosted setup / checkout / portal pages, managing customers and cards, canceling subscriptions, and receiving webhooks.
---

# CHING API Integration

CHING is an ILS-native payments API with hosted pages for card collection, checkout, and the customer portal. This skill covers the six operations a SaaS integration needs.

## Basics

- **API base URL:** `https://api.ching.co.il/ching/v1`
- **Hosted pages** (checkout, setup, portal): `https://secured.ching.co.il`. Always follow the `url` returned by the API - never construct these URLs yourself.
- **Auth header:** `Authorization: Bearer <API_KEY>`. Test keys start with `ck_test_`, live keys with `ck_live_`.
- **Amounts are integer agorot:** `9900` = 99.00 ILS. Never send decimals.
- **Response envelope:** `{ success: true, data: ... }` on success, `{ success: false, error: { code, message } }` on failure.
- **Secret keys never leave the server.** Call the API from your backend; hand the browser only the hosted `url` to redirect to.

---

## Customers

Most endpoints reference a `customer` (`cus_...`). Create one once per end user, store the id on your user row, reuse it forever - do NOT create a fresh CHING customer on each subscription or each login.

### Create

```bash
POST /v1/customers
{
  "name":     "Dana Cohen",
  "email":    "dana@acme.co.il",
  "phone":    "050-123-4567",
  "taxId":    "123456789",
  "locale":   "he",
  "metadata": { "app_user_id": "usr_42" }
}
```

```json
{ "data": { "id": "cus_xxx", "object": "customer", "name": "...", "email": "...", "phone": "...", "taxId": "...", "metadata": {}, "livemode": false, "created": "..." } }
```

Only `name` is required. `phone` accepts Israeli local format (`050-...`) or E.164 (`+972501234567`) and is normalized on save. `taxId` is the Israeli ID / company number, printed on tax documents.

### Retrieve

```bash
GET /v1/customers/{cus_id}
```

Returns the same shape as create.

### List

```bash
GET /v1/customers?limit=25&starting_after=cus_xxx
```

Cursor pagination. Response includes `has_more: boolean`; pass the last item's `id` as the next `starting_after`.

### Update

```bash
POST /v1/customers/{cus_id}
{ "email": "new@acme.co.il" }
```

All fields optional; unsupplied fields stay unchanged.

### List payment methods

```bash
GET /v1/customers/{cus_id}/payment_methods
```

```json
{ "data": [ { "id": "pm_xxx", "type": "card", "card": { "brand": "visa", "last4": "4242", "exp_month": 12, "exp_year": 2030 }, "is_default": true, "status": "active" } ] }
```

Use this on your "Payment methods" UI; combine with §4 for detach / set-default actions.

`GET /v1/customers/{cus_id}/payment_methods/inactive` returns previously-detached cards if you need a history view.

---

## 1. Pull products and prices (for a pricing page)

Build your pricing grid from two list calls.

```bash
GET /v1/products
Authorization: Bearer <API_KEY>
```

```json
{ "data": [ { "id": "prod_xxx", "name": "Pro", "description": "...", "image_url": null, "features": [{"title":"Unlimited","subtitle":"..."}], "active": true, "metadata": {} } ] }
```

Then per product, fetch active prices:

```bash
GET /v1/prices?product=prod_xxx&active=true
```

```json
{ "data": [ { "id": "price_pro_monthly", "product": "prod_xxx", "unit_amount": 9900, "currency": "ils", "type": "recurring", "recurring": { "interval": "month", "interval_count": 1, "trial_period_days": null } } ] }
```

Rendering tips:
- Divide `unit_amount` by 100 for display: `₪99.00`.
- `type: "recurring"` prices have a populated `recurring` object (month/year interval, optional trial). `type: "one_time"` prices have `recurring: null`.
- Use the product's `features` array for the plan's bullet list and `metadata` for any custom flags you attached.
- Filter out products whose `active` is `false` when rendering - inactive products should not be purchasable.

---

## 2. Add a credit card (setup session)

Use this when the user wants to save a card without paying right now (e.g. a "Payment methods" page). ALWAYS use a setup session - never build a card form yourself.

```bash
POST /v1/setup_sessions
{
  "customer": "cus_xxx",
  "success_url": "https://yoursaas.com/settings/cards?ok=1",
  "cancel_url":  "https://yoursaas.com/settings/cards"
}
```

```json
{ "data": { "id": "seti_xxx", "url": "https://secured.ching.co.il/setup/seti_xxx", "expires_at": "..." } }
```

Redirect the browser to `url`. CHING renders the card-entry page, collects the card, runs 3DS when needed, and on success redirects the user back to `success_url`. A new `pm_*` (payment method) is attached to the customer and becomes immediately usable.

After the user returns, the saved card is visible via `GET /v1/customers/{id}/payment_methods`.

---

## 3. Subscribe a user to a plan (checkout session)

Use this for every paid-plan signup. ALWAYS use a checkout session - it handles card collection, 3DS, installments, tax, and receipt in one hosted page.

```bash
POST /v1/checkout_sessions
{
  "customer": "cus_xxx",
  "price":    "price_pro_monthly",
  "success_url": "https://yoursaas.com/billing/success",
  "cancel_url":  "https://yoursaas.com/pricing"
}
```

```json
{ "data": { "id": "co_xxx", "url": "https://secured.ching.co.il/checkout/co_xxx", "expires_at": "..." } }
```

Redirect to `url`. On success, CHING creates the subscription and charges the first period automatically, then sends the user to `success_url`. The subscription is active by the time the redirect lands.

On your success page, confirm state before provisioning:

```bash
GET /v1/subscriptions?customer=cus_xxx
```

Look for `status: "active"` or `"trialing"`.

---

## 4. Remove a card / make a card default

Both are simple merchant-side calls - no redirect.

### Detach (remove)

```bash
POST /v1/payment_methods/{pm_id}/detach
```

```json
{ "data": { "id": "pm_xxx", "status": "inactive" } }
```

The card moves to the inactive list and is no longer chargeable.

### Set as default

```bash
POST /v1/payment_methods/{pm_id}/set_default
```

```json
{ "data": { "id": "pm_xxx", "is_default": true } }
```

Future charges and subscription renewals use the default card unless a specific `payment_method` is passed on the call.

List a customer's active cards via `GET /v1/customers/{id}/payment_methods`; each entry includes `is_default: boolean` and the card's `brand`, `last4`, `exp_month`, `exp_year`.

---

## 5. Cancel a subscription

```bash
POST /v1/subscriptions/{sub_id}/cancel
{
  "cancel_at_period_end": false
}
```

```json
{ "data": { "id": "sub_xxx", "status": "canceled", "cancel_at_period_end": false } }
```

Two modes:
- **`cancel_at_period_end: false`** (or omit) - cancels immediately. `status` becomes `canceled`. No prorated refund.
- **`cancel_at_period_end: true`** - stays `active` until `current_period_end`, then transitions to `canceled`. The customer keeps access for the remainder of the paid period and is not renewed.

If you want to let the customer self-cancel (including the "cancel at period end" choice), point them at the portal instead - see §6.

---

## 6. Open the customer portal

CHING's hosted self-service UI. The customer can view past invoices, change their default card, save a new card, and cancel subscriptions - without you building any of it.

```bash
POST /v1/billing_portal_sessions
{
  "customer":   "cus_xxx",
  "return_url": "https://yoursaas.com/settings/billing"
}
```

```json
{ "data": { "url": "https://secured.ching.co.il/portal/<token>", "expires_at": "..." } }
```

Redirect the user to `url`. When they click "Back to site" (or finish a destructive action), they land on `return_url`.

Use this for any customer-driven billing change; only call the API directly (sections 4-5) when YOU want the merchant dashboard to drive the change.

---

## 7. Webhooks

Your server receives asynchronous events from CHING - subscription state changes, charge outcomes, card attachments. Drive internal state (provisioning, emails, feature flags) off these events, NOT off the `success_url` redirect from the hosted pages. Redirects can be lost or faked; webhooks are authoritative and signed.

### Register an endpoint

```bash
POST /v1/webhooks
{
  "url":    "https://yoursaas.com/webhooks/ching",
  "events": [
    "setup_session.completed",
    "subscription.created",
    "subscription.updated",
    "subscription.canceled",
    "subscription.past_due",
    "subscription.trial_will_end",
    "charge.succeeded",
    "charge.failed",
    "refund.created",
    "payment_method.attached",
    "payment_method.detached"
  ]
}
```

```json
{ "data": { "id": 7, "url": "...", "events": [...], "secret": "whsec_abc..." } }
```

**Store the `secret` securely** - it is shown only on creation and is the HMAC key for signature verification.

Use `["*"]` to subscribe to every event type.

### Event payload

Every webhook POST has this shape:

```json
{
  "id":       "evt_xxx",
  "type":     "charge.succeeded",
  "data":     { /* the resource object that triggered the event */ },
  "livemode": true,
  "created":  "2026-04-24T12:00:00Z"
}
```

`data` mirrors the object returned from the corresponding `GET` endpoint - e.g. `subscription.created` carries a full subscription object (same fields as `GET /v1/subscriptions/{id}`).

### Verify the signature

Header: `Ching-Signature: <hex>`. Verify HMAC-SHA256 of the **raw request body** against your webhook secret. The raw bytes matter - re-serializing the JSON changes the hash.

```js
import crypto from 'crypto'
import express from 'express'

const app = express()

app.post('/webhooks/ching',
  express.raw({ type: 'application/json' }),  // raw body, not JSON-parsed
  (req, res) => {
    const sig = req.header('Ching-Signature')
    const expected = crypto.createHmac('sha256', WEBHOOK_SECRET).update(req.body).digest('hex')
    if (!sig ||
        Buffer.from(expected, 'hex').length !== Buffer.from(sig, 'hex').length ||
        !crypto.timingSafeEqual(Buffer.from(expected, 'hex'), Buffer.from(sig, 'hex'))) {
      return res.status(401).end()
    }
    const event = JSON.parse(req.body.toString('utf8'))
    // ... handle event.type
    res.json({ received: true })
  }
)
```

If you don't verify, anyone who knows your endpoint URL can POST fake events.

### Which events map to which flow

| You did | Listen to | Why |
|---|---|---|
| §1 pricing page (no action) | - | no events |
| §2 setup session completed | `setup_session.completed`, `payment_method.attached` | reflect the new card in your UI |
| §3 checkout session completed | `subscription.created`, `charge.succeeded` | provision the plan / unlock features |
| §4 detach card | `payment_method.detached` | sync your local card cache |
| §5 cancel subscription | `subscription.canceled` (immediate) or `subscription.updated` + later `.canceled` (cancel-at-period-end) | revoke access at the right time |
| §6 portal activity | the same resource events that the customer's action would have triggered via API (`subscription.canceled`, `payment_method.attached`, etc.) | no portal-specific event - react to the underlying change |
| automatic renewal | `charge.succeeded` + `subscription.updated` (success) or `charge.failed` + `subscription.past_due` (failure) | keep renewal state fresh |

### Idempotency

CHING retries failed deliveries, so your handler can receive the same `evt_*` more than once. Before acting:

```
if event_id exists in your webhook_events table → respond 200 and exit
else insert event_id, then process the event
```

A simple unique index on `ching_event_id` is enough. Always respond with a 2xx quickly; long-running work should be queued.

### Available event types

- `setup_session.completed` / `.failed` / `.expired`
- `charge.succeeded` / `.failed`
- `refund.created` (failed refunds return a synchronous 400; no webhook)
- `subscription.created` / `.updated` / `.canceled` / `.past_due` / `.trial_will_end`
- `payment_method.attached` / `.detached`

CHING does NOT emit `checkout_session.*` events - fulfil from the underlying `charge.succeeded` / `subscription.created` instead.

---

## Full endpoint catalog

See `api-reference.md` in this skill directory for complete request/response fields and every endpoint not covered here (customers, charges, refunds, webhooks, products/prices CRUD, etc.).
