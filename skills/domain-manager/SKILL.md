---
name: domain-manager
description: |
  Find, register, and manage custom domains for portfolios. Unified flow
  from search to purchase to DNS setup to ongoing management. Handles
  rebinding domains between portfolios and checking domain status.
  Use when the user says "find a domain", "register a domain", "buy a domain",
  "check my domain", "move my domain", "what domains do I have", "domain status",
  or anything domain-related.
---

# Domain Manager

The unified domain skill. Covers the entire lifecycle: search, purchase, DNS setup, and ongoing management. Conversational, step-by-step, with confirmation before any money is spent.

## Phase 1: Understand (ask before searching)

Before running any search, gather context. Ask questions ONE AT A TIME via AskUserQuestion.
Never assume a name from the user's git config, email, or system context. Always ask.

### Question 1: What's it for?

Ask: "What are you building this portfolio for?"

Options:
- A) College applications — need to look professional
- B) Job hunting / internships — need to stand out
- C) Side project / creative work — want something fun and memorable
- D) Just experimenting — want the cheapest option

### Question 2: Name ideas

Ask: "What name do you want for your domain?"

If they're not sure, offer to help brainstorm. Ask about their name, project, or what
they do. Suggest 5-8 options across styles (full name, short name, creative, role-based).
Let them pick or type their own.

Clean up the input: lowercase, remove spaces/special chars except hyphens, strip any TLD.

### Question 3: Budget

Ask: "What's your budget for the domain?"

Options:
- A) Under $5/year — cheapest possible
- B) Under $10/year — good value
- C) Under $20/year — includes .com
- D) Don't care about price — show me the best options

### Skip questions if context is clear

If the user already provided name AND budget (e.g., "search for reyanmakes under $10"),
skip answered questions. Only ask what's missing.

If the user says "just search" or seems impatient, skip to Phase 2 with defaults.

If the user's intent is clearly management (e.g., "what domains do I have", "move my domain"),
skip directly to Phase 5: Manage.

## Phase 2: Search

