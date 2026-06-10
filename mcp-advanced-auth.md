# MCP Authentication — Bearer Tokens and OAuth

The level-two setup in [`connect-in-parallel-mcp.md`](connect-in-parallel-mcp.md) uses the **OAuth** connector flow — you authorise in a consent screen, pick which Workspaces to share, and never handle a token by hand. That's the right default for interactive use. The alternative is an **API key** (a long-lived bearer token), which In Parallel issues on request for headless or scheduled use where an interactive OAuth handshake can't run. At level five — scheduled jobs across a team, a shared toolkit, exposing the connection through your own systems — you'll want to think carefully about which of these you're using and how you rotate it.

This document covers both options for connecting to the In Parallel MCP at production grade: long-lived **bearer tokens / API keys** (with rotation and scoping), and **OAuth** (with delegated access and refresh).

> **Note.** The patterns below reflect what the production cs-toolkit actually runs. Where details may evolve (tool names, endpoints), verify against In Parallel's current documentation or `list_tools()`.

## Bearer tokens

The same kind of token used at level two — long-lived, attached to a workspace, sent in the `Authorization: Bearer` header.

**When to use:**

- A scheduled cron job that needs to read meetings (no human in the loop to do an OAuth handshake)
- A personal toolkit on your own laptop (level-five cs-toolkit territory)
- Local development against the MCP

**When not to use:**

- Anything that should track *which user* made a request — bearer tokens are workspace-scoped, so every call looks the same to the audit log
- Anything customer-facing where access should be revocable per-user
- Anything where the token could end up in a place you don't fully control

**Setup.** You don't self-generate an In Parallel API key — **request one from your In Parallel customer success manager**, telling them which AI client or system it's for. (See the [support article](https://support.in-parallel.com/en/articles/691255-connect-your-ai-tool-to-in-parallel) — API keys are the path for "headless agents, CI runners, or restricted environments" where OAuth can't complete.) When you receive it, give it a descriptive local name so you can recognise it later: `cs-toolkit-cron-2026`, not `token-1`.

Store the token somewhere that isn't your repo. Two patterns that work:

```bash
# Option A — a dedicated dotenv file outside the repo
# ~/.config/in-parallel/.env
IN_PARALLEL_MCP_TOKEN=ip_mcp_xxxxxxxxxxxxxxxxxxxx
```

```bash
# Option B — macOS keychain (or the equivalent on Linux/Windows)
security add-generic-password \
  -s in-parallel-mcp \
  -a $USER \
  -w ip_mcp_xxxxxxxxxxxxxxxxxxxx
```

In a cron job, load the token from the env or keychain at start. Never commit it. Never paste it into a chat. Rotate it every quarter; revoke immediately on team changes or laptop loss.

**Example — calling the MCP from a script (Python):**

One thing to be clear about: the MCP endpoint speaks the MCP protocol (JSON-RPC over HTTP), not REST. You don't POST to resource paths — you open an MCP session and call the server's *tools*. The bearer token rides along as a header. With the official [`mcp` Python SDK](https://github.com/modelcontextprotocol/python-sdk):

```python
import asyncio, os
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

token = os.environ["IN_PARALLEL_MCP_TOKEN"]

async def fetch_meetings():
    async with streamablehttp_client(
        "https://www.in-parallel.ai/mcp",
        headers={"Authorization": f"Bearer {token}"},
    ) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await session.list_tools()   # discover what the server offers
            return await session.call_tool(
                "list_meeting_records",
                {"start_date": "2026-06-01", "end_date": "2026-06-10"},
            )

meetings = asyncio.run(fetch_meetings())
```

The In Parallel MCP exposes read tools like `list_meeting_records`, `get_meeting_record`, `get_transcript`, `list_action_items`, and `list_workspaces` — `list_tools()` shows the current set. This is what the cs-toolkit's scheduled snapshot does in production: an unattended bearer-auth MCP session that fetches the day's meetings into the shared cache, no interactive chat client in the loop. (The toolkit uses a thin JSON-RPC client of its own; the SDK gets you the same thing with less code, and absorbs one wrinkle for you — the server frames responses as server-sent events even for single-shot calls.)

## OAuth

OAuth is how the *interactive* connections work — and the practical thing to understand is that **you don't build any of it yourself**. The MCP specification includes an OAuth flow, and the AI clients implement it end to end: when you add the In Parallel connector to Claude, ChatGPT, or Microsoft 365 Copilot, the client discovers the server's authorisation endpoint, registers itself, and walks you through the consent screen (choose Workspaces → **Allow access**). Access tokens, refresh, and revocation are handled between the client and In Parallel. You never see a token, which is exactly the point.

**When OAuth is the right answer:**

- Interactive use in an AI client — it's the default, and there's nothing to set up beyond the consent screen
- Several teammates connecting their own AI tools — each consent is scoped to the user who gave it, and revocable per user
- Anything that needs a per-user audit trail

**Where it can't help (today):**

- Headless or scheduled runs — there's no one present to complete the consent screen. That's the API-key path above.
- Your own server-side application: In Parallel doesn't currently offer self-serve OAuth client registration (a developer portal issuing `client_id`/`client_secret`). If you're building a multi-user product on In Parallel data, talk to your In Parallel contact about options — and resist the shortcut of sharing one bearer key across users, because a workspace-scoped key makes every user look identical in the audit log.

If you're building your own *agent* rather than using a hosted client, the official MCP SDKs implement the client side of the MCP OAuth flow, so the same consent-screen path works from your own code. That, or the API-key header — those are the two supported shapes. Don't hand-roll an OAuth client against guessed endpoints.

**Revocation.** Audit your AI clients' connector lists periodically and disconnect what's no longer used — each disconnect revokes that consent. For API keys, revocation goes through the same channel that issued them.

## Which to choose

| Question | Bearer (API key) | OAuth (connector flow) |
|---|---|---|
| Solo cron on my laptop? | ✓ | can't — no one to consent |
| Interactive use in an AI client? | works, but unnecessary | ✓ default |
| Several teammates, each in their own AI tool? | risky — one shared key | ✓ per-user consent |
| Audit trail per user? | no — workspace-scoped | ✓ |
| Your own multi-user product? | no | talk to In Parallel |
| Revocation? | via the issuing channel | disconnect the connector |

The cs-toolkit uses a bearer token for its own scheduled jobs — single operator, full control of the environment — and the ordinary OAuth connector flow for interactive sessions. That split is the recommended default.

## What to keep in mind

- **Scope tightly.** In Parallel's unit of scoping is the Workspace: grant a connection only the Workspaces it needs, whether at the OAuth consent screen or when requesting an API key. Don't take "Full access" just in case.
- **Rotate on a schedule.** Bearer tokens quarterly. OAuth consents don't need rotation — instead, periodically disconnect connectors you no longer use.
- **Audit.** If In Parallel exposes an admin view of token usage, watch it. Tokens used from unexpected IPs are the first sign something has gone wrong.
- **Never paste tokens into chat.** The most common leak. Treat tokens like passwords.
