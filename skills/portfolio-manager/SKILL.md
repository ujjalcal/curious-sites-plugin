---
name: portfolio-manager
description: |
  Manage multiple portfolios — list, organize, rename, clone, delete, check status,
  move domains, and decide what to work on next. The "dashboard brain" for users
  who have built portfolios and need to manage them.
  Use when the user says "show my portfolios", "delete the test ones", "rename this",
  "what should I work on", "move my domain", "clone this", "clean up", or asks about
  their portfolio status.
---

# Portfolio Manager

DAY 2 workflows. Everything after a portfolio is built. List, organize, rename, clone,
delete, rebind domains, check status, recommend what to work on next.

## Phase 1: Overview

Start by loading everything the user has.

```
portfolios summary
→ [{ id, name, subdomain, domain, workflow_progress, created_at, updated_at }]
```

Present in plain language. NO technical details (no subdomains, no paths, no
workflow percentages, no UUIDs). The user is not a developer. They want to know:
what they have, what's live, and what needs work.

**Good:**
```
Active portfolios:
  - {name}'s portfolio — live at {domain}
  - Jane Doe — published, needs a custom domain
  - Executive — published

Test portfolios: 3 empty ones from trying templates
```

**Bad:**
```
  #  Name          Subdomain       Progress
  1  Avik Kumar    avik-kumar      100%
  2  Tanay         tanay-k         80%
```

Group portfolios by status:
- **Live** (has custom domain) → "{name} — live at {domain}"
- **Published** (has subdomain, no domain) → "{name} — published, needs a custom domain"
- **In progress** (some content but not published) → "{name} — started, not published yet"
- **Empty** (0% progress, test portfolios) → count them: "N empty/test portfolios"

Never show: subdomain paths, workflow percentages, UUIDs, created_at dates, or
API response fields. Translate everything into what it MEANS for the user.

If the user has **0 portfolios**, redirect: "You don't have any portfolios yet. Want to build one?" → hand off to `/portfolio-builder`.

If the user has **1 portfolio**, skip "which one?" questions. Act on the only one.

## Phase 2: Route

Based on what the user said, pick the right action.

| User says | Action |
|-----------|--------|
| "show my portfolios" | Phase 1 overview |
| "what should I work on" | Recommend (see below) |
| "delete X" / "clean up" | Delete flow |
| "rename X" | Rename flow |
| "clone X" | Clone flow |
| "move my domain to X" | Domain rebind flow |
| "what domains do I have" | Domain list |
| "show version history" | Version history |
| "unbind my domain" | Domain unbind flow |

If the intent is ambiguous, show the overview first and ask what they want to do.

## Phase 3: Execute

### Delete flow

Identify candidates. Show them. Confirm before deleting anything.

```
"You have 4 portfolios that look like tests (0% progress, no subdomain):
  - test copy
  - untitled
  - My Portfolio (3)
  - asdf

Delete all 4?"
```

Wait for confirmation. Then:

```
portfolios delete {id}    ← for each confirmed portfolio
```

After: "Deleted 4. You have 4 portfolios remaining."

**Never delete without confirmation.** Even if the user says "clean up everything," show
what will be deleted and wait for "yes."

### Rename flow

```
"What do you want to rename 'My Portfolio (Executive)' to?"
→ User: "Executive Portfolio"

portfolios update {id}
{ "name": "Executive Portfolio" }
→ { "success": true }

"Renamed."
```

If the user has multiple portfolios and doesn't specify which one, show the list and ask.

### Clone flow

```
portfolios clone
{ "portfolioId": "source-uuid", "name": "Suggested Clone Name" }
→ { "success": true, "portfolio": { id, name, subdomain } }
```

Suggest a name based on the original: "{name} (copy)" or ask the user.
Tell them: "Cloned. The new portfolio has the same content but a fresh subdomain."

### Domain rebind flow

Show where the domain currently points, then ask where to move it.

```
domains list
→ [{ domain: "{domain}", portfolio_id: "{portfolioId}", portfolio_name: "{name}" }]

"{domain} is currently pointing to '{name}'. Move it to which portfolio?"
→ Show numbered list of user's portfolios
→ User picks one

domains bind
{ "domain": "{domain}", "portfolioId": "{portfolioId}" }
→ { "success": true }

"Done. {domain} now points to '{name}'."
```

### Domain unbind flow

Confirm first. Unbinding means the domain stops resolving to any portfolio.

```
"Unbind {domain} from '{name}'? The domain will stop pointing to any portfolio."
→ User: "yes"

domains unbind
{ "domain": "{domain}" }
→ { "success": true }

"Unbound. {domain} is no longer connected to any portfolio."
```

### Domain list

```
domains list
→ [{ domain, portfolio_id, portfolio_name, status }]
```

Show a clean table:

```
YOUR DOMAINS
═══════════════════════════════════════════════════
  Domain          Points to                Status
  example.click   Jane Doe                 live
  jane.dev        —                        unbound
═══════════════════════════════════════════════════
```

### Version history

```
portfolios get {id}/versions
→ [{ version_id, created_at, published_by }]
```

Show the version list with timestamps. If the user wants to roll back, they can
use the web editor or re-publish a previous version.

### "What should I work on" flow

Look at all portfolios. Find the best candidate to recommend.

Priority order:
1. **Highest progress, not 100%.** Closest to done = lowest effort to finish.
2. **Has content but low progress.** Started but abandoned.
3. **0% with no content.** Probably junk — suggest cleanup instead.

```
"Your 'Jane Doe' portfolio is at 80% — just needs a custom domain.
Want to search for one? Or work on something else?"
```

Always explain WHY you're recommending it: "this one is closest to done,"
"this one has content but needs publishing," etc.

If everything is at 100%: "All your portfolios are complete. Want to build a new one,
or update an existing one?"

## Rules

- **Always confirm before destructive actions.** Delete, unbind — show what will happen, wait for yes.
- **Show counts after batch operations.** "Deleted 8. 4 remaining."
- **When recommending, explain WHY.** "This one is closest to done" beats "work on this."
- **0 portfolios → redirect to /portfolio-builder.** Don't manage what doesn't exist.
- **1 portfolio → skip "which one."** Don't ask obvious questions.
- **Keep it direct.** No fluff, no "Great question!", no emoji. State facts, take action.
