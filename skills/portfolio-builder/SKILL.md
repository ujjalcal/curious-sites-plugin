---
name: portfolio-builder
description: |
  Build a visual portfolio website from messy inputs — resumes, LinkedIn profiles,
  hand-drawn sketches, or verbal descriptions. Generates custom HTML and CSS through
  an iterative conversation loop. The user provides content and structure, the AI builds,
  the user gives feedback, the AI revises. Loop until the user says "publish."
  After publishing, the user can edit content visually in the web editor at /editor.
  Use when the user says "build a portfolio", "create a site", "make a page for",
  "portfolio for [name]", "turn this into a website", or provides a resume/sketch
  and wants it turned into a live page.
---

# Portfolio Builder

Build visual portfolio websites through conversation. The user gives you messy inputs.
You turn them into a live page. They tell you what's wrong. You fix it. Repeat until
they say publish. Then they can edit content visually in the web editor.

**HARD RULE:** Never decide the design is "done." Only the user decides. Your job is to
build, show, and iterate. The loop ends when the user explicitly says to publish or stop.

**Guard:** If the user wants to review, evaluate, or get feedback on an existing portfolio
(not build or update), this is NOT a portfolio-builder request. Hand off to
portfolio-reviewer instead. Same for delete/manage requests — hand off to portfolio-manager.

## What you learned from real usage

This skill was born from a live session where we built a portfolio through 8+ iterations:

- The user's hand-drawn sketch was the best design input (better than any form)
- LinkedIn content was richer than the resume docx for role details
- The user rejected "Evaluating" status badges, flow labels, and decorative text
- The user preferred the version with clear visual hierarchy over the "clean boxes" version
- Every time we added explanatory micro-copy, it got worse
- The user can't make it perfect through AI alone. They need to edit content visually.
- The iteration loop IS the product: AI generates → user reviews → edits visually → publishes
- CSS is the "theme" (agent controls), HTML body is the "content" (user edits)
- Always generate CSS and HTML body as separate concerns. The web editor shows them side by side.

## The Two Editing Surfaces

```
AGENT (Claude Code / future web agent)     USER (web editor at /editor)
─────────────────────────────────────      ────────────────────────────
Generates full HTML + CSS                  Edits text visually (contenteditable)
Handles structure, layout, flow            Fixes typos, updates content
Controls the CSS / theme                   Doesn't touch CSS
Publishes via publish operation             Publishes via Publish button
Iterates via conversation                  Iterates by clicking and typing
```

The agent does the heavy lifting (structure, CSS, layout). The user does the fine-tuning
(content edits, text changes). Both can publish. The web editor at /editor shows:
- Left pane: CSS (the agent's styles)
- Right pane: Visual preview with contenteditable (click any text to edit)

## Phase 0: Check for existing portfolios

Before gathering inputs, check if the user already has portfolios:

```
portfolios summary
```

If the user has existing portfolios, ask:

"You have {N} existing portfolio(s). What would you like to do?"

Options:
- A) Build a new portfolio from scratch
- B) Update an existing one (I'll show you the list)

**If A:** Continue to Phase 1.

**If B:** Show the portfolio list with names and URLs. User picks one.
Then load the current content:

```
portfolios content {id}
→ { html, css, themes }
```

Skip to Phase 3 (Iterate) with the existing HTML loaded. The user gives
feedback on their current portfolio, you revise it, same iteration loop.
When they say "publish," call the publish API with the existing portfolioId.

**If the user's message makes it clear they want a new portfolio** (e.g., "build a
portfolio for my friend"), skip Phase 0 entirely and go to Phase 1.

## Phase 1: Gather Inputs

Collect whatever the user has. Ask ONE question at a time via AskUserQuestion.
Skip questions the user already answered. Accept whatever format they give you.

### Question 1: Who is this for?

Ask: "Who is this portfolio for? Tell me their name and what they do."

Accept: a name, a title, a one-line description, or "me."

### Question 2: Content sources

Ask: "What content do you have? Share whatever you've got — I'll work with it."

Options:
- A) Resume / CV (paste or attach a file)
- B) LinkedIn profile (paste the text)
- C) I'll describe it verbally
- D) Multiple sources (resume + LinkedIn + other)

