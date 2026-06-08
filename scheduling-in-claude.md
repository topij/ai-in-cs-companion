# Automating with Claude's Built-in Scheduling — Level 4

You don't need cron, LaunchAgents, or a YAML config to put a skill on a schedule. Both Cowork and Claude Code have built-in scheduling that lets you ask Claude — in plain English — to run a prompt at a given time, every day, every Monday, hourly, or whenever you like.

This is the no-code path to level four. If you want to run your own scheduler (cron, LaunchAgents, a custom worker), see [`for-developers/automations.yaml.example`](for-developers/automations.yaml.example) — that's level five.

## How to schedule a skill

In Cowork or Claude Desktop, just ask:

```
Run the customer-voice prompt every Monday at 9am. Use the list of all my active customers and post the summary to my Slack #customer-success channel afterwards.
```

Claude sets up the scheduled run. You can confirm and review the scheduled tasks in the app — usually under Settings → Scheduled tasks.

The phrasing is natural — anything you'd say to a colleague works:

- *"Every weekday at 8:30am, do a quick scan of overnight emails and Slack mentions of customers, and surface anything that looks urgent."*
- *"Every Friday at 4pm, run the weekly customer summary and post it as a draft to #customer-success."*
- *"In an hour, remind me to send the follow-up email I drafted to Acme."*
- *"On the first of every month, run a 'customers gone quiet' check across the book and flag anyone with no signal in the last 30 days."*

## Before you rely on it — operational notes

Built-in scheduling is genuinely no-code, but a few things will surprise you the
first time. None are bugs; knowing them up front saves a confused morning.

- **The app has to be running.** Scheduled tasks fire while the app is open. If the
  app is closed when a task is due, it runs on next launch instead of being skipped.
  So a "missed" 8:30am run usually means the app was closed, not that the task broke.

- **Run it once manually to pre-approve its tools.** The first run pauses to ask for
  permission on each connector it touches (your meeting platform, Gmail, Slack…). An
  unattended 8:30am run can't answer that prompt. Hit "Run now" once, approve the
  tools, and future runs go through without stalling.

- **The exact minute drifts a little.** An 8:30 schedule may show and fire around
  08:37. This jitter is deliberate (it spreads load); don't set tight back-to-back
  schedules expecting to-the-second timing.

- **For Slack (or any channel target), use the real channel — and its ID.** The
  channel name in an example prompt is just an example; your workspace may not have a
  `#customer-success` channel at all. Point the task at whatever your CS channel is
  actually called, and give the scheduled prompt the channel's *ID*, not just its
  name — a fresh scheduled run has no chat history to disambiguate a name from.

- **Drafts beat sends for anything outbound.** For a scheduled task that posts to
  Slack or email, prefer "create a draft" over "send" so a human still eyes it. See
  Human-in-the-loop below.

## A few CS workflows worth scheduling

| Schedule | Workflow | Notes |
|---|---|---|
| Weekdays, 8:30am | A morning brief — what's new since yesterday in customer emails, Slack, and meeting summaries | Read before your first meeting |
| Monday, 9am | The weekly customer-voice scan (see [`skills/customer-voice.md`](skills/customer-voice.md)) | Output ready before the Monday CS standup |
| Friday, 4pm | The weekly customer summary (see [`skills/weekly-summary.md`](skills/weekly-summary.md)) | Ready for the team to read first thing Monday |
| 1st of month, 9am | "Silent customer" check — flag accounts with no recent signal | Catch churn risks early |
| Thursday, 1pm | Quality review — re-read this week's customer-voice output for inference language, unverified problems, contradictions across customers | Catch hallucinations before they ship downstream |

## Human-in-the-loop

You don't always want scheduled tasks to act autonomously. For things like sending an email, posting to Slack, or escalating to a colleague, the right pattern is *draft, then wait for approval*.

You can build this into the scheduled prompt:

```
Every Friday at 10am, draft a release-notes summary from this week's shipped PRs. Send the draft to me in chat and wait for my reply before posting anywhere. If I say "go", post it to the team channel. If I reply with edits, incorporate them and ask again. If I haven't replied by 2pm, hold the draft and remind me at 9am Monday.
```

The "wait for my reply" pattern is the no-code version of the level-five Slack review loop described in [`cs-toolkit-architecture.md`](cs-toolkit-architecture.md).

## Cancelling, editing, or pausing scheduled tasks

In Cowork, scheduled tasks are listed in the app. You can pause one (*"don't run the Monday customer-voice scan next week, I'm on holiday"*) or edit it (*"change the morning brief from 8:30 to 9:00"*) just by asking.

If you change your mind about a workflow entirely, ask Claude to cancel it the same way.

## When you'd want your own scheduler

Built-in scheduling covers most teams' needs. You'd graduate to something heavier when:

- You need pipelines with branches and complex error handling — retries, fall-back paths, alerting paths.
- You need scheduled jobs to run when your laptop is off, so they need to live on a server or cloud function.
- You need the schedule itself versioned and reviewable in a repository — for a team where the schedule is shared infrastructure.
- You need integrations Claude's scheduling doesn't reach yet.

That's level five. See [`for-developers/automations.yaml.example`](for-developers/automations.yaml.example) for a YAML-config approach, or the cron and LaunchAgents pattern described in [`cs-toolkit-architecture.md`](cs-toolkit-architecture.md).
