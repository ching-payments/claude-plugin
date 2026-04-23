---
name: ching-integration
description: Use when integrating the CHING payments API into a SaaS product, including subscriptions, checkout sessions, saved payment methods, webhooks, refunds, or customer billing portal. Covers ILS billing, agorot amounts, and the two golden rules around checkout/setup sessions.
---

# CHING API Integration

CHING is a payments API for the Israeli market, built ILS-native with support for subscriptions, one-time charges, Israeli tax documents, and installments. This skill teaches how to integrate it into a SaaS product so customers can pay, subscribe, manage cards, and self-serve billing.

## Two Golden Rules (NON-NEGOTIABLE)

These two rules override any creative implementation. Always follow them:

1. **When a customer wants to join a plan (subscribe / purchase) -> ALWAYS open a CHING checkout page.**
   - Do NOT charge the card directly, do NOT collect card data yourself, do NOT build a custom payment form.
   - The only correct flow: `POST /v1/checkout_sessions` -> redirect the user to the returned `url`.
   - CHING's hosted checkout handles PCI, 3DS, installments, tax, and receipt.

2. **When a customer wants to add or save a credit card (no charge right now) -> ALWAYS open a CHING setup session.**
   - Do NOT tokenize the card yourself. Do NOT create an empty charge.
   - The only correct flow: `POST /v1/setup_sessions` -> redirect to the returned `url`.
   - After success, the card is attached as a `payment_method` on the customer and can be used for future charges/subscriptions.

Violations of either rule = broken integration. If you find yourself about to build a card form, STOP and use the hosted session instead.

## When to Use Which Endpoint

| Business goal | Correct action |
|---|---|
| Customer signs up for a paid plan | Create customer -> create checkout session -> redirect |
| Customer adds a credit card without charging | Create setup session -> redirect |
| Charge an already-saved card one time | `POST /v1/charges` with `customer` + `payment_method` |
| Recurring billing from a saved card | `POST /v1/subscriptions` with `customer` + `payment_method` + `price` |
| Let the customer manage their subscriptions / cards themselves | Create billing portal session -> redirect |
| Issue a refund | `POST /v1/refunds` with `charge` + `amount` |
| React to payment events | Register a webhook endpoint, verify signatures |

## Base URL & Authentication

- **Base URL:** `https://api.ching.co.il/ching/v1`
- **Auth header:** `Authorization: Bearer <API_KEY>`
- **Test keys:** `ck_test_*` (never charge real money)
- **Live keys:** `ck_live_*` (real charges; must be activated in dashboard)
- **Livemode header (optional):** `X-Livemode: false` to explicitly run in test mode

Test and live data are strictly isolated. A test-mode customer cannot be used with a live-mode charge.

## Amounts, Currency, IDs

- **Currency:** `ils` (ILS is the only supported currency today).
- **Amounts are in agorot (integer, smallest unit):** `9900` = 99.00 ILS. NEVER send `99.00` or `"99"`.
- **ID prefixes (human-debuggable):**

| Resource | Prefix | Example |
|---|---|---|
| Customer | `cus` | `cus_abc123` |
| Product | `prod` | `prod_xyz456` |
| Price | `price` | `price_def789` |
| Payment Method | `pm` | `pm_ghi012` |
| Checkout Session | `co` | `co_jkl345` |
| Setup Session | `seti` | `seti_mno678` |
| Subscription | `sub` | `sub_pqr901` |
| Charge | `ch` | `ch_stu234` |
| Refund | `re` | `re_vwx567` |
| Event | `evt` | `evt_yza890` |

## The Canonical Integration Flow

### 1. Sign the customer up on your SaaS and mirror them in CHING

```bash
POST https://api.ching.co.il/ching/v1/customers
Authorization: Bearer ck_test_...
Content-Type: application/json

{
  "name": "Dana Cohen",
  "email": "dana@acme.co.il",
  "phone": "050-123-4567",
  "metadata": { "your_user_id": "usr_42" }
}
```

Store the returned `id` (`cus_...`) in your database alongside your user record. This is the link between your SaaS and CHING.

### 2a. User clicks "Upgrade to Pro" -> checkout session (GOLDEN RULE #1)

```bash
POST /v1/checkout_sessions

{
  "customer": "cus_abc123",
  "price": "price_pro_monthly",
  "success_url": "https://yoursaas.com/billing/success?sid={CHECKOUT_SESSION_ID}",
  "cancel_url": "https://yoursaas.com/billing"
}
```

Response contains `url`. Redirect the browser to it. CHING hosts checkout, collects the card, and on success redirects to your `success_url`. You do NOT need to fetch the session to confirm - listen to the `checkout_session.completed` webhook instead (or tolerate a round-trip to `GET /v1/checkout_sessions/{id}`).

