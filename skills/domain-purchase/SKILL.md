---
name: domain-purchase
description: |
  Register a domain via AWS Route 53 and set up Cloudflare DNS to point it
  to the CuriousCirkits portfolio app. Handles the full chain: register,
  create Cloudflare zone, set nameservers, add A record, verify resolution.
  Use when the user asks to "buy a domain", "register a domain", "claim this
  domain", "set up a domain", or wants to connect a domain to their portfolio.
  Always run /domain-search first to find available domains.
---

# Domain Purchase

Register a domain and connect it to the user's portfolio. Conversational, step-by-step, with confirmation before any money is spent.

## Phase 1: Pre-flight check + Confirm intent

First, check availability and get the price. Then confirm with the user before spending money.

"I'm going to register **<domain>** for **$<price>/year**. Here's what happens:

1. AWS Route 53 registers the domain (charged to the platform's AWS account)
2. Cloudflare sets up DNS and TLS (automatic, free)
3. The domain points to your published portfolio on Vercel

This costs real money. The domain is registered in CuriousCirkits' name with WHOIS privacy enabled. You own it via our terms of service.

Ready to proceed?"

Options:
- A) Yes, register it
- B) Wait, let me check something first
- C) Cancel

If the user hasn't run `/domain-search` first, suggest they do that to confirm availability and pricing. Don't assume a domain is available.

### Step 1: Pre-flight check

Run this FIRST to get availability and price before showing the confirmation:

```
domains check
{ "domain": "{domain}" }
→ { "domain": "{domain}", "available": true, "price": 3, "currency": "USD" }
```

If unavailable, stop and suggest `/domain-search` for alternatives. If available, proceed to confirmation.

### Step 2: Confirm with user

Use the price from the pre-flight check in the confirmation message (above).

## Phase 2.5: Payment

The domain register API returns **402 Payment Required** if the user hasn't paid.
Before registering, create a Stripe checkout session and wait for payment.

```
billing checkout
{ "product_type": "domain_registration", "domain": "{domain}" }
→ { "checkout_url": "https://checkout.stripe.com/...", "session_id": "cs_...", "amount_cents": 1200 }
```

Tell the user: "To register {domain}, you'll need to complete payment ($X).
Here's your checkout link: {checkout_url}"

Wait for the user to confirm they've paid, then verify:

```
billing status --product domain_registration --domain {domain}
→ { "paid": true, "payment": { "amount_cents": 1200, "paid_at": "..." } }
```

If `paid: false`, the register call will fail with 402. Don't proceed until paid.

**In dev mode** (`DOMAIN_DEV_MODE=true`), the 402 gate is skipped. You can register
without payment for testing.

## Phase 3: Register

Only after user confirmation AND pre-flight passes AND payment confirmed.

```
domains register
{ "domain": "{domain}", "portfolioId": "{portfolioId}" }
→ { "success": true, "domain": "{domain}", "status": "in_progress", "operation_id": "abc123" }
```

Registration is async. The API returns immediately with an `operation_id`. Poll for completion:

```
domains registration-status
{ "operation_id": "abc123" }
→ { "operation_id": "abc123", "status": "successful" | "in_progress" | "failed" }
```

If registration fails, suggest:
- Check AWS account has sufficient funds
- Verify the domain wasn't claimed in the last few minutes
- Try a different domain

## Phase 4: Set up DNS

After registration succeeds (status = "successful").

```
domains setup-dns
{ "domain": "{domain}" }
→ { "success": true, "domain": "{domain}", "nameservers": ["ns1.cloudflare.com", "ns2.cloudflare.com"], "nameservers_set": true }
```

This single call does 3 things: creates Cloudflare zone + CNAME, sets nameservers on Route 53, adds domain to Vercel.

**Important:** CNAME not A record. Proxy must be OFF. Vercel needs the Host header
to match the domain. Cloudflare proxy rewrites it, causing 525 SSL errors.

If Cloudflare zone creation fails, check:
- Cloudflare token has Zone:Zone:Edit + Zone:DNS:Edit permissions
- The domain doesn't already have a zone in Cloudflare

## Phase 5: ICANN verification (platform-side)

After registration, Route 53 sends an ICANN verification email to the platform's
registrant email (configured in `DOMAIN_CONTACT_JSON`). The platform operator must
click the verification link within 15 days or the domain gets suspended.

This is NOT the customer's responsibility — the email goes to the platform, not the user.
Do NOT tell the user to check their email. Instead tell them:
"Your domain is registered. We'll handle the ICANN verification on our end — no action needed from you."

## Phase 6: Verify

```
domains verify
{ "domain": "{domain}" }
→ { "domain": "{domain}", "dns_resolves": true, "https_works": true, "db_status": "live" }
```

Checks DNS resolution and HTTPS availability. The API also updates the DB status to "live" if both pass.

Tell the user:
- Cloudflare-side resolves in minutes
- Public DNS can take up to 48 hours
- TLS cert is automatic via Cloudflare

## Phase 7: Report

After all steps complete, show the summary:

```
══════════════════════════════════════
  DOMAIN REGISTERED
══════════════════════════════════════
  Domain:      <domain>
  Price:       $<price>/year
  Points to:   Vercel (76.76.21.21)
  TLS:         Automatic (Cloudflare)
  Status:      LIVE (or PROPAGATING)
══════════════════════════════════════

  Your portfolio is now accessible at https://<domain>
```

## Important: Registration is async

Route 53 domain registration is NOT instant. It takes 1-15 minutes (sometimes longer for
some TLDs). The API polls internally.

If the user cancels or the session ends during polling:
- The registration IS still happening on AWS. Money has been charged.
- Check status via the registration-status API
- Don't re-register. Check status first.

Always tell the user: "The registration has been submitted. Even if this window closes,
the domain IS being registered. You can check status anytime."

## Error recovery

| Step | Error | What to do |
|------|-------|------------|
| Pre-flight | AWS creds missing | Check env vars: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY |
| Pre-flight | CF token invalid | Create new token at dash.cloudflare.com/profile/api-tokens |
| Register | Domain unavailable | Someone grabbed it. Run `/domain-search` for alternatives |
| Register | Still IN_PROGRESS | Normal. Poll via registration-status API. |
| Register | Timeout (>5 min) | Registration continues on AWS. Poll later. |
| DNS setup | Zone already exists | Domain may have a stale zone in Cloudflare. Delete it first. |
| Nameservers | Update failed | Check Route 53 console for domain lock status |
| Verify | Not resolving | Wait. DNS propagation takes up to 48 hours. |

## Key rules

- **Always confirm before registering.** This costs real money.
- **Always run pre-flight first.** Don't register with broken credentials.
- **Auto-renew is OFF** for test domains. Turn ON for production domains.
- **WHOIS privacy always enabled.** Platform info, not student info.
- **Route 53 Domains API only works in us-east-1.**
- **Payment required.** Stripe checkout must complete before registration (Phase 2.5).

## Requirements

Environment variables needed:
- `AWS_ACCESS_KEY_ID` — AWS credentials for Route 53
- `AWS_SECRET_ACCESS_KEY` — AWS credentials for Route 53
- `CLOUDFLARE_API_TOKEN` — Zone:Zone:Edit + Zone:DNS:Edit permissions
- `CLOUDFLARE_ACCOUNT_ID` — Cloudflare account ID
- `DOMAIN_CONTACT_JSON` — JSON string with ICANN registrant contact info

IAM permissions: `AmazonRoute53FullAccess` + `AmazonRoute53DomainsFullAccess`
