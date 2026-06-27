# Phase 10: Agent Authentication & API Access

Make the app accessible to AI agents, scripts, and MCP servers. Adds a dedicated agent token endpoint that mints long-lived JWTs from the caller's existing session, exposes the OpenAPI schema for agent discovery, and documents the agent workflow.

**Prerequisite:** Phase 02 complete (auth system in place). Can be done at any point after Phase 02.

---

## Overview

Agents authenticate using the same JWT system as browser users, but with a dedicated endpoint that issues long-lived tokens (up to 1 year). The endpoint is **protected** — the caller must already hold a valid session JWT (obtained via the normal `/login` or `/google` flow). Agents get a Bearer token and use the exact same API endpoints as the frontend.

| | Browser Users | Agents |
|---|---|---|
| **Token endpoint** | `POST /api/auth/login` | `POST /api/auth/agent-token` |
| **Auth required to call it** | None (public) | Bearer of existing session JWT |
| **Token format** | JWT (Bearer) | JWT (Bearer) — same format |
| **Token lifetime** | 7 days (default) | 1-365 days (configurable per request) |
| **Auth header** | `Authorization: Bearer <token>` | Same |
| **API access** | All endpoints | All endpoints — same routes |

**Why bearer-auth, not password re-prompt:** the user is already signed in via the frontend when they request a long-lived token. Re-asking for the password adds friction without meaningfully improving security — if a session JWT is stolen, an attacker can already do anything the user can do (CRUD on all the user's resources). Demanding a password to mint a longer token only protects against one specific theft scenario (stolen session, intact password) and breaks the Google-only sign-in flow entirely (those users have no password). Re-using the session is simpler, works for all sign-in methods, and matches the threat model.

---

## Agent Token DTOs

Add to `api/schemas/dto/auth.py`:

```python
from pydantic import BaseModel, Field

class AgentTokenRequest(BaseModel):
    expires_in_days: int = Field(default=30, ge=1, le=365)

class AgentTokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expires_in_days: int
```

No email or password — the caller's identity comes from the session JWT.

---

## Update Token Creation

Update `create_access_token` in `api/utils/auth.py` to accept an optional `expires_delta`:

```python
from datetime import datetime, timedelta, timezone
from typing import Optional

def create_access_token(subject: str, expires_delta: Optional[timedelta] = None) -> str:
    settings = get_settings()
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(minutes=settings.jwt_expire_minutes)
    payload = {"sub": subject, "exp": expire}
    return jwt.encode(payload, settings.jwt_secret, algorithm=settings.jwt_algorithm)
```

The browser login path continues to use the default expiry. The agent endpoint passes a custom `expires_delta`.

---

## Agent Token Endpoint

Add to `api/routes/auth.py`:

```python
from datetime import timedelta
from fastapi import Depends
from api.schemas.dto.auth import AgentTokenRequest, AgentTokenResponse
from api.utils.auth import get_current_user

@router.post("/agent-token", response_model=AgentTokenResponse)
async def agent_token(
    data: AgentTokenRequest,
    user: User = Depends(get_current_user),
):
    """Mint a long-lived JWT for the currently-authenticated user.

    Caller must hold a valid session JWT (obtained via /login or /google).
    The new token has the same format and identity — it just lives longer.
    """
    access_token = create_access_token(
        str(user.id),
        expires_delta=timedelta(days=data.expires_in_days),
    )
    return AgentTokenResponse(
        access_token=access_token,
        expires_in_days=data.expires_in_days,
    )
```

Key points:
- `Depends(get_current_user)` requires a valid session JWT — same dependency used to protect every other authenticated endpoint
- No email/password lookup, no special checks for verified email or active user — `get_current_user` already enforces all of that
- Works identically for password users and Google-only users
- The new token is a standard Bearer JWT — `get_current_user` accepts it transparently
- All operations on the new token are scoped to the authenticated user

---

## Expose OpenAPI Schema for Agents

Update the FastAPI app in `api/main.py` to expose the API docs at an agent-friendly URL:

```python
app = FastAPI(
    title="MyApp API",
    description="MyApp API — used by the frontend and AI agents",
    version="0.1.0",
    lifespan=lifespan,
    docs_url="/api/agent",           # Swagger UI for agents
    openapi_url="/api/openapi.json", # Machine-readable schema
    redoc_url=None,                  # Disable redoc (one docs UI is enough)
)
```

Add a convenience endpoint that returns the schema directly:

```python
@app.get("/api/schema")
async def get_schema():
    return app.openapi()
```

This gives agents three ways to discover the API:
- **`/api/openapi.json`** — standard OpenAPI 3.x schema (machine-readable)
- **`/api/schema`** — same schema via a simple GET (useful for `curl` or `fetch`)
- **`/api/agent`** — interactive Swagger UI (useful for debugging)

---

## Frontend: agent-token UI on the profile page

The newbuild app has a single profile page that holds all settings. Add an "Agent tokens" section there. Skip the separate `/agent-auth` route — the user is already signed in when they want a token.

The section needs:
- Always-visible discovery URLs (Swagger UI, OpenAPI schema, API base) so agents have something to look at without minting a token first
- A **Download skill** button next to Generate (see next section) that produces a markdown bootstrap doc the user hands to their agent alongside the token
- A "Generate" button that opens a duration picker (1 / 7 / 30 / 90 / 365 days; default 365 for cron-driven agents)
- POST `/api/auth/agent-token` with the duration — the existing session token in `localStorage` authenticates the request
- A read-only textarea showing the JWT once, with a "Copy to clipboard" button — token is **shown once and discarded from React state** when dismissed; no token persistence

API client method:

```typescript
async agentToken(expiresInDays: number) {
  return this.post<{ access_token: string; token_type: string; expires_in_days: number }>(
    '/api/auth/agent-token',
    { expires_in_days: expiresInDays },
  )
}
```

The `post` helper already attaches the session `Authorization: Bearer <session-jwt>` header — that's what authenticates the request.

---

## Frontend: downloadable agent setup guide

The token alone isn't enough — an agent that has never seen this app needs to know what it does, what to ask the operator for if it's missing the base URL or token, where the schema lives, and how to behave when it gets a 4xx. Generate that as a downloadable markdown file from the same profile section. Together, **token + setup guide** are the two artifacts a user hands their agent to bootstrap a session.

### Design rules (do not break)

- **Audience is the agent at SETUP time, not runtime.** The markdown is onboarding instructions, not a runtime cookbook. It tells the agent what it needs and what to ask the operator for, then points at the schema for everything else.
- **Stay framework-agnostic.** Don't prescribe `curl`, `jq`, environment variable names, or other tool-specific idioms — the file is meant to work for Claude Code, Codex, openclaw, MCP servers, and custom scripts equally. Stay at the contract level: HTTP header rules, status codes, schema URLs.
- **Generate it client-side.** No backend route. The content depends only on app metadata + `window.location.origin`. Means no caching to think about and the file picks up whatever host you're served from (localhost, Pi, Cloudflare tunnel) automatically.
- **Bootstrap, not reference.** The markdown must NOT pin specific endpoint paths — endpoints change between releases. Tell the agent to read `/api/openapi.json` instead. The doc covers data model, auth header rule, discovery instructions, and a failure-modes table.
- **Never embed the token.** Token + guide are separate artifacts. The guide stays static and copyable; the token is sensitive and ephemeral.
- **Use `<base>` as a placeholder in discovery URLs.** The literal known base URL appears once for context and once in the footer, but the URLs the agent should *call* are templated as `<base>/api/openapi.json` so the agent treats them as substitutable when the host has moved.
- **Keep the template reusable across apps.** Future apps copy the file, change four strings, ship.

### `frontend/src/lib/agentSkill.ts`

```typescript
export interface AgentSkillInput {
  /** Lowercase app slug, used as filename and shown in the title (e.g. "invest"). */
  appName: string
  /** Origin of the app, e.g. "https://invest.example.com". */
  baseUrl: string
  /** One paragraph: what the app is for. No endpoint names — describe purpose. */
  description: string
  /** Short paragraph describing the data model in plain English (resources + relationships). */
  dataModel: string
  /** Optional list of "things this agent is typically asked to do" (individual tasks). */
  capabilities?: string[]
  /**
   * Optional list of higher-order setups the agent can proactively offer the
   * operator — recurring jobs, on-demand skills, alerting flows, etc.
   * Different from `capabilities` (individual tasks): suggestions are standing
   * arrangements the operator might want once and benefit from forever.
   */
  suggestions?: string[]
}

export function agentSkill(input: AgentSkillInput): string {
  const { appName, baseUrl, description, dataModel, capabilities = [] } = input
  const base = baseUrl.replace(/\/$/, '')

  const capsBlock = capabilities.length
    ? `\n## Typical tasks an operator will give you\n\n${capabilities
        .map((c) => `- ${c}`)
        .join('\n')}\n`
    : ''

  return `# ${appName} — agent setup guide

