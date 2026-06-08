# MCP Authentication — Bearer Tokens and OAuth

The level-two setup in [`connect-in-parallel-mcp.md`](connect-in-parallel-mcp.md) uses the **OAuth** connector flow — you authorize in a consent screen, pick which Workspaces to share, and never handle a token by hand. That's the right default for interactive use. The alternative is an **API key** (a long-lived bearer token), which In Parallel issues on request for headless or scheduled use where an interactive OAuth handshake can't run. At level five — scheduled jobs across a team, a shared toolkit, exposing the connection through your own systems — you'll want to think carefully about which of these you're using and how you rotate it.

This document covers both options for connecting to the In Parallel MCP at production grade: long-lived **bearer tokens / API keys** (with rotation and scoping), and **OAuth** (with delegated access and refresh).

> **Note on placeholders.** Specific URLs, scope names, and registration flows below are illustrative. Check In Parallel's authoritative auth documentation for current details. The shape and the trade-offs are what matters here.

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

**Example — direct HTTP call (Python):**

```python
import httpx
import os

token = os.environ["IN_PARALLEL_MCP_TOKEN"]

with httpx.Client(
    base_url="https://www.in-parallel.ai/mcp",  # MCP connector URL; REST path below is illustrative
    headers={"Authorization": f"Bearer {token}"},
    timeout=30.0,
) as client:
    response = client.post("/meetings/search", json={
        "workspace_id": "ws_xxxx",
        "since": "2026-05-20",
        "query": "customer:acme",
    })
    response.raise_for_status()
    meetings = response.json()
```

This is roughly what the cs-toolkit's snapshot script does — direct bearer-auth HTTP to fetch the day's meetings into the shared cache. No interactive session in the loop.

## OAuth

OAuth gives you per-user access with revocation and refresh — the right choice when several people use the same integration, or when you need an audit trail of who saw what.

**When to use:**

- A shared internal tool that several teammates will sign into
- A customer-facing integration where users authorise access to *their* In Parallel workspace
- Anything that needs a per-user audit trail
- Anything where you'd rather not store a long-lived secret on the user's behalf

**When not to use:**

- A solo cron job on your own laptop — bearer is simpler and just as safe
- One-off scripts where setting up an OAuth client is overkill

**Setup.** Register an OAuth client with In Parallel (Settings → Developer → OAuth applications — check the current path in In Parallel's docs). You'll provide:

- A **client name** — visible to users on the consent screen
- A **redirect URI** — where In Parallel sends users back after they authorise (your app's `/oauth/callback` endpoint)
- The **scopes** you need — for example `meetings:read`, `signals:read`

You'll receive a `client_id` and `client_secret`. Store the secret like any other production credential — never in the repo, never in client-side code.

**The flow — authorisation code with PKCE:**

1. Your app redirects the user to In Parallel's authorise URL with the `client_id`, requested `scope`, `redirect_uri`, a `state` value, and a PKCE `code_challenge`.
2. The user signs in to In Parallel (if they aren't already) and approves the requested scopes.
3. In Parallel redirects back to your `redirect_uri` with an authorisation `code`.
4. Your app exchanges the `code` for an access token (and refresh token) by POSTing to In Parallel's token endpoint with `client_id`, `client_secret`, the `code`, and the PKCE `code_verifier`.
5. Use the access token in `Authorization: Bearer` headers for subsequent MCP calls. When it expires, use the refresh token to get a new one.

**Skeleton (Python with `authlib`):**

```python
from authlib.integrations.requests_client import OAuth2Session

client = OAuth2Session(
    client_id="YOUR_CLIENT_ID",
    client_secret="YOUR_CLIENT_SECRET",
    scope="meetings:read signals:read",
    redirect_uri="https://your-app.example/oauth/callback",
)

# 1. Build the authorisation URL and redirect the user there
auth_url, state = client.create_authorization_url(
    "https://auth.in-parallel.com/oauth/authorize",  # placeholder
    code_challenge_method="S256",
)

# 2. After the user is redirected back to your callback,
#    exchange the code for an access token
token = client.fetch_token(
    "https://auth.in-parallel.com/oauth/token",      # placeholder
    authorization_response=request_url,
)

# 3. Use the token to call the MCP
response = client.get(
    "https://www.in-parallel.ai/mcp/meetings/search",  # MCP URL; REST path illustrative
    params={"customer": "acme"},
)
```

**Refresh.** Access tokens expire — typically within an hour. Use the refresh token to get a new access token without prompting the user again. Most OAuth libraries handle this transparently if you persist the refresh token. Rotate the refresh token on user logout.

**Revocation.** When a user signs out, or their access should be removed, revoke both the access and refresh tokens against In Parallel's revocation endpoint. Don't leave dangling tokens.

## Which to choose

| Question | Bearer | OAuth |
|---|---|---|
| Solo cron on my laptop? | ✓ | overkill |
| Shared team integration? | risky | ✓ |
| Audit trail per user? | no | ✓ |
| Customer-facing integration? | no | ✓ |
| Easiest to set up? | ✓ | no |
| Cleanest to rotate or revoke? | manual | ✓ |

The cs-toolkit uses bearer tokens for its own cron jobs — single user, single laptop, full control of the environment. A team product that exposes In Parallel data to colleagues would use OAuth.

## What to keep in mind

- **Scope tightly.** Whichever you use, ask for only the scopes you actually need. `meetings:read` is fine if you don't write — don't request `meetings:write` just in case.
- **Rotate on a schedule.** Bearer tokens quarterly. OAuth client secrets at least yearly.
- **Audit.** If In Parallel exposes an admin view of token usage, watch it. Tokens used from unexpected IPs are the first sign something has gone wrong.
- **Never paste tokens into chat.** The most common leak. Treat tokens like passwords.
