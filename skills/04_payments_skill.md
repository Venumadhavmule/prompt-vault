# Payments Skill -- Instruction Set

> Load this skill at the START of any task involving Stripe, payment intents,
> subscriptions, webhooks, billing portals, invoices, refunds, or trial periods.
> Money movement requires zero tolerance for logic errors.

---

## Prime Directive

Payments involve real money, legal compliance, and trust.
A bug here results in customers being double-charged, undercharged, or locked out.
Every payment flow must be idempotent. Every webhook must be idempotent.
Test every path. Never hardcode prices. Never trust client-provided amounts.

---

## 1. Pre-Work: Map the Entire Payment Flow

Before writing any code, map the complete flow on paper:

1. Where does the user enter payment information?
2. What product/price are they purchasing? (one-time, subscription, metered)
3. Does the product need a trial period?
4. What happens after payment succeeds? (access granted, email sent, webhook handled)
5. What happens after payment fails? (retry logic, email, access revoked)
6. What happens when a subscription renews, upgrades, or cancels?
7. What data needs to be stored in your DB vs delegated to Stripe?

Answer all 7 before starting.

---

## 2. Core Architecture Decisions

### Store Prices in Stripe, Not in Your DB

NEVER hardcode price amounts in your application code or database.
Create products and prices in the Stripe dashboard (or via API).
Store only the Stripe Price ID in your `.env` or DB config table.

```
CORRECT: priceId = process.env.STRIPE_PRICE_PRO_MONTHLY
WRONG:   amount = 2900 (hardcoded cents)
```

This gives you the ability to change pricing without code deployments.

### Store the Minimum in Your DB

What to store in YOUR database:
- `stripe_customer_id` on the user record
- `stripe_subscription_id` on the subscription record
- `subscription_status` (enum: trialing, active, past_due, canceled, unpaid)
- `current_period_start`, `current_period_end`
- `cancel_at_period_end` (boolean)
- `stripe_price_id` (which plan)

What NOT to store in your DB:
- Card numbers, CVV, expiry (Stripe handles this -- you are NOT PCI compliant for raw card data)
- Invoice amounts, line items (fetch from Stripe API when needed)
- Full customer billing address (unless needed for tax compliance)

---

## 3. Customer Lifecycle Management

### Rule: One Stripe Customer Per User

When a user first enters a payment context (checkout, adding a card, etc.):
1. Check if `stripe_customer_id` already exists on the user
2. If NOT: create a Stripe customer and save the ID to DB immediately
3. ALL subsequent Stripe operations use the stored customer ID

NEVER create duplicate Stripe customers for the same user.
If you detect duplicates: merge via Stripe customer merge API and update your DB.

### Creating a Stripe Customer

Always pass:
- `email`: the user's email
- `name`: user's name (helps in Stripe dashboard)
- `metadata.userId`: your internal user ID (critical for debugging and webhook correlation)

```
metadata: {
  userId: "internal_user_uuid",
  environment: "production" | "staging"
}
```

---

## 4. Checkout Session vs Payment Intent

| Use Case | Use |
|---|---|
| Subscription signup, one-time purchase | Checkout Session (Stripe-hosted page) |
| Custom checkout UI in your app | Payment Intent + Stripe Elements |
| In-app upgrade with card already saved | Payment Intent with saved payment method |
| B2B with invoicing | Stripe Invoices |

Default to Checkout Session (hosted) for new integrations. It handles:
- SCA/3DS authentication automatically
- Coupon/promo code entry
- Tax collection
- Subscription creation
- Failed payment retry

Only build custom checkout (Payment Intent + Elements) when UX requirements demand it.

---

## 5. Webhook Architecture -- This Is the Heart of Stripe Integration

The webhook is the authoritative source of truth for payment state.
NEVER rely on redirect URLs (success_url) as confirmation of payment.
A user can close the browser tab after payment -- the redirect never fires.
The webhook always fires.

### Webhook Security

ALWAYS verify the Stripe webhook signature before processing any event.
Use `stripe.webhooks.constructEvent(rawBody, signature, webhookSecret)`.
This requires the RAW request body (before JSON parsing).
If your framework parses JSON automatically, configure raw body access for the webhook route only.

On signature verification failure: return 400 immediately. Log the attempt.

### Webhook Idempotency

Webhooks can be delivered more than once.
EVERY webhook handler MUST be idempotent (safe to process multiple times).

Implementation: Store processed event IDs in DB or Redis.
Before processing: check if `event.id` has been processed.
If yes: return 200 immediately (acknowledge receipt without re-processing).
If no: process, then store the event ID.

```
webhook_events table:
  stripe_event_id  (unique)
  event_type
  processed_at
  created_at
```

### Minimum Events to Handle

You MUST handle all of these -- no exceptions:

