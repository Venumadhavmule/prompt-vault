# WF04 -- Payment Integration Workflow (Stripe)

## Purpose

Step-by-step runbook for integrating Stripe payments. Covers subscription billing
and one-time payments. Stripe integration has more failure modes than most integrations
because it involves webhooks, async state changes, and real money. Follow every step.

---

## Pre-conditions

- Stripe account exists (test mode for development)
- Stripe CLI installed for local webhook testing
- Environment variable management in place
- User/organization model exists in the database

---

## Step 1: Install dependencies

```bash
pnpm add stripe
pnpm add -D @types/stripe
```

---

## Step 2: Add environment variables

`.env`:
```
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRICE_ID_PRO_MONTHLY=price_...
STRIPE_PRICE_ID_PRO_YEARLY=price_...
```

`.env.example`:
```
STRIPE_SECRET_KEY=sk_test_your_key_here
STRIPE_WEBHOOK_SECRET=whsec_your_webhook_secret_here
STRIPE_PRICE_ID_PRO_MONTHLY=price_monthly_plan_id
STRIPE_PRICE_ID_PRO_YEARLY=price_yearly_plan_id
```

NEVER commit real Stripe keys. Not even test keys.

---

## Step 3: Create the Stripe client singleton

`src/lib/stripe.ts`:
```typescript
import Stripe from 'stripe';

if (!process.env.STRIPE_SECRET_KEY) {
  throw new Error('STRIPE_SECRET_KEY is not set');
}

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  apiVersion: '2024-06-20',
});
```

---

## Step 4: Update the database schema

Add billing fields to your User or Organization model in `prisma/schema.prisma`:

```prisma
model User {
  // ... existing fields ...
  stripe_customer_id    String?   @unique
  subscription_id       String?   @unique
  subscription_status   String?   // active | past_due | canceled | trialing
  subscription_plan     String?   // pro_monthly | pro_yearly
  current_period_end    DateTime?
}
```

Create migration:
```bash
npx prisma migrate dev --name add_stripe_billing_fields
```

---

## Step 5: Implement the payments service

`src/services/paymentsService.ts`:

Functions to implement:
1. `createCustomer(userId, email, name)` -- create Stripe customer, save stripe_customer_id to DB
2. `createCheckoutSession(userId, priceId, successUrl, cancelUrl)` -- create Stripe hosted checkout
3. `createBillingPortalSession(userId, returnUrl)` -- Stripe customer portal for plan management
4. `getSubscriptionStatus(userId)` -- return current plan and billing state from DB
5. `cancelSubscription(userId)` -- cancel at period end via Stripe API
6. `handleWebhookEvent(event: Stripe.Event)` -- route webhook events to handlers

Reference: `skills/04_payments_skill.md` for full implementation details.

---

## Step 6: Implement webhook handler

This is the most critical part of the integration. All permanent state changes must
happen in the webhook -- never directly at checkout completion, because the user
might close the browser before the redirect completes.

`src/routes/webhooks.ts`:

```
POST /api/webhooks/stripe
- Get raw body (MUST be raw, not parsed JSON)
- Verify signature: stripe.webhooks.constructEvent(body, sig, STRIPE_WEBHOOK_SECRET)
- Return 400 immediately if signature verification fails
- Route to handler by event.type
- Return 200 to Stripe (idempotent handling)
```

Webhook events to handle:

| Event | Action |
|-------|--------|
| `checkout.session.completed` | Save subscription_id, set status to trialing or active |
| `customer.subscription.updated` | Update status, plan, period_end |
| `customer.subscription.deleted` | Set status to canceled, clear plan |
| `invoice.payment_succeeded` | Update period_end, set status to active |
| `invoice.payment_failed` | Set status to past_due, trigger dunning email |

**Critical:** Make all webhook handlers idempotent. Check if the state is already
what the event would set it to before writing to the database. Stripe will retry
events that do not get a 200 response.

---

## Step 7: Implement the billing API endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | /api/billing/checkout | Create checkout session, return URL |
| POST | /api/billing/portal | Create portal session, return URL |
| GET | /api/billing/status | Return subscription status from DB |
| POST | /api/billing/cancel | Cancel at end of period |
| POST | /api/webhooks/stripe | Receive Stripe events (unauthenticated, signature verified) |

The webhook endpoint must be excluded from auth middleware.
The webhook endpoint must receive the raw request body (not parsed).

---

## Step 8: Implement access control

Create a function that checks whether the current user has an active subscription:

```typescript
function hasActiveSubscription(user: User): boolean {
  return (
    user.subscription_status === 'active' ||
    user.subscription_status === 'trialing'
  );
}
```

Use this gate on every route or page that requires a paid plan.

Return `402 Payment Required` with `{"error": "SUBSCRIPTION_REQUIRED"}` for
unauthorized access attempts to paid features.

---

## Step 9: Test locally with Stripe CLI

```bash
# Install Stripe CLI if not already installed
# Then forward webhooks to local server:
stripe listen --forward-to localhost:3000/api/webhooks/stripe

# In another terminal, trigger test events:
stripe trigger checkout.session.completed
stripe trigger customer.subscription.updated
stripe trigger invoice.payment_failed
```

Verify webhook handler processes each event type correctly.

---

## Step 10: Write tests

Tests to write:

1. `paymentsService.test.ts` -- unit tests with mocked Stripe client
   - [ ] createCustomer calls stripe.customers.create and saves ID to DB
   - [ ] createCheckoutSession returns a valid URL
   - [ ] handleWebhookEvent routes events to correct handlers
   - [ ] handleWebhookEvent handles duplicate events idempotently

2. `webhook.routes.test.ts` -- integration tests
   - [ ] POST /webhooks/stripe returns 400 for invalid signature
   - [ ] POST /webhooks/stripe returns 200 for valid event
   - [ ] checkout.session.completed event updates user subscription_id in DB
   - [ ] invoice.payment_failed event sets status to past_due in DB

---

## Step 11: Pre-production security checklist

- [ ] Webhook signature verification is implemented and cannot be bypassed
- [ ] No Stripe keys are in version control
- [ ] Test keys in dev, live keys only in production env vars
- [ ] Webhook endpoint is excluded from CSRF protection
- [ ] Webhook endpoint logs every received event type
- [ ] All webhook handlers are idempotent
- [ ] Access control (hasActiveSubscription) is applied to all paid features
- [ ] Stripe Customer Portal is used for subscription management (do not build custom cancel UI)
- [ ] Failed payment flow tested end-to-end with Stripe test cards

---

## Done definition

- All tests pass
- Local webhook test with Stripe CLI succeeds for all event types
- Checkout flow tested end-to-end in test mode
- Failed payment triggers dunning or status update correctly
- Security checklist complete
- No live Stripe keys in non-production environments
