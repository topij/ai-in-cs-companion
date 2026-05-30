# cs-toolkit — Architectural Overview

A simplified structural overview of cs-toolkit, the production Customer Success toolkit referenced in the Medium post's Level 5 section. The source is private; this document describes the shape so others can build something similar.

## Purpose

cs-toolkit reads customer signal from many sources — meetings, email, support tickets, chat, product telemetry, CRM — and produces:

- Per-customer reports (what each customer is dealing with this week)
- A cross-customer rollup (top needs, top problems, emerging themes)
- A persistent topic graph (how themes evolve across customers and quarters)
- A health view per customer (combining voice signal with product telemetry)
- A shipped-impact mapping (which customer's open problem did this week's release actually close?)
- Slack notifications, weekly slide decks, Google Drive sync, an office kiosk display
- Release notes drafted from shipped PRs, reviewed by humans in Slack, sent to customers

All driven by Claude skills, with deterministic quality checks layered on top.

## Repository structure

```
cs-toolkit/
├── libs/                    # Shared Python libraries, each independent
│   ├── substrate-core/      # Identity, observation, pack-loading primitives
│   ├── pipeline-engine/     # State, resume, review loops, config
│   ├── report-utils/        # Schemas, topic matching, markdown assembly
│   ├── eval-harness/        # Semantic quality metrics
│   ├── slack-utils/         # Markdown conversion, dedup, reactions, thread polling
│   ├── slack-bot/           # Always-on bot that answers questions in Slack
│   ├── google-utils/        # Drive + Sheets + Gmail clients
│   ├── credentials/         # Single dotenv loader with format validation
│   ├── intercom-utils/      # Intercom API client
│   ├── postmark-utils/      # Email engagement stats
│   ├── in-parallel-utils/   # In Parallel MCP client
│   └── cs-server/           # MCP + REST server exposing toolkit content to colleagues
├── packs/                   # Pack content — read by the substrate at runtime
│   └── voice/               # Customer-voice pack: schemas, prompts, taxonomy
├── domains/                 # Workspaces
│   ├── customer-voice/      # Insights from meetings, email, tickets
│   ├── customer-comms/      # Release notes, shipped-impact, email distribution
│   └── support-docs/        # KB articles, guides, screenshots, Intercom sync
├── scripts/                 # Orchestration (snapshot, automations, recovery)
├── state/                   # Shared cache and history (gitignored where it carries PII)
├── .claude/commands/        # Cross-domain skills (~50 of them)
└── config/                  # sources.yaml, customers.yaml, automations.yaml, …
```

The pattern: shared utilities in `libs/`, schemas and prompts in `packs/`, workspaces in `domains/`, and skills as markdown files in `.claude/commands/`. Each library is an independent Python package with its own lockfile and tests.

## The data flow

```
┌─────────────────────────────────────────────────────────────┐
│ External sources                                            │
│ In Parallel  Gmail  Slack  Linear  Intercom  CRM  Grafana   │
└────────────────────────────┬────────────────────────────────┘
                             │
                ┌────────────▼─────────────┐
                │ Snapshot pipeline (cron) │
                │ Daily, before work hours │
                └────────────┬─────────────┘
                             │
                  ┌──────────▼──────────┐
                  │   state/cache/      │
                  │   24h TTL           │
                  └──────────┬──────────┘
                             │
                  ┌──────────▼──────────┐
                  │ /customer-voice     │
                  │ Extract + reconcile │
                  └──────────┬──────────┘
                             │
              ┌──────────────▼──────────────┐
              │ Per-customer reports        │
              │ + cross-customer rollup     │
              │ (markdown + JSON)           │
              └──────────────┬──────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼────┐         ┌─────▼─────┐        ┌─────▼─────┐
   │ Weekly  │         │ Customer  │        │ Shipped   │
   │ slides  │         │ health    │        │ impact    │
   └─────────┘         └───────────┘        └───────────┘
```

Single fan-in (the snapshot), single shared cache, many downstream skills consuming the same shaped output. Skills don't re-fetch from the external sources during the workday — they read from the cache, which makes runs cheap, fast, and replayable.

## Key concepts