```
domains search
{
  "name": "{name}",
  "budget": 10,
  "variants": true,
  "tlds": "com,xyz,me"
}

→ { "success": true, "results": [{ "domain": "{name}.com", "available": true, "price": 14, "currency": "USD" }] }
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | string | Required. Domain name to search. |
| `budget` | number | Optional. Max price per year in USD. |
| `variants` | boolean | Optional. Generate name variants (my-name, the-name, name-dev, etc.) |
| `tlds` | string | Optional. Comma-separated TLD list (e.g., "com,xyz,me") |

### Map user answers to search parameters:

| Budget answer | API parameter |
|---|---|
| Under $5/year | `"budget": 5` |
| Under $10/year | `"budget": 10` |
| Under $20/year | `"budget": 20` |
| Don't care | (no budget param, default 9 TLDs) |

If .com is taken in the results, add `"variants": true` and re-search to suggest alternatives.

## Phase 3: Recommend

After showing results, give a specific recommendation based on their Phase 1 answers:

- **College applications**: "For college apps, .com looks the most professional.
  If it's taken, .me is perfect for a personal portfolio."
- **Job hunting**: "Recruiters trust .com. If budget is tight, .link or .xyz
  still look professional on a resume."
- **Side project**: ".xyz or .click are fun and cheap. Save money for what matters."
- **Experimenting**: "Grab .click for $3. If you love it, upgrade to .com later."

### Iterate if needed

If the user doesn't like any results:
- Offer to brainstorm different names
- Try variations: set `"variants": true` in the search
- Expand budget: "Want me to show options up to $20/year?"
- Try a completely different name

Keep going until they find one they like.

## Phase 4: Register

Once the user picks a domain, walk through registration step by step.

### Step 1: Confirm intent

Before spending money, make sure the user knows exactly what will happen. Ask via AskUserQuestion:

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

### Step 2: Pre-flight check

```
domains check
{ "domain": "{domain}" }
→ { "domain": "{domain}", "available": true, "price": 3, "currency": "USD" }
```

Checks domain availability and price. If the check fails, stop and tell the user what's wrong.

### Step 2.5: Payment

The register API returns **402 Payment Required** if unpaid. Before registering:

```
billing checkout
{ "product_type": "domain_registration", "domain": "{domain}" }
→ { "checkout_url": "...", "amount_cents": 1200 }
```

Tell the user the price and share the checkout link. Wait for confirmation, then verify:

```
billing status --product domain_registration --domain {domain}
→ { "paid": true }
```

In dev mode (`DOMAIN_DEV_MODE=true`), the 402 gate is skipped.

### Step 3: Register

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

### Step 4: Set up DNS

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

### Step 5: ICANN verification (platform-side)

After registration, Route 53 sends an ICANN verification email to the platform's
registrant email (configured in `DOMAIN_CONTACT_JSON`). The platform operator must
click the verification link within 15 days or the domain gets suspended.

This is NOT the customer's responsibility — the email goes to the platform, not the user.
Do NOT tell the user to check their email. Instead tell them:
"Your domain is registered. We'll handle the ICANN verification on our end — no action needed from you."

### Step 6: Verify

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

### Step 7: Report

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

## Phase 5: Manage

For ongoing domain management. Jump here directly when the user's intent is management,
not search or purchase.

### List domains

"What domains do I have?"

```
domains list
→ { "domains": [{ "domain": "{domain}", "portfolioId": "{portfolioId}", "status": "live" }] }
```

Show a clean table: domain, status, linked portfolio.

### Move domain to a different portfolio

"Move my domain to a different portfolio" or "rebind my-portfolio.click to portfolio X"

```
domains bind
{ "domain": "{domain}", "portfolioId": "{portfolioId}" }
→ { "success": true, "domain": "{domain}", "portfolioId": "{portfolioId}" }
```

Confirm before rebinding: "This will point **<domain>** to portfolio **<name>** instead of **<current>**. Proceed?"

### Check if domain is live

"Is my domain working?" or "check example.click"

```
domains verify
{ "domain": "{domain}" }
→ { "domain": "{domain}", "dns_resolves": true, "https_works": true, "db_status": "live" }
```

If not resolving, suggest waiting (DNS propagation) or checking nameserver configuration.

### Unbind domain

"Remove my domain" or "unbind example.click"

```
domains unbind
{ "domain": "{domain}" }
→ { "success": true, "domain": "{domain}" }
```

Confirm before unbinding: "This will disconnect **<domain>** from your portfolio. The domain stays registered but won't point anywhere. Proceed?"

### Check registration status

"Is my domain registered yet?" (for in-progress registrations)

```
domains registration-status
{ "operation_id": "abc123" }
→ { "operation_id": "abc123", "status": "successful" | "in_progress" | "failed" }
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
| Register | Domain unavailable | Someone grabbed it. Search for alternatives |
| Register | Still IN_PROGRESS | Normal. Poll via registration-status API. |
| Register | Timeout (>5 min) | Registration continues on AWS. Poll later. |
| DNS setup | Zone already exists | Domain may have a stale zone in Cloudflare. Delete it first. |
| Nameservers | Update failed | Check Route 53 console for domain lock status |
| Verify | Not resolving | Wait. DNS propagation takes up to 48 hours. |
| Bind | Portfolio not found | Check portfolioId is correct via portfolios summary |
| Unbind | Domain not bound | Nothing to do. Domain is already unbound. |

## Key rules

- **Always confirm before registering.** This costs real money.
- **Always run pre-flight first.** Don't register with broken credentials.
- **ICANN email must be verified within 15 days** or the domain gets suspended.
- **Auto-renew is OFF** for test domains. Turn ON for production domains.
- **WHOIS privacy always enabled.** Platform info, not student info.
- **Route 53 Domains API only works in us-east-1.**
- **Payment required.** Stripe checkout must complete before registration (Phase 4, Step 2.5).
- **Confirm before rebinding or unbinding.** These change what users see at a live URL.

## Requirements

Environment variables needed:
- `AWS_ACCESS_KEY_ID` — AWS credentials for Route 53
- `AWS_SECRET_ACCESS_KEY` — AWS credentials for Route 53
- `CLOUDFLARE_API_TOKEN` — Zone:Zone:Edit + Zone:DNS:Edit permissions
- `CLOUDFLARE_ACCOUNT_ID` — Cloudflare account ID
- `DOMAIN_CONTACT_JSON` — JSON string with ICANN registrant contact info

IAM permissions: `AmazonRoute53FullAccess` + `AmazonRoute53DomainsFullAccess`
