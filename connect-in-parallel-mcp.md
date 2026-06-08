# Connecting to the In Parallel MCP — Level 2

The single step from level one to level two: give your AI its own access to your data
instead of copy-pasting it. In Parallel exposes its meetings, goals, decisions, action
items, and Workspaces — plus a few actions — over the **Model Context Protocol (MCP)**,
the open standard AI clients use to talk to external tools. You connect once, authorize
in an **OAuth** consent screen, and from then on your AI assistant can read and act on
your In Parallel data directly from a chat.

> **Authoritative source.** This file is the short version. In Parallel's own help
> article has the full click-by-click walkthrough with screenshots, and it's kept
> current as the client UIs change:
> **<https://support.in-parallel.com/en/articles/691255-connect-your-ai-tool-to-in-parallel>**
> If anything below has drifted from what you see on screen, trust the support article.

## What you need

- An **In Parallel account**.
- A supported AI client: **Claude**, **ChatGPT**, or **Cursor**. (GitHub Copilot is coming next.)
- The connector URL: **`https://www.in-parallel.ai/mcp`** — you point your client at this address.

There's no JSON config to edit and no token to paste for the standard setup. The
connection is OAuth: you click **Connect**, choose which Workspaces to share, and click
**Allow access**. (If you specifically need an API key instead — for headless or
scheduled use — see "Prefer an API key?" below.)

## Claude (desktop)

1. Open your account menu → **Settings**.
2. Open **Connectors** in the left nav. Claude now keeps these under **Customize** — click through to **Customize → Connectors**.
3. Find **In Parallel** under **Web** connectors and select it. If it isn't listed, use the **+** add icon and enter the MCP URL `https://www.in-parallel.ai/mcp`.
4. Click **Connect**. In Parallel opens its connect page; click **Connect** there to start authorization.
5. On the **OAuth consent** screen, choose what to share: **Full access** (all current *and future* Workspaces) or tick individual Workspaces. Click **Allow access** — you're redirected back to Claude.
6. Review **Tool permissions**. Write and delete tools default to **Needs approval**; you can set any tool to **Needs approval**, **Blocked**, or **Custom**.
7. In any chat, open the **+** menu → **Connectors** and toggle **In Parallel** on for that conversation.

**Verify it's working.** In a fresh chat: *"What In Parallel tools do you have available?"* If you see a tool list, you're connected.

## ChatGPT

In ChatGPT, In Parallel is a **custom app (MCP server)**, which lives under **Developer Mode** (currently beta).

1. Account menu → **Settings** → **Apps**.
2. Under **Advanced settings**, click **Create app**. Fill in: **Name** *In Parallel*, **MCP Server URL** `https://www.in-parallel.ai/mcp`, **Authentication** **OAuth**. Acknowledge the custom-MCP risk warning and click **Create**.
3. Complete the same In Parallel **OAuth consent** (choose Workspaces → **Allow access**).
4. In the composer, open **+** → **More** → **In Parallel** to enable it. The chat runs in **Developer mode** (memory is off for that chat).

## Cursor

Add In Parallel as an MCP server in Cursor's settings using the same URL,
`https://www.in-parallel.ai/mcp`, and complete the OAuth consent step (choose
Workspaces → **Allow access**). Cursor's MCP config location moves around as the client
evolves — follow Cursor's own MCP setup docs for where the server config lives.

## Other Claude products

- **Cowork** — add In Parallel from the connector/integrations UI using the same URL and the same OAuth consent flow.
- **Claude Code** — add the remote MCP server pointing at `https://www.in-parallel.ai/mcp` and complete OAuth on first use.

## Prefer an API key?

OAuth is the default and the easiest path, and it's what most people should use. But if
OAuth can't complete — **headless agents, CI runners, scheduled jobs on a server, or
otherwise restricted environments** — In Parallel can issue an **API key** (a bearer
token) instead. You don't self-generate it: **contact your In Parallel customer success
manager to request an API key for your AI client.** Once you have it, it's sent as an
`Authorization: Bearer <token>` header; the production patterns for storing and rotating
it are in [`mcp-advanced-auth.md`](mcp-advanced-auth.md). Treat it like a password —
never paste it into a chat, never commit it.

## A few starter prompts

Once connected, try these to confirm everything's wired up and to feel the level-two difference:

```
What meetings did I have with [customer name] last week? Summarise each one in a sentence.
```

```
Across the last month of meetings, what topics came up most often with [customer name]?
```

```
Find every meeting in the last two weeks where someone raised a concern about [topic]. Quote the moment it came up.
```

## What it can do once connected

Read your meetings, goals, decisions, action items, and Workspaces — and run actions
like creating an action item or running the `get_drift_report` execution-health check.
You stay in control: per-tool permissions require approval before any write or delete
tool runs, or block specific tools entirely. See In Parallel's
[Integrations overview](https://support.in-parallel.com/en/articles/518624-integrations)
for everything else it connects to.

## If it doesn't work

- **The OAuth step won't complete while you're adding it.** This is a *connect-time* failure — distinct from the ones below, which assume an already-connected server. If the consent screen errors, the sign-in won't finish, or you never get redirected back to your client, check: the URL is exactly `https://www.in-parallel.ai/mcp` (no typo or trailing path); a pop-up blocker or browser isn't interrupting the redirect (try again, or a different browser); and the account you're authorizing with has access to the Workspaces you expect. If it persists, it's likely server-side rather than your config — check the support article or ask In Parallel support. (For environments where OAuth simply can't run, use the API-key path above.)
- **Claude doesn't see the In Parallel tools.** Make sure the connector is toggled on in the chat (**+** → **Connectors**). A full restart of the client can help if it was just connected.
- **Empty results.** Confirm you granted the consent screen access to the right Workspaces — and remember a meeting is only visible once it's been **published to a Workspace** you shared. A freshly-recorded meeting in an unshared or goal-less Workspace won't show up until it's published.
- **Anything else.** The support article is kept more current than this snapshot: <https://support.in-parallel.com/en/articles/691255-connect-your-ai-tool-to-in-parallel>

For production setups — scheduled jobs, shared team access, audit requirements — see [`mcp-advanced-auth.md`](mcp-advanced-auth.md).
