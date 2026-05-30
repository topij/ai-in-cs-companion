# Weekly Customer Summary — No-Code Skill

A standalone weekly summary for the team. Pairs naturally with [`customer-voice.md`](customer-voice.md), but works on its own too.

## How to use this

- **As a follow-up to customer-voice.** After running the customer-voice scan in a Claude chat, paste the prompt below as the next message — Claude will summarise the per-customer reports it just produced.
- **As a one-off.** Paste it into a new chat after you've connected In Parallel. Tell Claude which customers to summarise.
- **As a Claude Project.** Save the prompt as the project's custom instructions for repeatable weekly use.
- **Scheduled.** Ask Claude to run it every Friday afternoon — see [`../scheduling-in-claude.md`](../scheduling-in-claude.md).

## The prompt

```
Produce a weekly customer summary for the team. The audience is everyone in product and customer success.

Use what you know from In Parallel (and any other MCPs I have connected) to cover the following sections:

1. Top needs across all customers. The 4 needs raised most across the book this week. For each one, name the customers raising it and the one-line specifics they raised. Use display names, not IDs.

2. Top problems. The 3 most significant problems this week, with severity (blocker / serious / nagging). For each, name the customer or customers, give the verbatim quote that captured the pain, and note whether it's verified or unverified.

3. Emerging themes. Up to 4 patterns showing up across multiple customers — things that wouldn't catch the eye in a single account but are visible from the book-wide view.

4. Customer state. One row per active customer with a status word (green / amber / red) and one specific line about why. Be specific — "capture-toggle delay still blocking onboarding" beats "issues persist".

5. Solved this week. Problems that closed in the last 10 days, grouped by customer. Note which were closed by us shipping something versus customer-side resolution.

Keep descriptions concrete and specific — strip implementation detail and internal jargon. Title-case headings. Customer display names everywhere.
```

## Tips

- This is the prompt you'd run on Friday afternoon to have the team's Monday briefing ready. If you connect Slack via MCP, you can extend the prompt: *"…and post the result as a draft message to #customer-success."*

- The summary is intentionally derived from the customer-voice scan, not regenerated from raw transcripts each time. If you want fresh signal, run customer-voice first.

- The output is a chat — ask Claude to expand any section: *"unpack the Acme line in customer state — what's the actual blocker?"*

## When you want more

For a styled PowerPoint or PDF version of this summary, that's level-five territory — the cs-toolkit has a `/weekly-slides` skill that produces a navy-and-teal deck for the Monday meeting. See [`../cs-toolkit-architecture.md`](../cs-toolkit-architecture.md).
