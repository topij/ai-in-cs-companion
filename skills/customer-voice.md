# Customer Voice — No-Code Skill

A weekly "scan every customer" workflow for Claude. No config files, no YAML, no developer setup — just connect In Parallel via MCP (see [`../connect-in-parallel-mcp.md`](../connect-in-parallel-mcp.md)) and use the prompt below.

## How to use this

Three ways, depending on how often you'll run it:

- **One-off.** Paste the prompt below into a new Claude chat. Tell Claude which customers to look at.
- **Repeated.** Save the prompt as the custom instructions on a [Claude Project](https://claude.ai/projects). Start a fresh chat in that project each week.
- **Scheduled.** Ask Claude to run it every Monday at 9am — see [`../scheduling-in-claude.md`](../scheduling-in-claude.md).

## The prompt

```
You're going to help me do a weekly review of customer signal across my book of business.

For each customer I name, please:

1. Pull every meeting from the last 7 days from In Parallel. Look at the structured summary and the signals — decisions, action items, risks, anything raised as a concern.

2. If I have email or ticketing tools connected via MCP, pull the customer's recent email threads and ticket activity from those sources too.

3. Extract observations of these types:
   - Need — something they want that we don't yet provide
   - Problem — something they're frustrated with
   - Praise — something working well that's worth remembering
   - Decision — something we (or they) committed to

   For each observation, give me: a one-line description, a verbatim quote from the source (or note that there isn't one), the source it came from (which meeting, email, or ticket), and the date. If a problem doesn't have a verbatim customer quote, flag it as "unverified" rather than dropping it.

4. Reconcile across sources: if the same point came up in multiple places, group those observations. If something flagged as a problem in an older source now looks resolved (a feature shipped, a ticket closed, the customer mentioned it was fixed), mark it resolved with the evidence.

5. Give me a per-customer summary in this shape:
   - Customer name (use the display name, not an internal ID)
   - One or two sentences on the state of the account
   - Needs — top 3 to 5
   - Problems — top 3 to 5, with verified-or-not flag
   - Notable praise — 1 or 2 items
   - Closed since last week — recent resolutions

6. After all customers, give me an aggregated view: themes that came up across multiple customers, features that just shipped which help specific customers, anything that looks like an emerging risk across the book.

Keep descriptions specific and concrete — "delays on the capture-toggle in mobile" not "issues persist". Don't fabricate; if a customer has no signal in the period, say so.

Which customers should I look at this week?
```

## Tips

- For the customer list, you can paste names, paste a list from a spreadsheet, or just say *"all the customers I had meetings with last week"* — Claude will figure it out from your In Parallel data.

- This isn't instant for many customers. If you have ten or more on your book, run it on a schedule overnight or early morning so the digest is waiting when you sit down. See [`../scheduling-in-claude.md`](../scheduling-in-claude.md).

- The output is a chat — you can ask follow-ups: *"tell me more about Customer X's third problem"*, or *"draft the renewal-conversation talking points for Customer Y"*, or *"which of these warrant escalation this week?"*

## When you want more

If you want the output saved in a specific file format, persisted across sessions, validated against a strict schema, or feeding downstream systems automatically, that's level five — see [`../cs-toolkit-architecture.md`](../cs-toolkit-architecture.md) and [`../for-developers/skill-file-format.md`](../for-developers/skill-file-format.md).