**Signals vs observations.** In Parallel produces *signals* — nine structured types per meeting (decisions, action items, risks, dependencies, escalations, opportunities, learnings, progress updates, obstacles). The customer-voice domain reads those signals, joins them with email and ticket data, and derives *observations* — categorised by type (need, problem, praise, decision), each backed by a verbatim quote. Signals are In Parallel's vocabulary; observations are the toolkit's customer-voice layer built on top.

**The Parallel Context Graph.** The structured record In Parallel maintains from every meeting your team runs. Exposed to the toolkit via In Parallel's MCP server. The toolkit reads from the graph; it doesn't try to recreate or duplicate it.

**The shared cache.** The snapshot pipeline writes one cache file per source (one for daily-drops, one for PRs, one for Linear tickets, …) with a 24-hour TTL. Every downstream skill reads from the cache instead of re-fetching from the external API. This is what makes the rest of the day cheap and idempotent.

**Skills as markdown.** Every workflow is a markdown file in `.claude/commands/`. Claude reads the file and follows the steps. No Python orchestration glue — the orchestration *is* the markdown. When a step needs determinism (topic matching, report rendering, quote tier scoring), the skill calls a small Python helper from `libs/`.

**Pack content vs library code.** A *pack* (in `packs/`) holds schemas, prompts, and taxonomies that the substrate reads at runtime — never imported as Python. A *library* (in `libs/`) holds reusable Python helpers. The separation lets prompts evolve independently of the orchestration code.

## Quality assurance

Three layers, each doing different work:

1. **Prevention.** The customer-voice skill enforces a verbatim-quote requirement on every problem observation. The skill prompt explicitly checks the learning log of known misclassifications before extracting anything new.

2. **Detection.** A deterministic Python checker runs over every customer-voice run and flags:
   - Unverified problems (no verbatim quote)
   - Inference language ("the underlying issue seems to be…")
   - Source conflicts (the same source produced a resolved item and an open item)
   - Cross-report topic contradictions (one customer's resolved is another customer's still-active)
   - Stale topics with no new observations in 30+ days

3. **Correction.** A `/qa-fix` skill consumes the integrity report and applies corrections — annotating unverified items, rewriting inference language, marking conflicts for human review.

The principle: don't trust the AI's prose. Validate the structured data with rules. Reserve human attention for the things the rules can't decide.

## Integrations

**MCP, via In Parallel and others.** Meetings and signals come through the In Parallel MCP. Gmail, Slack, Linear, and the CRM each have their own MCP servers for interactive use. The snapshot pipeline uses direct API calls instead, because cron-side work is better off without an interactive session in the loop.

**Slack review loops.** Skills that produce material humans must review (release notes, customer health summaries, weekly QA verdicts) post to a review channel and wait. A separate Slack bot watches reactions and either advances the pipeline (👍) or sends the draft back for revision (👎).

**Google Workspace.** Drive for report sync, Sheets for the dashboard, Gmail for outbound customer email. Cron paths use a service account with domain-wide delegation; interactive skills use the user's session.

**Output channels.** Slack (`#product`, `#customer-success`), a Google Drive folder, an office kiosk display, customer email via Postmark, Intercom Messenger News for release notes.

## Automation

About forty scheduled jobs run via macOS LaunchAgents on a laptop kept deliberately powered on. The schedule has dependency-aware waits — downstream jobs poll for upstream completion instead of silently skipping when the upstream runs a little late.

The cron jobs commit to a dedicated `auto/daily` branch. Once a week, a Friday job opens a pull request merging that branch into `main`, which gets reviewed and merged by a human. This separates *machine wrote this* from *I read it and approved it*, and gives the toolkit a real reviewable history.

A daily drift-checker compares the auto-committed work to the main branch and surfaces anything that needs attention.

## What level five is for

Levels one through four (in the companion Medium post) cover the patterns most teams need: chat with AI, connect MCPs, write skills, schedule them. Level five — a toolkit like this one — earns itself when:

- You need schemas and contracts. Downstream tools consume structured data, not parsed prose.
- You need to evaluate AI output systematically. An eval harness scores faithfulness, categorisation accuracy, source attribution coverage.
- You have multi-step pipelines with branching that scheduled tasks can't express cleanly.
- You're connecting AI work into other teams' systems (product, comms, engineering) in ways that need auditability.
- You're running this often enough that "it broke" needs a recovery story, not a manual restart.

If none of those apply, level four is the right destination. Level five is a year of compounding work; it earns itself when the friction of operating at level four becomes structural.