Accept ANY format: docx, PDF, pasted text, a URL, bullet points, stream of consciousness.
Don't ask for a specific format. Take what they give you.

If they attach a file, read it. If it's a docx, extract text via:
```bash
unzip -p <file> word/document.xml 2>/dev/null | sed -e 's/<[^>]*>//g' -e "s/&amp;/\&/g" -e "s/&lt;/</g" -e "s/&gt;/>/g" -e "s/&apos;/'/g" | tr -s '[:space:]' '\n' | grep -v '^$'
```

### Question 3: Starting point

Ask: "How do you want to start?"

Options:
- A) Pick from a template gallery (10 starter layouts to choose from)
- B) I have a sketch / wireframe (attach image)
- C) Start from scratch — you decide the layout
- D) I want a specific flow / structure (I'll describe it)

**If A (template gallery):** Run `/template-gallery` to show the 10 options. User picks
one. Read the template HTML from `portfolios/templates/<name>.html`. Split into CSS
and body. Use the body structure as the skeleton — replace placeholder content with
the user's real content. Generate 4 CSS theme variants based on the template's CSS.

**If B (sketch):** READ IT CAREFULLY. Map every box, arrow, and connection.
The sketch is the design brief. Don't interpret it loosely — match the structure exactly.

**If C (from scratch):** Start with a clean single-column layout. Hero → about →
experience → skills → contact. They'll redirect from there.

**If D (described structure):** Build exactly what they describe.

### Question 4: Theme preference

Ask: "Any preference for the overall feel? I'll generate 4 theme variants you can
switch between in the editor."

Options:
- A) Professional / corporate feel
- B) Creative / bold feel
- C) You decide — surprise me

Don't overthink this. You generate 4 themes regardless. This just biases which one
is the default (first in the list).

### Skip logic

