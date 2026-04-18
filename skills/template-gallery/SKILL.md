---
name: template-gallery
description: |
  Browse and manage HTML starter templates for CuriousCirkits portfolios.
  Templates are starting points — the AI agent fills them with real content
  and generates CSS theme variants. Use when the user wants to see available
  templates, add a new template, or pick a template to start a portfolio.
  Invoked by /portfolio-builder when the user wants to "pick a template"
  instead of starting from scratch.
---

# Template Gallery

Manage the collection of HTML starter templates that kickstart portfolio creation.

## Browse Templates

```
templates list
→ { "templates": [{ "id": "minimal", "name": "Minimal", "description": "..." }, ...] }
```

Present as a numbered list:
```
TEMPLATES
═════════
1. Minimal     — Clean single-column layout with system fonts
2. Executive   — Two-track timeline + expertise tree
3. Creative    — Project-focused grid
4. Developer   — GitHub-style with tech badges
5. Academic    — Publications + research
6. Startup     — Pitch-deck style with metrics
7. Freelancer  — Services + testimonials + pricing
8. Student     — Coursework + achievements
9. Consultant  — Case studies + methodology
10. Personal   — Blog-style + interests

Pick a number, or "preview N" to see the HTML.
```

## Preview a Template

```
templates list/{id}
→ { "id": "minimal", "name": "Minimal", "description": "...", "html": "...", "css": "...", "raw": "..." }
```

The response includes:
- `html` — body content with `<style>` tags removed
- `css` — extracted CSS from `<style>` tags
- `raw` — the full self-contained HTML file

## Use a Template (called by /portfolio-builder)

When /portfolio-builder asks for a template:

1. Call `templates list/{id}` to get the HTML + CSS
2. Return the CSS and body to /portfolio-builder
3. /portfolio-builder fills the body with the user's content and generates theme variants

The template is a STARTING POINT. The agent replaces placeholder content with real content.
The CSS becomes the base for theme variant generation.

## Error Handling

If `templates list/{id}` returns 404 (template not found):

1. Tell the user that template doesn't exist
2. Call `templates list` to load the full list
3. Present the numbered list so they can pick a valid template

## Create from Template (direct)

To create a portfolio directly from a template without the full builder flow:

```
portfolios create-from-template
{ "templateId": "developer" }
→ { "portfolioId": "uuid" }
```

This creates a portfolio with the template HTML already loaded as a draft version.
The user can then edit visually at `/editor/{portfolioId}`.