### 2b. User clicks "Add a new card" -> setup session (GOLDEN RULE #2)

```bash
POST /v1/setup_sessions

{
  "customer": "cus_abc123",
  "success_url": "https://yoursaas.com/settings/cards?sid={SETUP_SESSION_ID}",
  "cancel_url": "https://yoursaas.com/settings/cards"
}
```

Redirect to `url`. After the customer saves the card, a `payment_method` is attached to the customer. Listen to `setup_session.completed` or fetch payment methods when the user returns.

### 3. Charge a saved card (one-time) or create a subscription

One-off charge (e.g. top-up, overage):

```bash
POST /v1/charges

{
  "customer": "cus_abc123",
  "payment_method": "pm_ghi012",
  "amount": 5000,
  "description": "Credit pack - 100 credits"
}
```

Recurring subscription:

```bash
POST /v1/subscriptions

{
  "customer": "cus_abc123",
  "payment_method": "pm_ghi012",
  "price": "price_pro_monthly"
}
```

Note: you only get to call these directly if the card was saved via a setup session. The normal "subscribe" flow (§ 2a) goes through a checkout session, which creates the subscription for you.

### 4. Let customers self-manage (billing portal)

```bash
POST /v1/billing_portal_sessions

{ "customer": "cus_abc123" }
```

Redirect to the returned `url`. The customer can view invoices, swap default card, cancel/resume subscriptions.

### 5. Listen to webhooks

Always drive business logic (provisioning, emails, feature unlocks) from webhooks, not from the redirect to `success_url`. Redirects can be skipped, lost, or faked; webhooks are authoritative.

```bash
POST /v1/webhooks

{
  "url": "https://yoursaas.com/webhooks/ching",
  "events": [
    "checkout_session.completed",
    "setup_session.completed",
    "subscription.created",
    "subscription.updated",
    "subscription.canceled",
    "subscription.past_due",
    "charge.succeeded",
    "charge.failed",
    "refund.created"
  ]
}
```

Store the returned `secret` (starts with `whsec_`). Verify every incoming webhook:

```js
const crypto = require('crypto')

function verifyChingWebhook(rawBody, signatureHeader, secret) {
  const expected = crypto.createHmac('sha256', secret).update(rawBody).digest('hex')
  const a = Buffer.from(expected, 'hex')
  const b = Buffer.from(signatureHeader, 'hex')
  if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) {
    throw new Error('Invalid CHING signature')
  }
}
```

The signature arrives as `Ching-Signature: <hex>`. Read the raw body (NOT the parsed JSON) before signature verification.

## Full Endpoint Reference

See `api-reference.md` in this skill directory for the complete catalog (request/response fields for every endpoint).

## Webhook Events

| Event | When it fires | Good for |
|---|---|---|
| `checkout_session.completed` | Hosted checkout succeeded | Provision access, unlock features |
| `checkout_session.failed` | Checkout failed | Show retry UI |
| `setup_session.completed` | Card saved successfully | Mark card setup done in your UI |
| `setup_session.failed` / `setup_session.expired` | Card not saved | Prompt user to retry |
| `subscription.created` | New subscription (fires before `charge.succeeded`) | Start provisioning |
| `subscription.updated` | Plan change, cycle rolled | Sync plan state |
| `subscription.canceled` | Terminated | Revoke access (respect `cancel_at_period_end`) |
| `subscription.past_due` | Payment failed, dunning started | Notify user, show banner |
| `subscription.trial_will_end` | 3 days before trial ends | Remind user to add card |
| `charge.succeeded` | Payment captured | Fulfill, send receipt |
| `charge.failed` | Payment declined | Surface failure reason |
| `refund.created` / `refund.failed` | Refund processed | Update ledger |
| `payment_method.attached` / `payment_method.detached` | Card added/removed | Sync card list |

## Subscription Lifecycle

Statuses: `active` | `trialing` | `past_due` | `canceled` | `incomplete` | `incomplete_expired`

- `incomplete` -> first charge failed; expires after ~23h if not resolved -> `incomplete_expired`.
- `past_due` -> CHING retries on days 3, 5, 7; cancels after all retries fail.
- `trialing` -> no charge until trial ends; listen to `subscription.trial_will_end` to nudge the user.
- `cancel_at_period_end: true` -> subscription stays `active` until `current_period_end`, then transitions to `canceled`.

