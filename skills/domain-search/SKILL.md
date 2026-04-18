---
name: domain-search
description: |
  Search for available domain names across multiple TLDs and show pricing.
  Starts by understanding what the user is building and their budget, then
  searches intelligently using AWS Route 53. Conversational, not robotic.
  Use when the user asks to "search for a domain", "find a domain", "check if
  a domain is available", "domain search", "what domains are available for",
  "cheapest domain", "domain under $X", or mentions buying/registering a domain.
  Do NOT trigger for DNS troubleshooting, domain configuration (pointing to Vercel,
  setting up nameservers), or registrar comparison questions — those are not domain searches.
---

# Domain Search

Help users find the perfect domain name. Understand first, search smart, recommend well.

## Input cleanup (always apply before any search)

Before passing any name to the API, always clean it up:
- Lowercase the input
- Remove spaces and special characters (keep hyphens)
- Strip any TLD suffix (e.g., "reyanmakes.com" → "reyanmakes")

This applies on every path — Phase 1 questions, skip-to-search, and fast-track.

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

### Question 3: Budget

Ask: "What's your budget for the domain?"

Options:
- A) Under $5/year — cheapest possible
- B) Under $10/year — good value
- C) Under $20/year — includes .com
- D) Don't care about price — show me the best options

### Skip questions if context is clear

If the user already provided name AND budget (e.g., "search for reyanmakes under $10"),
skip ALL questions and go straight to Phase 2. Don't ask purpose — infer a neutral
recommendation or use "Don't care about price" defaults. The user is action-oriented;
respect their momentum.

If the user says "just search" or seems impatient, skip to Phase 2 with defaults.

### Fast-track purchase intent

If the user names an exact domain with TLD (e.g., "I want avikdebnath.com",
"is reyanmakes.com available?"), they already know what they want. Skip ALL Phase 1
questions. Go straight to Phase 2 — check availability for that specific domain
(plus a few alternative TLDs). If available, immediately hand off to Phase 5
(`/domain-purchase`). Only fall back to Phase 1 questions if the domain is taken
and they need alternatives.

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

For low budgets ($5 or less), omit the `tlds` parameter so the API searches all available
TLDs and the budget filter can surface genuinely cheap options. Don't limit to `com,xyz,me`
when the user wants the cheapest — there are many affordable TLDs the API knows about
that you might not.

If .com is taken in the results, add `"variants": true` and re-search to suggest alternatives.

## Phase 3: Recommend

After showing results, give a specific recommendation based on their Phase 1 answers:

- **College applications**: "For college apps, .com looks the most professional.
  If it's taken, .me is perfect for a personal portfolio."
- **Job hunting**: "Recruiters trust .com. If budget is tight, .link or .xyz
  still look professional on a resume."
- **Side project**: ".xyz or .click are fun and cheap. Save money for what matters."
- **Experimenting**: "Grab .click for $3. If you love it, upgrade to .com later."

## Phase 4: Iterate

If the user doesn't like any results:
- Offer to brainstorm different names
- Try variations: set `"variants": true` in the search
- Expand budget: "Want me to show options up to $20/year?"
- Try a completely different name

Keep going until they find one they like.

## Phase 5: Hand off

After they pick a domain: "Run `/domain-purchase` to register it and connect it to your portfolio."

This skill only searches. It never registers or charges anything.

## Requirements

- Environment variables: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
- IAM permissions: `AmazonRoute53DomainsFullAccess`
- Region: `us-east-1` (the API handles this)
