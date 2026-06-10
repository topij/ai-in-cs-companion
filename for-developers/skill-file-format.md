# Skill File Format for Claude Code and cs-toolkit

For teams running their own setup, a "skill" is a markdown file Claude reads at the start of a session and follows as instructions. This is the format used by Claude Code and the cs-toolkit.

This is the level-five pattern. For the no-code version of the same workflows — copy-pasteable prompts you can run in Claude or Cowork without any files or configuration — see [`../skills/`](../skills/) instead.

## Where the files live

In a Claude Code project, skill files go in `.claude/commands/`. Each file becomes a slash command — `customer-voice.md` becomes `/customer-voice`.

A project might look like this:

```
your-project/
├── .claude/
│   └── commands/
│       ├── customer-voice.md
│       ├── weekly-slides.md
│       └── ...
├── config/
│   ├── customers.yaml
│   ├── sources.yaml
│   └── ...
└── ...
```

## What a skill file looks like

A skill file is plain markdown. Claude reads it and follows the steps. The file usually contains:

- A title with the slash-command name (e.g. `# /customer-voice`)
- A short description of what the skill does
- Parameters the user can pass in (since, customer, mode, etc.)
- Prerequisites — required configs and MCP servers
- What it does — numbered steps in plain English
- Quality rules — invariants the skill must respect

Example skeleton:

```markdown
# /customer-voice

Pull customer signal from meetings, email, and tickets — produce per-customer reports and an aggregated rollup.

## Configuration

This skill reads `config/customers.yaml` for the list of active customers and `config/sources.yaml` for which MCP servers provide each data source.

Required MCPs:
- In Parallel — for meeting summaries and signals
- Gmail — for customer email threads
- Linear — for ticket activity

## Parameters

- `customer:` (optional) — run for a single customer only. Default: all active.
- `since:` (optional) — how far back to look. Default: 7 days.

## What it does

For each active customer in `config/customers.yaml`:

1. Pull meetings from the last 7 days via the In Parallel MCP.
2. Pull email threads via the Gmail MCP, filtered to the customer's known contacts.
3. ...

## Quality rules

- Every observation needs a source pointer. No untraceable claims.
- Problems need a verbatim customer quote. Inferred problems are flagged as unverified.
- ...
```

The cs-toolkit's real skill files follow this format with much more detail — from a hundred lines to several hundred for the big pipelines, with explicit references to schema files, output file paths, and helper Python modules in `libs/`.

Three production-proven patterns worth copying once your skills run unattended:

- **An autonomy banner.** A skill run by cron via `claude -p` must say so explicitly at the top: *"Proceed autonomously — do not ask for confirmation. Stop only if [the two or three genuinely fatal conditions]."* An unattended run that pauses to ask a question exits silently with no work done, and the scheduler records a false success.
- **Orchestrator + isolated subagents.** The production customer-voice skill is an orchestrator that runs one subagent per customer, each seeing exactly one customer's data. Cross-customer contamination — customer A's quote landing in customer B's report — becomes structurally impossible rather than something you hope the model avoids.
- **Phase splits for crash isolation.** Long pipelines are splittable by a `mode:` parameter (e.g. `gather` writes per-customer files, a separate `rollup` run reads whatever landed on disk). A crash mid-gather then costs one customer's report, not the whole run.

## Why this format

The trade-off vs. the no-code prompts is real:

- **No-code prompts** are easier to share and adapt, but each run can produce subtly different output, and the contract between skills is informal (Claude reads what the previous skill produced and figures it out).
- **Skill files** are more code-like — versioned in git, reviewed by colleagues, with explicit configs and file paths. The contract between skills is formal: skill A writes to `reports/customer-voice_{id}_latest.json`, skill B reads from that exact path.

For a personal weekly workflow, the no-code prompts are usually enough. For a team-shared toolkit with quality gates, scheduled runs, and downstream automation, the skill-file format earns the extra setup cost.

## See also

- [`automations.yaml.example`](automations.yaml.example) — scheduling skill files with your own cron, LaunchAgents, or other infrastructure.
- [`../cs-toolkit-architecture.md`](../cs-toolkit-architecture.md) — the broader architecture pattern these skill files fit into.
- [`../mcp-advanced-auth.md`](../mcp-advanced-auth.md) — bearer tokens and OAuth for production MCP connections.