Provision access when status is `active` or `trialing`. Revoke when `canceled` (respecting period end) or `incomplete_expired`.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Building a card form on your own site | Use checkout session (golden rule 1) or setup session (golden rule 2) |
| Sending amount as `99` expecting 99 ILS | Amounts are agorot - `99` = 0.99 ILS. Send `9900` for 99 ILS. |
| Confirming a plan purchase from the `success_url` redirect | Use the `checkout_session.completed` webhook; redirects can be bypassed. |
| Mixing test and live IDs (e.g. test customer + live price) | Test/live are strictly isolated. Use the matching API key. |
| Skipping webhook signature verification | Anyone can POST to your endpoint. Always verify `Ching-Signature`. |
| Parsing JSON before signature verification | Verify against the **raw request body** - re-serialized JSON produces a different signature. |
| Calling `POST /v1/charges` without a saved `payment_method` | Charges and subscriptions require a pre-attached `pm_*`. Run a setup session first, OR use a checkout session which handles card collection for you. |
| Trusting an unauthenticated `customer` id from the browser | Always look up the customer from your authenticated session, not from a query param. |

## Red Flags - STOP

If you catch yourself doing any of these, stop and reconsider:

- "I'll just collect the card number with a small form..." -> **NO.** Setup session or checkout session.
- "I'll hit `/v1/charges` directly for the first subscription payment..." -> **NO.** Checkout session.
- "The redirect hit our `/success` page, so we're done..." -> **NO.** Wait for the webhook.
- "I'll compute signature by JSON.stringify'ing the parsed body..." -> **NO.** Verify the raw bytes.
- "Let me hardcode the amount in dollars..." -> **NO.** Integer agorot, ILS only.

## Testing Locally

- Use the test key (`ck_test_...`).
- Test mode mocks card processing - any valid-shape card number works (e.g., `4242 4242 4242 4242`).
- Use a tunneling tool (ngrok, cloudflared) to receive webhooks on `localhost`.
- Set `X-Livemode: false` on requests to be explicit.

## One-Page Code Example (Node + Express)

```js
import express from 'express'
import crypto from 'crypto'
import fetch from 'node-fetch'

const CHING = 'https://api.ching.co.il/ching/v1'
const KEY = process.env.CHING_API_KEY           // ck_test_... or ck_live_...
const WH_SECRET = process.env.CHING_WEBHOOK_SECRET

const ching = (path, body) => fetch(`${CHING}${path}`, {
  method: 'POST',
  headers: { Authorization: `Bearer ${KEY}`, 'Content-Type': 'application/json' },
  body: JSON.stringify(body),
}).then(r => r.json())

const app = express()

// On your signup handler, mirror the user into CHING.
async function createChingCustomer(user) {
  const r = await ching('/customers', {
    name: user.fullName,
    email: user.email,
    metadata: { app_user_id: user.id },
  })
  return r.data.id
}

// Subscribe flow - GOLDEN RULE #1: always use checkout session.
app.post('/billing/subscribe', async (req, res) => {
  const { customerId, priceId } = req.body          // from your authenticated session
  const r = await ching('/checkout_sessions', {
    customer: customerId,
    price: priceId,
    success_url: `${process.env.PUBLIC_URL}/billing/success`,
    cancel_url: `${process.env.PUBLIC_URL}/billing`,
  })
  res.json({ url: r.data.url })                     // client redirects here
})

// Add card flow - GOLDEN RULE #2: always use setup session.
app.post('/billing/add-card', async (req, res) => {
  const { customerId } = req.body
  const r = await ching('/setup_sessions', {
    customer: customerId,
    success_url: `${process.env.PUBLIC_URL}/settings/cards`,
    cancel_url: `${process.env.PUBLIC_URL}/settings/cards`,
  })
  res.json({ url: r.data.url })
})

// Self-service portal.
app.post('/billing/portal', async (req, res) => {
  const { customerId } = req.body
  const r = await ching('/billing_portal_sessions', { customer: customerId })
  res.json({ url: r.data.url })
})

// Webhook handler - use raw body parser for signature verification.
app.post('/webhooks/ching', express.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.header('Ching-Signature')
  const expected = crypto.createHmac('sha256', WH_SECRET).update(req.body).digest('hex')
  if (!sig || !crypto.timingSafeEqual(Buffer.from(expected, 'hex'), Buffer.from(sig, 'hex'))) {
    return res.status(401).end()
  }
  const event = JSON.parse(req.body.toString('utf8'))
  switch (event.type) {
    case 'checkout_session.completed':
      // event.data is the checkout session; provision the user's plan now
      break
    case 'subscription.canceled':
      // revoke features after current_period_end if cancel_at_period_end
      break
    case 'charge.succeeded':
      // store receipt, unlock
      break
    // ...handle more events
  }
  res.json({ received: true })
})
```

## Summary

- Two rules above everything: **checkout session for joining plans, setup session for saving cards.**
- Amounts in agorot, currency `ils`, IDs prefixed by resource.
- Webhooks (not redirects) are the source of truth.
- Always verify `Ching-Signature` against the raw body.
- Full endpoint catalog in `api-reference.md`.