| Event | What to Do |
|---|---|
| `checkout.session.completed` | Create subscription record in DB, grant access, send welcome email |
| `customer.subscription.created` | Sync subscription status to DB |
| `customer.subscription.updated` | Sync status, plan, period to DB |
| `customer.subscription.deleted` | Mark canceled in DB, revoke access |
| `invoice.payment_succeeded` | Record payment, extend access period |
| `invoice.payment_failed` | Update status to `past_due`, send payment failure email |
| `invoice.upcoming` | (Optional) Send renewal reminder email |
| `customer.subscription.trial_will_end` | Send trial ending soon email (3 days before) |
| `payment_intent.payment_failed` | Log failure, notify user if card declined |

### Webhook Response Rules

- Always return HTTP 200 to Stripe within 30 seconds
- If you receive a 200 from Stripe on your webhook endpoint, Stripe considers delivery successful
- If your handler throws an error mid-processing: do NOT return 200 -- return 500 so Stripe retries
- But make handlers idempotent so retries are safe

---

## 6. Subscription Management Patterns

### Creating a Subscription

Always use `checkout.session.create` with mode `'subscription'` for new subscriptions.
Never create subscriptions directly on the server -- let the user complete checkout.
Stripe handles 3DS, failed cards, and SCA challenges in the hosted flow.

### Upgrading / Downgrading

When a user changes their plan, use `stripe.subscriptions.update`:
- Set `items[0].price` to the new price ID
- Set `proration_behavior` to `'always_invoice'` for immediate billing or `'create_prorations'` for credit
- Show the user a proration preview (`stripe.invoices.retrieveUpcoming`) BEFORE confirming

### Canceling

Two types:
1. Immediate cancellation: `cancel_at_period_end: false` -- access revoked now, no refund
2. End of period: `cancel_at_period_end: true` -- access until period ends, no more renewals

Default to end-of-period cancellation. Provide immediate cancellation only with explicit confirmation.

### Pausing

Use Stripe's pause_collection feature for subscriptions that should pause but not cancel.
Store the pause status in your DB.

---

## 7. Refunds

NEVER initiate a refund automatically without explicit human or user action.
Refund amount must be validated: partial refund cannot exceed original charge.

Rules:
- Full refund: refund the payment intent, no amount needed
- Partial refund: specify amount in cents, validate it <= original amount
- Log every refund: who initiated it, reason, amount, timestamp
- Stripe does not auto-credit your user's DB wallet -- you must handle that logic separately

---

## 8. Testing Strategy for Payments

NEVER test payments against the live Stripe API in development.
ALWAYS use Stripe test mode with test API keys.

### Stripe CLI for Webhook Testing

To receive webhooks locally:
```
stripe listen --forward-to localhost:3000/webhooks/stripe
```

This gives you a `whsec_...` secret for `STRIPE_WEBHOOK_SECRET` in `.env`.

### Test Card Numbers to Cover

| Scenario | Card Number |
|---|---|
| Successful payment | 4242 4242 4242 4242 |
| Requires 3DS authentication | 4000 0025 0000 3155 |
| Card declined | 4000 0000 0000 9995 |
| Insufficient funds | 4000 0000 0000 9995 |
| Card declined after attach | 4000 0000 0000 0341 |
| Subscription past due | Use `stripe trigger invoice.payment_failed` |

### Test Scenarios Required

- [ ] New user subscribes -> subscription created in DB, access granted
- [ ] Subscription payment succeeds -> period extended in DB
- [ ] Subscription payment fails -> status set to `past_due`, email sent
- [ ] User upgrades plan -> proration invoice created
- [ ] User cancels -> status set to cancel at period end
- [ ] User reactivates before period end -> cancel_at_period_end reversed
- [ ] Webhook delivered twice -> second delivery is no-op (idempotency)
- [ ] Webhook with invalid signature -> rejected with 400

---

## 9. Go-Live Checklist

Before switching from test to live Stripe:
- [ ] Webhook endpoint registered in Stripe dashboard (live mode)
- [ ] `STRIPE_SECRET_KEY` is the live `sk_live_...` key (in production env only)
- [ ] `STRIPE_WEBHOOK_SECRET` is the live `whsec_...` secret
- [ ] All price IDs are live mode prices, not test mode
- [ ] Test a real payment end-to-end with a real card (then refund)
- [ ] Payment failure emails tested with Stripe's test failure trigger
- [ ] SSL/HTTPS enforced on the webhook endpoint
- [ ] Rate limiting applied to payment endpoints
- [ ] Idempotency verified in staging by sending duplicate webhook events

---

## 10. What to NEVER Do

- Never store raw card data -- that's Stripe's job
- Never use redirect URL as payment confirmation -- use webhooks
- Never process webhook events without verifying the signature
- Never hardcode price amounts -- use Stripe Price IDs
- Never create duplicate customers -- always check for existing `stripe_customer_id`
- Never show Stripe's raw error messages to users -- translate to friendly messages
- Never skip idempotency on webhook handlers
- Never call Stripe API without a try/catch

---

## 11. Cross-References

- Error types: `PaymentError` -> see `00_error_system`
- Queue for sending emails after payment -> see scenario `SC05_background_jobs`
- Subscription status on user record -> see `02_database_skill`
- Environment variables -> see `00_environment_config`
- Testing webhooks -> see `05_testing_skill`
