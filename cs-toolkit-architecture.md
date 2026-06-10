# cs-toolkit — Architectural Overview

A simplified structural overview of cs-toolkit, the production Customer Success toolkit referenced in the Medium post's Level 5 section.

The source is private — not out of secrecy about the approach, but because the repo is dense with company-specific tweaks (our customers, our channels, our naming, our org's review habits) that would be noise to anyone else. What transfers is the approach. This document describes the shape, and — more usefully — the aspects that proved important to get right, so you can build something similar for your own context.

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
│   ├── in-parallel-utils/   # In Parallel MCP client (direct bearer-auth, for scheduled runs)
│   ├── linear-client/       # Direct Linear GraphQL client (for scheduled runs)
│   └── cs-server/           # MCP + REST server exposing toolkit content to colleagues
├── packs/                   # Pack content — read by the substrate at runtime
│   └── voice/               # Customer-voice pack: schemas, prompts, taxonomy
├── domains/                 # Workspaces
│   ├── customer-voice/      # Insights from meetings, email, tickets
│   ├── customer-comms/      # Release notes, shipped-impact, email distribution
│   └── support-docs/        # KB articles, guides, screenshots, Intercom sync
├── scripts/                 # Orchestration (snapshot, automations, recovery)
├── state/                   # Shared cache and history (gitignored where it carries PII)
├── .claude/commands/        # Skills (cross-domain + per-domain, ~100 total)
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

**Observations and enrichment.** In Parallel does the heavy lifting: it maintains shared context across every meeting and makes detailed, intelligent observations grounded in that context — surfaced as nine signal types (decisions, action items, risks, dependencies, escalations, opportunities, learnings, progress updates, obstacles), with action items attributed to the people who own them. The customer-voice domain doesn't re-derive that intelligence. It *enriches* it with a CS-specific lens: re-categorising into the types CS cares about (need, problem, praise, decision), joining with email and ticket data, backing each item with a verbatim quote, and adding its own observations where the CS lens catches something a general-purpose pass wasn't looking for — drawn both from In Parallel's observations and from the raw transcripts. The division of labour: In Parallel observes; the toolkit specialises.

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

**MCP, via In Parallel and others.** Meetings and signals come through the In Parallel MCP. Gmail, Slack, Linear, and the CRM each have their own MCP servers for interactive use. The scheduled snapshot pipeline uses direct API calls instead, since unattended runs are better off without an interactive session in the loop.

**cs-server — internal access for colleagues.** The toolkit exposes its own data through a `cs-server` with an MCP interface and a REST API. This is how the customer signal stops being one operator's private output: a colleague in product, sales, or engineering can query the same reports, observations, and health views from their own AI tools — without going through the person who runs the toolkit. It's the bridge from a personal toolkit to genuinely shared context.

**Slack review loops.** Skills that produce material humans must review (release notes, customer health summaries, weekly QA verdicts) post to a review channel and wait. A separate Slack bot watches reactions and either advances the pipeline (👍) or sends the draft back for revision (👎).

**Google Workspace.** Drive for report sync, Sheets for the dashboard, Gmail for outbound customer email. Unattended runs authenticate with a service account; interactive skills use the user's own session.

**Output channels.** Slack (`#product`, `#customer-success`), a Google Drive folder, an office kiosk display, customer email via Postmark, Intercom Messenger News for release notes.

## Automation

About forty scheduled jobs run on a recurring schedule — wherever the toolkit is hosted, whether that's a developer machine or an internal server. The schedule has dependency-aware waits — downstream jobs poll for upstream completion instead of silently skipping when the upstream runs a little late.

Machine-generated work is kept separate from human-approved work: scheduled runs write to a staging area that a human reviews and promotes on a regular cadence. This separates *machine wrote this* from *I read it and approved it*, and gives the toolkit a real reviewable history.

A regular drift-checker compares the auto-generated work to the approved baseline and surfaces anything that needs attention.

## What proved important to get right

The structure above is the easy part. These are the lessons that took months of production runs to learn — the ones worth stealing even if your architecture looks nothing like this.

**Never let the model keep its own books.** Early on, the per-customer reports carried LLM-written counts ("12 threads reviewed, 8 observations extracted"). They were wrong in both directions — sometimes inflated, sometimes zero while the same run demonstrably wrote a dozen new observations. Every count, status summary, and "what changed this run" field is now recomputed deterministically from the structured data at write time, overwriting whatever the model claimed. The model produces content; code produces bookkeeping.

**Enforce deterministically what the LLM applies inconsistently.** Skill prompts contain rules ("demote an open problem when the linked ticket is done"). On scheduled, unattended runs, the model skips some of them — reliably enough that you can measure it. Any rule that doesn't need the run's raw conversational context is re-applied by a plain Python script after the skill finishes, and a schema verifier fails the run if the rule visibly wasn't applied. Prompt rules are a request; post-run scripts are a guarantee.

**Stamp provenance on every artifact.** Each published report carries a run manifest: which code version wrote it, from which data snapshot, when. Paired with a small "deploy status" tool, this turns the most common operational question — *was the fix live for the run that produced this artifact?* — into a query instead of an afternoon of forensics across git logs and cron timestamps.

**Build decay in.** An observation nobody reinforces should age out on its own — first downgraded to context, eventually archived — with the reason stamped on it. Without this, the system slowly fills with stale findings that read as current, and trust erodes precisely because nothing ever looks wrong.

**Weight evidence by who said it.** A first-person statement from an active user is not the same signal as an exec sponsor's secondhand summary, or an internal teammate's impression. Every observation carries an evidence class derived from who spoke, and only verifiable first-person customer voice qualifies for the cross-customer surfaces that product and leadership read. Per-customer reports keep everything for audit.

**Recovery beats rerun.** When a scheduled run dies mid-write, the instinct is to rerun it — but a full LLM cycle is the most expensive operation in the system. Most failures have cheaper shapes: patch a stale report deterministically from the source of truth, or replay the file-writes from the dead session's log. Having these procedures written down before the first 7am failure is the difference between a three-minute fix and a lost morning.

**Separate machine-written from human-approved.** Scheduled runs commit to a staging branch that a human reviews and merges on a regular cadence; guards on both the writer and a drift-detector keep the boundary honest. The audit trail "the machine wrote this, I approved it" turns out to matter as much as the content.

## What level five is for

Levels one through four (in the companion Medium post) cover the patterns most teams need: chat with AI, connect MCPs, write skills, schedule them. Level five — a toolkit like this one — earns itself when:

- You need schemas and contracts. Downstream tools consume structured data, not parsed prose.
- You need to evaluate AI output systematically. An eval harness scores faithfulness, categorisation accuracy, quote attribution, and quote tiering.
- You have multi-step pipelines with branching that scheduled tasks can't express cleanly.
- You're connecting AI work into other teams' systems (product, comms, engineering) in ways that need auditability.
- You're running this often enough that "it broke" needs a recovery story, not a manual restart.

If none of those apply, level four is the right destination. Level five is a year of compounding work; it earns itself when the friction of operating at level four becomes structural.
