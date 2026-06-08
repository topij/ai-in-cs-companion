# Connecting to the In Parallel MCP — Level 2

The single step from level one to level two: give your AI its own access to your data instead of copy-pasting it. This document covers connecting to the In Parallel MCP server from Claude Desktop and ChatGPT.

> **Note on placeholders.** The URLs, setting-screen paths, and exact field names below are illustrative. The authoritative setup details live in In Parallel's own docs — check there first. This file shows the *shape* of the configuration, which is what most people get stuck on.

## What you need

- An **In Parallel account** with access to the workspace you want to expose. The free tier works for trying it.
- An **MCP token** issued from your In Parallel settings (Settings → Developer → MCP tokens, or similar — confirm in In Parallel's docs).
- A compatible client. The two most common:
  - **Claude Desktop** (macOS, Windows)
  - **ChatGPT** with custom connectors enabled (Plus / Team / Enterprise plans)

## Claude Desktop

Claude Desktop reads a JSON config file at startup. Add an entry for the In Parallel MCP server.

**Config file locations:**

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

If the file doesn't exist yet, create it.

**The config:**

```json
{
  "mcpServers": {
    "in-parallel": {
      "url": "https://mcp.in-parallel.com",
      "headers": {
        "Authorization": "Bearer YOUR_IN_PARALLEL_TOKEN"
      }
    }
  }
}
```

Replace `YOUR_IN_PARALLEL_TOKEN` with the token from your In Parallel settings. (Confirm the exact host name in In Parallel's docs — the one above is illustrative.)

If you already have other MCP servers configured, just add the `in-parallel` entry alongside them inside `mcpServers`.

**Then restart Claude Desktop.** Quit it fully — not just close the window — and reopen. Claude reads the config only at launch.

**Verify it's working.** In a fresh chat, try:

```
What In Parallel tools do you have available?
```

Claude will list the tools the In Parallel MCP exposes — usually things like searching meetings, reading specific meeting summaries, and querying signals. If you see a tool list, you're connected.

## ChatGPT

ChatGPT's MCP support is evolving — exact steps vary by plan and may have changed since this file was written. The pattern: open ChatGPT settings, go to *Connectors* (or *Custom GPTs* / *MCP servers*), and add a new connector with the In Parallel server URL and your bearer token. Check OpenAI's MCP setup docs and In Parallel's connector page for the current flow.

The token and URL are the same as the Claude Desktop config — the only difference is where you paste them.

## Other Claude products

- **Cowork** — MCP setup lives in Settings → Integrations. Paste the In Parallel URL and token there.
- **Claude Code** — uses the same `claude_desktop_config.json` structure, or a project-local `.mcp.json` for repo-scoped servers.

## A few starter prompts

Once connected, try these to confirm everything's wired up and to start feeling the level-two difference:

```
What meetings did I have with [customer name] last week? Summarise each one in a sentence.
```

```
Across the last month of meetings, what topics came up most often with [customer name]?
```

```
Find every meeting in the last two weeks where someone raised a concern about [topic]. Quote the moment it came up.
```

## If it doesn't work

- **The connection errors while you're adding it.** This is a *connect-time* failure
  — distinct from the ones below, which all assume an already-configured server. If
  the add/sign-in step itself fails (an OAuth consent screen that errors, a login via
  your identity provider that won't complete, a "couldn't connect" on save), check:
  the server URL/host is exactly right (no trailing path or typo); the
  redirect/callback completes (some browsers or pop-up blockers interrupt it — try
  the in-app connector flow or a different browser); and the account you're signing in
  with actually has access to the workspace. If it persists, it's an auth/registration
  issue on the server side rather than your config — check In Parallel's docs or ask
  their support.
- **Claude doesn't see the In Parallel tools.** You probably need a full restart, not just reopen. Quit Claude completely and relaunch.
- **Auth errors.** The token may have expired or been revoked. Generate a new one from In Parallel settings.
- **Empty results.** Confirm the token belongs to the workspace that contains the meetings you're asking about. Tokens are workspace-scoped by default — and remember a meeting is only visible once it's been published to a workspace the token can reach.
- **Anything else.** Open In Parallel's support docs or message their support — MCP setup specifics evolve faster than a snapshot document like this one.

For production setups — scheduled jobs, shared team access, audit requirements — see [`mcp-advanced-auth.md`](mcp-advanced-auth.md).
