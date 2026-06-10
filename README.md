# AI in Customer Success — Companion Repo

A snapshot of examples accompanying the Medium piece *"The Five Levels of AI in Customer Success"* by Topi Järvinen.

**Read the articles**


- Medium — *The Five Levels of AI in Customer Success* (the practical map this repo accompanies): `to-be released soon`
- LinkedIn — *The CSM job didn't disappear. AI helped move it closer to real customer value.* (the companion essay): `to-be released soon`

This repo is a frozen snapshot — not actively maintained — to illustrate the patterns described in the post. Adapt freely.

The examples wear CS glasses, but the underlying pattern — shared context maintained by [In Parallel](https://www.in-parallel.com), reached by your AI over MCP — is general-purpose. Project management, team management, sales, and marketing benefit just as well, and most of these CS workflows repurpose for them with little more than a change of vocabulary: swap the customer list for a project list, "needs and problems" for "risks and blockers", and the same skills run.

## What's here

| Level | What it is | Where to look |
|---|---|---|
| 1 | Chatting with AI about one meeting at a time | [`customer-success-prompts.md`](customer-success-prompts.md) |
| 2 | Your data comes to AI via MCP | [`connect-in-parallel-mcp.md`](connect-in-parallel-mcp.md) |
| 3 | Skills — reusable workflows | [`skills/customer-voice.md`](skills/customer-voice.md), [`skills/weekly-summary.md`](skills/weekly-summary.md) |
| 4 | Scheduling skills to run themselves | [`scheduling-in-claude.md`](scheduling-in-claude.md) |
| 5 | A full toolkit for technical readers | [`cs-toolkit-architecture.md`](cs-toolkit-architecture.md), [`mcp-advanced-auth.md`](mcp-advanced-auth.md), [`for-developers/`](for-developers/) |

## Two paths through the repo

Levels one through four are designed to be used **without writing code or editing configs**. The prompts and skills paste straight into Claude (web, desktop, or Cowork); the scheduling uses Claude's built-in natural-language scheduling — no cron, no YAML.

Level five is the **developer path** — for teams or individuals who want to run their own infrastructure with versioned configs, bearer tokens, scheduled jobs, and integrations beyond what built-in scheduling reaches. The `for-developers/` folder and the level-five docs cover those patterns.

## The five levels in a paragraph

If you haven't read the post: Level 1 is chatting with ChatGPT or Claude about a transcript you paste in — useful, but one conversation at a time. Level 2 connects your meeting platform, email, ticketing, and CRM via MCP so AI has its own access — no more copy-paste. Level 3 captures recurring workflows as reusable skills — prompts you save and reuse, no code. Level 4 runs those skills on a schedule, with human-in-the-loop review where it matters. Level 5 builds a real toolkit around the patterns — schemas, evaluations, multi-step pipelines, cross-team integrations — for when the friction at level four becomes structural. Most teams will land between levels three and four and stay there happily.

## What each file is

`customer-success-prompts.md` (level 1) — practical paste-and-ask prompts for common CS jobs: extracting commitments, drafting follow-ups, finding the concern beneath the request, spotting tone shifts.

`connect-in-parallel-mcp.md` (level 2) — Claude, ChatGPT, and Microsoft 365 Copilot setup for the In Parallel MCP server via the OAuth connector flow (no token or config file to edit). Based on In Parallel's [official connect guide](https://support.in-parallel.com/en/articles/691255-connect-your-ai-tool-to-in-parallel), which has the click-by-click steps with screenshots.

`skills/customer-voice.md` and `skills/weekly-summary.md` (level 3) — no-code "skills." Long prompts you can paste into a new Claude chat, save as a Claude Project's custom instructions, or run on a schedule. The customer-voice skill scans every customer's recent signal; the weekly-summary skill produces the team digest.

`scheduling-in-claude.md` (level 4) — how to use Claude's natural-language scheduling to run the skills above automatically. No cron, no YAML, just *"run this every Monday at 9am."*

`cs-toolkit-architecture.md` (level 5) — a structural overview of the production toolkit referenced in the post. The toolkit's source is private — it's dense with company-specific tweaks that wouldn't transfer — so this document outlines the approach instead, including the aspects that proved most important to get right when building something like it.

`mcp-advanced-auth.md` (level 5) — bearer tokens and OAuth for the same MCP connection at production grade — when you're running scheduled jobs across a team, building a shared toolkit, or wiring the connection into your own systems.

`for-developers/` (level 5) — for teams running their own scheduling and skill infrastructure: a YAML automations example and a description of the markdown skill-file format used by Claude Code and the cs-toolkit.

## Adapting these in your own work

For the **no-code path** (levels one through four): copy the prompts and skill bodies straight into a Claude chat, save them as a Claude Project's custom instructions, or have Claude schedule them for you. Tweak the wording for your context.

For the **developer path** (level five): drop the technical skill files into Claude Code's `.claude/commands/` directory, set up your config files, connect the MCPs the skill references, and adapt.

The architecture document is more useful as a checklist than a copy target. Most teams don't need everything; the post explains when level five is worth building, and when it isn't.

---

*Snapshot accompanying the Medium piece. Questions: topi@in-parallel.com.*
