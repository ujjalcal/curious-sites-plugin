# CuriousCirkits Portfolio Plugin

One set of photos or text. Infinite possibilities. Your AI agent builds the portfolio. You just talk.

## Install

```bash
git clone https://github.com/ujjalcal/curious-sites-plugin.git ~/.claude/skills/curious-sites && cd ~/.claude/skills/curious-sites && ./setup
```

That's it. Open Claude Code and say "build me a portfolio".

## What you get

9 skills that teach any AI agent to build, publish, and manage portfolio websites:

| Skill | What it does |
|-------|-------------|
| portfolio-orchestrator | Reads state, proposes action plans, drives to completion |
| portfolio-builder | Builds sites from messy inputs (resume, sketch, photos, conversation) |
| portfolio-reviewer | Reviews quality, gives specific feedback |
| portfolio-manager | List, rename, clone, delete portfolios |
| template-gallery | Browse starter templates |
| domain-search | Find available domains with pricing |
| domain-purchase | Register + DNS setup |
| domain-manager | Unified domain lifecycle |
| billing | Stripe checkout for domain payments |

## How it works

```
You: "build me a portfolio"

Agent reads the skill → learns operation names
Agent calls: cc portfolios create --name "Your Name"
Agent writes HTML/CSS/JS → your portfolio
Agent calls: cc publish {id}
→ live at portfolios.curiouscirkits.com/p/your-name
```

The agent is the designer. The CLI is the transport. The API does the work.

## CLI operations

```
cc auth setup                              Set up your account (opens browser)
cc auth status                             Check connection
cc portfolios summary                      List all portfolios
cc portfolios create --name "Name"         Create a new portfolio
cc portfolios content {id}                 Get published HTML/CSS
cc portfolios get {id}                     Get portfolio details
cc portfolios update {id} --name "Name"    Update portfolio
cc portfolios delete {id}                  Delete a portfolio
cc publish {id}                            Publish (reads JSON payload from stdin)
cc images upload {file} {id}               Upload an image
cc images list --portfolio {id}            List uploaded images
cc workflow get {id}                       Get workflow state
cc workflow complete {id} {step}           Complete a workflow step
cc domains search --name "Name"            Search available domains
cc domains list                            List domain bindings
```

## Architecture

```
Agent → Skill (informs) → CLI (transport) → API (facade) → Engine → DB
```

Skills teach operation names. The CLI maps them to HTTP. The API does the work. You never see the engine.

## License

MIT