You are an AI agent (Claude, Codex, openclaw, MCP server, custom script — doesn't
matter which) being asked to work with **${appName}**. This document is your
onboarding: it tells you what the app does, what you need from the operator
before you can talk to it, and how to find the rest on your own.

Read this once. If you're missing anything from "What you need" below, ask the
operator before doing anything else. Don't guess and don't retry.

## What this app is

${description}

${dataModel}

## What you need before you can call anything

Two pieces of information. Neither is encoded in this file.

1. **The app's base URL.** Where the API actually lives. The most recent value
   the app knew about is \`${base}\`, but the operator may have moved it
   (Cloudflare tunnel, different host, etc.) — confirm if you're unsure.

2. **A bearer token (JWT).** The operator mints this themselves from the
   profile page of the running app, under "Agent tokens". It's a long opaque
   string scoped to that operator's user account; it lives for a configurable
   1–365 days.

Store both however your runtime handles secrets (environment variable, MCP
config, secrets file, vault entry — your call). Never log the token, never
echo it back, never write it to a committed file.

### What to ask the operator if you're missing either

Be specific so they know exactly what to hand back:

> I need two things to start working with ${appName}: (1) the base URL where
> it's running, and (2) a bearer token from the "Agent tokens" section of your
> profile page on that app. Can you provide both?

## How to authenticate every request

One header on every call:

\`\`\`
Authorization: Bearer <the token>
\`\`\`

That's the whole auth model. No refresh tokens, no OAuth flow, no per-endpoint
scopes. Same token on every request until it expires.

## How to discover endpoints

**Don't assume endpoint paths.** The app publishes its full API surface as a
standard OpenAPI 3.x schema. Fetch this once at the start of a session and
work from it:

- Machine-readable schema: \`<base>/api/openapi.json\`
- Browsable Swagger UI: \`<base>/api/agent\`

To plan a task: pull the schema, search \`paths\` for endpoints relevant to
the task (by tag, path keyword, or method), read the request and response
schemas, match your payload exactly. If something stops working, re-read
the schema — it's the source of truth.

## Idiomatic behavior the app expects

- **Use idempotent endpoints when they exist.** Endpoints whose schema
  description mentions "bulk", "upsert", or "idempotent" can safely be
  re-run; prefer them for backfills over many single-record creates.
- **Stay in your user.** Every resource is scoped to the authenticated user.
- **Don't overwrite what the human writes.** Anything described as
  "user-authored" is for the operator to edit in the UI; agents append to
  versioned/AI siblings instead.
- **Respect your cadence.** If you've been scheduled to run daily, don't
  poll hourly without a reason. If hourly, don't burst.

## Status codes that aren't bugs

| Status | Meaning | What to do |
|---|---|---|
| 401 | Token missing, malformed, or expired | Stop. Ask the operator for a fresh one. Do not retry. |
| 403 | Authenticated but the action is forbidden for this user | Read the response body. Do not retry. |
| 404 | Resource doesn't exist or isn't yours | Don't retry; the ID is stale. |
| 409 | Conflict (usually a unique-key collision) | If your goal was idempotent insert, treat as success. |
| 422 | Payload doesn't match the schema | Re-read the schema for that path, fix the body, retry once. |
| 5xx | Server error | Backoff with jitter. Alert if it persists. |
${capsBlock}
## When you can't make progress

Surface specifics to the operator: which endpoint, what payload, what status,
what the response body said. Don't loop. The operator can mint a fresh token,
correct the base URL, or check server logs you can't see.

---

*Generated by ${appName} at \`${base}\`. The endpoint list is intentionally
not pinned here — fetch \`${base}/api/openapi.json\` for that.*
`
}

