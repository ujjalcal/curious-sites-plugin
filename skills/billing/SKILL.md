---
name: billing
description: |
  Handle payments for CuriousCirkits services via Stripe Checkout. Currently
  supports domain registration payments. Creates checkout sessions, verifies
  payment status, and gates paid actions (like domain registration) behind
  payment confirmation.
  Use when the user hits a 402 Payment Required error, asks about payment,
  says "pay for my domain", "checkout", "billing status", or when domain
  registration fails due to missing payment. Also triggered by domain-purchase
  and domain-manager skills during their payment flow (Phase 2.5).
  Do NOT trigger for pricing questions ("how much does a domain cost?") — those
  are handled by domain-search.
---

# Billing

Handle payments through Stripe Checkout. The agent creates a checkout session,
gives the user a link, waits for them to pay, then verifies payment before
proceeding. Simple and linear.

## When this skill runs

This skill is usually called mid-flow by another skill (domain-purchase or
domain-manager) when they hit the payment gate. It can also be invoked directly
if a user asks about billing or hits a 402 error.

## Phase 1: Create checkout session

```
billing checkout
{
  "product_type": "domain_registration",
  "domain": "example.com"
}

→ { "checkout_url": "https://checkout.stripe.com/...", "session_id": "cs_...", "amount_cents": 1200, "domain": "example.com" }
```

| Field | Type | Description |
|-------|------|-------------|
| `product_type` | string | Required. Currently only `"domain_registration"` is supported. |
| `domain` | string | Required for domain_registration. The domain being purchased. |

### Error handling

| Response | Meaning | What to do |
|----------|---------|------------|
| 200 | Session created | Show checkout URL to user |
| 400 | Bad request | Check product_type and domain params |
| 401 | Not authenticated | User needs to log in first |
| 409 | Already paid | Skip checkout — payment already exists. Proceed to registration. |

**If the API returns 409 (already paid):** Don't create a new session. Tell the user
they've already paid for this domain and proceed directly to registration.

## Phase 2: Present to user

Show the checkout link clearly. Don't bury it in text.

```
"To register {domain}, complete payment (${amount_cents / 100}):

  {checkout_url}

Open the link, complete checkout, then come back and tell me when you're done."
```

The checkout URL opens Stripe's hosted page. The user pays there, not in this app.
After payment, Stripe sends a webhook that marks the payment as "paid" in the database.

**Do NOT auto-open the URL.** Present it as a link. The user clicks it themselves.

## Phase 3: Verify payment

After the user says they've paid (or after a reasonable wait):

```
billing status --product domain_registration --domain example.com

→ { "paid": true, "payment": { "id": "uuid", "amount_cents": 1200, "currency": "usd", "paid_at": "2026-04-17T..." } }
→ { "paid": false, "payment": null }
```

**If `paid: true`:** Payment confirmed. Proceed with the gated action (domain registration).
Tell the user: "Payment confirmed. Proceeding with registration."

**If `paid: false`:** Payment hasn't been processed yet. This can happen if:
- The user hasn't completed checkout
- The Stripe webhook hasn't fired yet (takes a few seconds)
- The user cancelled checkout

Tell the user: "Payment hasn't been confirmed yet. Did you complete the checkout?
If you just finished, give it a moment — it can take a few seconds to process."

Wait for the user to respond before checking again. Don't poll aggressively.

## Phase 4: Return to calling skill

If this skill was invoked by domain-purchase or domain-manager, return control
after payment is confirmed. The calling skill resumes at the registration step.

If invoked directly (user asked about billing), just report the status and stop.

## Dev mode

When `DOMAIN_DEV_MODE=true` is set, the 402 payment gate on domain registration
is skipped entirely. No checkout session needed. This is for local testing only.

If you're in dev mode and the user asks about payment, tell them:
"Dev mode is active — payment is skipped for testing. In production, Stripe checkout
is required before domain registration."

## Supported product types

Currently only `domain_registration` is supported. The billing system is designed
to expand — the `product_type` and `product_ref` fields in the payments table allow
for future product types (premium templates, custom designs, etc.) without schema changes.

## Key rules

- **Never store or handle card details.** Stripe Checkout handles all payment UI.
- **Always verify via the status API.** Don't assume payment succeeded because the
  user said "I paid." The webhook updates the database; the status API reads it.
- **409 means already paid.** Don't ask the user to pay again. Skip to registration.
- **Don't poll aggressively.** Check once after the user says they paid. If still
  pending, wait for them to confirm before checking again.
- **Webhook handles the state change.** The agent never marks payments as paid.
  The Stripe webhook (`billing webhook (internal)`) does that automatically.

## Requirements

Environment variables:
- `STRIPE_SECRET_KEY` — Stripe API key
- `STRIPE_WEBHOOK_SECRET` — Webhook signing secret
- `NEXT_PUBLIC_APP_URL` — Base URL for success/cancel redirects
