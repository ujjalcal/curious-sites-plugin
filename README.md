# CuriousCirkits

Your AI agent builds the portfolio. You just talk.

One set of photos or text. Infinite possibilities. Tell your AI agent what you want — a portfolio for grad school, a freelance showcase, a personal brand page — and it designs, builds, publishes, and manages the whole thing. You never touch HTML. You never deploy anything. You just have a conversation.

**Who this is for:**
- **Students** applying to grad school, internships, or jobs — stand out with a real portfolio
- **Freelancers and creatives** — showcase your work without learning web dev
- **Anyone** who wants a personal site but doesn't want to build one

## Install — 30 seconds

**Requirements:** [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Git](https://git-scm.com/), `curl`, `jq`

Open Claude Code and paste this. Claude does the rest.

> Install CuriousCirkits: run **`git clone https://github.com/ujjalcal/curious-sites-plugin.git ~/.claude/skills/curious-sites && cd ~/.claude/skills/curious-sites && ./setup`** then add a "CuriousCirkits" section to CLAUDE.md that lists the available skills: portfolio-orchestrator, portfolio-builder, portfolio-reviewer, portfolio-manager, template-gallery, domain-search, domain-purchase, domain-manager, billing. Note that all CLI commands use `csites` (not `cc`).

Or run it yourself:

```bash
git clone https://github.com/ujjalcal/curious-sites-plugin.git ~/.claude/skills/curious-sites && cd ~/.claude/skills/curious-sites && ./setup
```

Setup clones the repo into Claude Code's skills directory, symlinks the `csites` CLI onto your PATH, adds it to your shell profile if needed, and runs auth setup automatically. One command, fully installed.

After setup, say "build me a portfolio".

## See it work

```
You:    Build me a portfolio for my job search.

Claude: I'll set that up. What's your name?

You:    Priya Sharma. I'm a product designer with 4 years of experience.

Claude: [creates portfolio]
        [asks about your work, projects, style preferences]
        [writes HTML/CSS/JS — a full responsive portfolio]
        [publishes]
        → Live at portfolios.curiouscirkits.com/p/priya-sharma

You:    Can you review it? I want a recruiter's perspective.

Claude: [reads the published site]
        Content: 8/10 — strong project descriptions, missing quantified impact
        Visual: 7/10 — clean hierarchy, hero section needs more contrast
        Audience fit: 9/10 — good for product design recruiters
        Suggestions:
        1. Add metrics ("reduced onboarding time by 30%")
        2. Make the hero image full-bleed
        3. Add a "Currently exploring" section

You:    Make those changes.

Claude: [updates the portfolio, republishes]
        → Changes live.

You:    I want a custom domain. priyasharma.design.

Claude: [searches domain availability]
        priyasharma.design is available — $12/year
        Want me to register it?

You:    Yes.

Claude: [opens Stripe checkout → you pay]
        [registers domain → sets up DNS]
        → priyasharma.design is live.
```

Five minutes. A published portfolio with a custom domain. No code, no deploy, no config files.

## The 9 skills

Skills are markdown files that teach your AI agent how to use CuriousCirkits. The agent reads them and knows what to do.

| Skill | Your specialist | What they do |
|-------|----------------|--------------|
| `portfolio-orchestrator` | **Project Manager** | Reads all your portfolio and domain state, proposes what to do next, drives to completion. The conductor. |
| `portfolio-builder` | **Designer + Developer** | Builds sites from messy inputs — a resume, a conversation, photos, a LinkedIn URL. Multi-phase: gather info → design → build → publish. |
| `portfolio-reviewer` | **Design Critic** | Scores your portfolio on 5 dimensions (content, writing, visual, audience, technical). Offers recruiter/investor/professor perspectives. |
| `portfolio-manager` | **Site Admin** | List, rename, clone, delete portfolios. Rebind domains. Manage what you've built. |
| `template-gallery` | **Curator** | Browse 10 starter templates (Minimal, Executive, Creative, Developer, Academic, and more). |
| `domain-search` | **Domain Scout** | Search available domains with budget awareness. Recommends TLDs based on your use case. |
| `domain-purchase` | **Registrar** | Full domain registration: Stripe payment → AWS Route 53 → Cloudflare DNS → verification. |
| `domain-manager` | **DNS Admin** | Unified domain lifecycle — search, purchase, DNS setup, bind/unbind, ongoing management. |
| `billing` | **Payments** | Stripe Checkout for domain purchases. Creates sessions, verifies payment. |

## How it works

```
You talk to your AI agent
  → Agent reads the skill (learns what operations exist)
  → Agent calls the CLI (csites portfolios create, csites publish, etc.)
  → CLI sends HTTP requests to the API
  → API does the work (create, publish, DNS, payments)
  → You get a live portfolio
```

Skills teach operation names. The CLI is the transport. The API does the work. You never see the engine.

## CLI reference

```
SETUP
  csites auth setup                              Set up your account (opens browser)
  csites auth status                             Check connection

PORTFOLIOS
  csites portfolios summary                      List all portfolios
  csites portfolios create --name "Name"         Create a new portfolio
  csites portfolios content {id}                 Get published HTML/CSS
  csites portfolios get {id}                     Get portfolio details
  csites portfolios update {id} --name "Name"    Update portfolio
  csites portfolios delete {id}                  Delete a portfolio

PUBLISH
  csites publish {id}                            Publish (reads JSON payload from stdin)

IMAGES
  csites images upload {file} {id}               Upload an image
  csites images list --portfolio {id}            List uploaded images

WORKFLOW
  csites workflow get {id}                       Get workflow state
  csites workflow complete {id} {step}           Complete a workflow step

DOMAINS
  csites domains search --name "Name"            Search available domains
  csites domains list                            List domain bindings
```

## Architecture

```
Agent → Skill (informs) → CLI (transport) → API (facade) → Engine → DB
```

The plugin has three layers:

1. **Skills** (9 markdown files) — teach the agent what operations exist and how to chain them
2. **CLI** (`bin/csites`) — a bash script that maps commands to HTTP calls against the API
3. **API** (`api.curiouscirkits.com`) — the backend that does the actual work

The agent is the designer. You are the client. Everything in between is plumbing.

## Uninstall

```bash
~/.claude/skills/curious-sites/bin/uninstall
```

Removes the CLI symlink, API key, and plugin directory. Clean.

## License

MIT. Free. Go build something.