If the user provides everything upfront ("here's my resume, make a portfolio
with a white theme"), skip to Phase 2 immediately. Only ask questions whose answers
aren't clear from context.

If the user says "just build it" at any point, stop asking and start building with
what you have.

## Phase 2: Build First Draft

### Step 1: Analyze the inputs

Before writing any HTML, produce a brief content map. This ensures you and the user
agree on what goes on the page.

```
CONTENT MAP
═══════════
Name: [name]
Title: [current role]
Theme: [light/dark]

Sections (top to bottom):
1. [section name] — [what goes here]
2. [section name] — [what goes here]
...

Key content:
- [N] roles in experience
- [N] skills/expertise areas
- [N] projects/achievements
- Contact: [what links]

Structure: [describe the flow — linear, two-track, timeline, etc.]
```

Ask via AskUserQuestion: "Here's what I plan to build. Look right?"

Options:
- A) Looks good, build it
- B) Change something (I'll specify)

### Step 2: Write the HTML + Generate Theme Variants

Generate a single self-contained HTML file with CSS and body as separate concerns.

**CSS goes in `<style>` tags in `<head>`.** This is the theme. The agent controls it.
**Content goes in `<body>`.** This is what the user edits visually in the web editor.

Keep them cleanly separated. Don't use inline styles on body elements. All visual
styling goes in the `<style>` block using CSS classes.

**Generate 3-4 CSS theme variants** for the same HTML body. Each theme is a complete
`<style>` block that changes the look without touching the body structure. Examples:
- **Clean** — white background, system fonts, subtle grey accents
- **Bold** — dark background, high contrast, blue accents
- **Warm** — cream/beige background, serif fonts, earthy tones
- **Minimal** — lots of whitespace, thin lines, muted colors

Save the default theme (first variant) in the HTML file's `<style>` tag. Save ALL
variants in a companion JSON file AND embed them in the Supabase content JSON on publish.

```bash
# Save theme variants alongside the HTML (source of truth in repo)
cat > "portfolios/<name>-themes.json" << 'EOF'
[
  { "name": "Clean", "css": "..." },
  { "name": "Bold", "css": "..." },
  { "name": "Warm", "css": "..." },
  { "name": "Navy", "css": "..." }
]
EOF
```

**Critical: On publish, include themes in the API call.** The web editor reads `themes`
from the `portfolio_versions` row (stored as JSONB). Without it, no theme pills appear.

The `publish` operation accepts a `themes` field: an array of `{name, css}` objects.
Pass the same array you saved in `<name>-themes.json`.

The web editor loads `themes` on mount and renders pill buttons. Clicking a pill
swaps the `<style>` tag in the iframe instantly (no re-render, no scroll reset).

**Theme CSS rules:**
- Every theme must use the exact same CSS class names as the HTML body
- Only change colors, fonts, backgrounds, borders, spacing
- Never change layout structure (grid columns, flex direction)
- Include the mobile responsive `@media` block in every theme
- Test: switching themes should never break the layout, only change the look

Rules:
- **All CSS in a `<style>` tag in `<head>`.** No external stylesheets, no CDN links.
- **No inline styles on body elements.** Use CSS classes instead. This lets the user
  edit body content without accidentally breaking styles.
- **No JavaScript frameworks.** Vanilla JS only, and only for interactions (expand/collapse).
- **Mobile responsive.** Use CSS grid/flexbox. Include a 640px breakpoint.
- **Self-contained.** The HTML file is the entire portfolio. No dependencies.
- **Match the sketch.** If a sketch was provided, the HTML structure must mirror it exactly.

Design principles (learned from real usage):
- **Structure IS the argument.** If you need text to explain the flow, the flow isn't working.
- **No decorative micro-copy.** No "Where Both Tracks Meet," no "All Converges Here."
  The visual hierarchy communicates the relationships.
- **Expandable detail on click.** Default view is clean. Click to reveal full content.
- **Minimal connector lines.** Thin, grey, subtle. Color only for the ONE focus area.
- **Real content, not lorem ipsum.** Use the actual content from the inputs.

### Step 3: Save the HTML

```bash
# In the repo (source of truth)
mkdir -p portfolios
# Filename: kebab-case of the person's name
FILENAME="portfolios/$(echo '<name>' | tr '[:upper:]' '[:lower:]' | sed 's/ /-/g').html"
```

### Step 4: Show the user

Tell them: "First draft is ready at `<filename>`. Open it in your browser:
`open <filename>`"

If a Cloudflare Worker is deployed and a domain binding exists, also publish to Supabase
and tell them the live URL. See "Publishing" section below.

After publishing, tell them: "You can also edit the content visually at /editor.
Toggle to 'Custom HTML' mode. Click any text in the preview to change it."

Then STOP. Wait for feedback. Do not ask "what do you think?" — just wait.

## Phase 3: Iterate

This is the core of the skill. The user gives feedback. You revise. Repeat.

### How to handle feedback

| User says | What to do |
|-----------|-----------|
| "the flow is wrong" | Re-read the sketch. Ask which specific connection is off. |
| "make it [color]" | Change the CSS only. Don't touch the body HTML. |
| "this section looks bad" | Fix that section only. Don't touch the rest. |
| "add links" | Add href attributes to the relevant elements. |
| "the content is wrong/missing" | Compare against the source material. Fix the gap. |
| "it's ugly" | Ask: "The layout, the colors, the typography, or the spacing?" |
| "I liked the previous version" | Re-publish the previous version: `portfolios content {id}` to load last published HTML, then publish it again. |
| "make it interactive" | Add expand/collapse with `onclick="this.classList.toggle('open')"` |
| "publish" | Go to Phase 4. |
| "good enough" / "let's move on" | Go to Phase 4. |

### Rules for iteration

1. **Edit the existing HTML.** Don't rewrite from scratch unless asked.
2. **CSS changes = theme changes.** Body changes = content changes. Keep them separate.
3. **Re-publish after each meaningful change.** Call `publish {portfolioId}` with the updated HTML.
   Each publish creates a new version in the database. Previous versions are preserved.
4. **Never say "done."** The user ends the loop, not you.
5. **Count iterations.** After 5+, gently note: "We're on iteration N. Want to checkpoint?"

### What NOT to do during iteration

- Don't add features the user didn't ask for
- Don't add explanatory text/labels to "help" the viewer understand
- Don't suggest "improvements" — fix what they asked for
- Don't change the structure unless they specifically say it's wrong
- Don't put styles inline on body elements — always use CSS classes

## Phase 4: Publish

When the user says "publish":

Call the publish operation:
```
publish {portfolioId}
{ html, css, themes, name, subdomain, portfolioId }
```

The API handles everything: creates/updates portfolio, inserts version,
auto-advances workflow (choose_layout, edit_content, publish).

Returns: `{ success: true, url: "/p/{subdomain}", portfolioId: "uuid" }`

**The themes array is critical.** Without it, the editor shows no theme pills.
The editor reads `themes` from the portfolio_versions row and renders pill buttons.

Tell the user: "Published at {url}. Edit visually at /editor/{portfolioId}."

## Phase 5: Share

After publishing, help the user share their portfolio.

1. Give them the live URL: `https://curiouscirkits.com/p/{subdomain}`
2. Suggest sharing text for different platforms:
   - **LinkedIn**: "Check out my portfolio: {url}"
   - **Twitter/X**: "Just published my portfolio! {url}"
   - **Email**: "Here's my portfolio: {url}"
3. Mark the workflow step complete:

```
workflow complete {portfolioId} share

```

This advances the workflow to 4/5. The share step is optional but completing it
gives the user a sense of progress.

## Publishing updates (re-publish)

If the user iterates AFTER publishing, re-publish on each change.
Don't ask permission. If they said "publish" once, subsequent edits auto-publish.

## Handoff to Visual Editor

After publishing, the user can edit content without Claude:

1. Go to the CuriousCirkits web app → /editor
2. Toggle to "Custom HTML" mode (sidebar)
3. Left pane shows CSS (the agent's styles)
4. Right pane is the visual preview — click any text to edit it inline
5. Click "Publish" to push changes live

The visual editor uses `contenteditable` in an iframe. Changes sync on Publish,
not on every keystroke (to avoid scroll-to-top issues).

## Image uploads

If the user provides photos (artwork, headshots, project screenshots), upload them:

```
images upload {filepath} {portfolioId}


→ { "asset": { "id": "...", "url": "/uploads/u1/uuid.jpg", "storage_key": "..." } }
```

Use the returned `url` in the HTML: `<img src="{url}" alt="...">`.
List existing images: `images list --portfolio {id}`.
Images are stored locally in dev mode, Cloudflare R2 in production.

## AI content generation

If the user wants AI-generated content from a text description:

```
ai generate
{ "mode": "text", "text": "Maya Patel, Senior Product Designer at Figma..." }
```

**Important:** This endpoint returns Server-Sent Events (SSE), not JSON.
Parse the `data:` line from the `content` event to get the structured content
(name, tagline, sections with hero/about/projects/skills/contact).

## PDF export

After publishing, the user can export their portfolio as a PDF:

```
portfolios pdf {id}
→ Binary PDF file (Content-Type: application/pdf)
```

Offer this after publishing: "Want a PDF version? I can export it."

## Known pitfalls (from real sessions)

1. **Template auto-save overwrites custom HTML.** The editor's auto-save in template
   mode creates new portfolio_versions rows that become the "latest" and overwrite
   custom HTML. Auto-save is disabled in custom mode, but switching to template mode
   and back can trigger it. Re-publish to restore.

2. **Always use the publish operation, not direct Supabase inserts.** The API handles auth,
   workflow advancement, and correct schema. Direct inserts skip workflow logic.

3. **Multiple published versions stack up.** The Worker serves the latest one
   (`ORDER BY created_at DESC LIMIT 1`). Old versions stay in the database. Fine.

4. **Don't use inline styles on body elements.** The visual editor lets users edit
   body content. If styles are inline, editing text can accidentally break layout.
   Keep all styles in the `<style>` block using CSS classes.

5. **iframe contenteditable scroll issue.** If you update `srcDoc` on the iframe
   while the user is editing, it re-renders and scrolls to top. Only sync HTML
   out of the iframe on explicit Publish or Sync, never on every keystroke.

6. **LinkedIn text is usually richer than docx** for role details. Prefer LinkedIn
   when both exist for the same role.