export function downloadSkill(appName: string, markdown: string) {
  const blob = new Blob([markdown], { type: 'text/markdown;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `${appName}-agent.md`
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}
```

### Wire it into `AgentTokenSection`

Add a `Download skill` button next to `Generate` in the section header. The handler builds the markdown with the four app-specific strings and triggers the download:

```typescript
import { agentSkill, downloadSkill } from '../lib/agentSkill'

const handleDownloadSkill = () => {
  const md = agentSkill({
    appName: 'myapp',                     // ← change per app
    baseUrl: window.location.origin,
    description: 'One paragraph on what the app is and who uses it.',
    dataModel: 'Plain-English data model summary — resources and relationships.',
    capabilities: [
      'Typical agent task #1',
      'Typical agent task #2',
    ],
    suggestions: [
      'Set up a recurring job that does X every <interval>, alerting on <condition>',
      'Build an on-demand skill that, given <ask>, returns <shape>',
    ],
  })
  downloadSkill('myapp', md)              // ← change per app
}
```

The button itself sits in the section header alongside Generate (use a flex container so they stack on mobile and row on desktop). Keep `Download skill` outlined / secondary so `Generate` stays the primary CTA.

### Tests (regression-guard the contract)

Add `frontend/src/lib/agentSkill.test.ts` to lock in the rules that make this template work. Without these the design quietly drifts:

```typescript
import { describe, expect, it } from 'vitest'
import { agentSkill } from './agentSkill'

const sample = {
  appName: 'demo',
  baseUrl: 'https://demo.example.com',
  description: 'demo is a placeholder.',
  dataModel: 'There is one Thing.',
  capabilities: ['do a thing'],
}

describe('agentSkill template', () => {
  it('uses <base> placeholders in discovery URLs (not the literal base)', () => {
    // Discovery URLs are templated so the agent treats them as substitutable
    // when the host has moved.
    const md = agentSkill(sample)
    expect(md).toContain('<base>/api/openapi.json')
    expect(md).toContain('<base>/api/agent')
  })

  it('strips a trailing slash from baseUrl in any literal mention', () => {
    const md = agentSkill({ ...sample, baseUrl: 'https://demo.example.com/' })
    expect(md).not.toContain('https://demo.example.com//')
  })

  it('does not list any specific endpoint path', () => {
    const md = agentSkill(sample)
    expect(md).not.toMatch(/\/api\/(stocks|things|tasks|users)\b/)
  })

  it('does not embed any token-shaped value', () => {
    const md = agentSkill(sample)
    expect(md).not.toMatch(/eyJ[A-Za-z0-9_-]{10,}/)
  })

  it('tells the agent what to ask the operator for', () => {
    // Setup-time framing: the guide must teach the agent how to onboard.
    const md = agentSkill(sample)
    expect(md).toContain('What you need before you can call anything')
    expect(md).toMatch(/ask the operator/i)
    expect(md).toContain('Authorization: Bearer')
  })

  it('includes the failure-modes table', () => {
    const md = agentSkill(sample)
    expect(md).toContain('| 401 |')
    expect(md).toContain('| 422 |')
  })
})
```

---

## Agent Workflow

### 1. Get a token

Open the app in a browser, go to `/profile`, click "Generate" under Agent tokens, pick a duration (e.g. 365 days), and copy the JWT.

For headless setups (no browser), call the endpoint directly with a session token obtained via `/login`:

```bash
SESSION=$(curl -s -X POST https://myapp.example.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"david@example.com","password":"…"}' | jq -r .access_token)

curl -X POST https://myapp.example.com/api/auth/agent-token \
  -H "Authorization: Bearer $SESSION" \
  -H "Content-Type: application/json" \
  -d '{"expires_in_days": 365}'
```

Response:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "expires_in_days": 365
}
```

### 2. Discover the API

```bash
curl https://myapp.example.com/api/openapi.json
```

Or visit `https://myapp.example.com/api/agent` in a browser for interactive docs.

### 3. Call endpoints

```bash
# List things
curl https://myapp.example.com/api/things \
  -H "Authorization: Bearer <access_token>"

# Create a thing
curl -X POST https://myapp.example.com/api/things \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "New thing", "description": "Created by agent"}'
```

### 4. Refresh when needed

Tokens don't auto-refresh. When a token expires, mint a new one (sign in via the UI again, or use the bash flow in step 1). For long-running agents, request a 365-day token.

---

## Agent User Account (optional)

For multi-user apps you can create a dedicated user for agents (so the agent's data doesn't mix with a human user's). For single-user apps it's fine to use your own account — the long-lived token is scoped to that user just like any other token.

---

## Checklist

- [ ] `AgentTokenRequest` and `AgentTokenResponse` DTOs added to `api/schemas/dto/auth.py` (no email/password)
- [ ] `create_access_token` updated to accept optional `expires_delta`
- [ ] `POST /api/auth/agent-token` endpoint added with `Depends(get_current_user)`
- [ ] FastAPI app configured with `docs_url="/api/agent"` and `openapi_url="/api/openapi.json"`
- [ ] `GET /api/schema` convenience endpoint added
- [ ] Frontend "Agent tokens" section added to the profile page (duration picker + copy-to-clipboard, no password re-prompt)
- [ ] `frontend/src/lib/agentSkill.ts` created with `agentSkill()` template + `downloadSkill()` helper
- [ ] `Download skill` button wired into the profile section, generating `<appname>-agent.md` with app-specific `description` / `dataModel` / `capabilities`
- [ ] `frontend/src/lib/agentSkill.test.ts` regression tests pass (no endpoints leaked, no JWT-shaped values, base URL sanitized)
- [ ] Authenticated user can mint a long-lived token from the profile UI
- [ ] OpenAPI schema accessible at `/api/openapi.json`
- [ ] Swagger UI accessible at `/api/agent`
