# TypeScript SPA Example (`@atproto/oauth-client-browser`)

This pattern is for public browser clients (no backend session broker).

## 1) Install

```bash
npm install @atproto/api @atproto/oauth-client-browser
```

## 2) Serve metadata at `/oauth-client-metadata.json`

### Read-only (sign-in + display profile)

This is the minimal scope set. Profile reads route through the public AppView at runtime (see Section 3), so no `rpc:` scope is needed.

```json
{
  "client_id": "https://app.example.com/oauth-client-metadata.json",
  "client_name": "Example SPA",
  "client_uri": "https://app.example.com",
  "redirect_uris": ["https://app.example.com/oauth/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "atproto",
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "application_type": "web",
  "dpop_bound_access_tokens": true
}
```

### Write-capable (list-management app, BLM-style)

Granular `repo:<nsid>?action=...` scopes per collection. No `transition:generic`. Reads still go through the public AppView.

```json
{
  "client_id": "https://app.example.com/oauth-client-metadata.json",
  "client_name": "Example List Manager",
  "client_uri": "https://app.example.com",
  "redirect_uris": ["https://app.example.com/oauth/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "atproto repo:app.bsky.graph.list?action=create&action=update&action=delete repo:app.bsky.graph.listitem?action=create&action=delete",
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "application_type": "web",
  "dpop_bound_access_tokens": true
}
```

## 3) OAuth client setup (two-agent pattern)

Use **two agents** at runtime: an unauthenticated `publicAgent` for all `app.bsky.*` reads, and the OAuth-bound `oauthAgent` for `com.atproto.repo.*` writes (and any reads that genuinely need viewer-bound state).

This split avoids the `agent.app.bsky.actor.getProfile()` → `401 Unauthorized` failure mode against non-bsky.social PDS deployments, where the OAuth-bound Agent proxies AppView reads through the user's PDS and the third-party PDS's proxy contract may be missing or incomplete.

```ts
// src/oauth.ts
import { Agent, AtpAgent } from '@atproto/api'
import {
  AtprotoDohHandleResolver,
  BrowserOAuthClient,
  OAuthSession,
} from '@atproto/oauth-client-browser'

// Unauthenticated AppView agent — use for ALL app.bsky.* reads.
// No OAuth, no DPoP, no scope cost, no PDS-proxy fragility.
export const publicAgent = new AtpAgent({
  service: 'https://public.api.bsky.app',
})

const client = await BrowserOAuthClient.load({
  clientId: 'https://app.example.com/oauth-client-metadata.json',
  // Browser runtime needs a DNS-capable resolver path.
  handleResolver: new AtprotoDohHandleResolver(
    'https://cloudflare-dns.com/dns-query',
  ),
})

export async function initSession(): Promise<null | {
  session: OAuthSession
  oauthAgent: Agent
  appState: string | null
}> {
  // Handles restore + OAuth callback processing.
  const result = await client.init()
  if (!result) return null

  // OAuth-bound agent — use ONLY for com.atproto.repo.* writes
  // and reads that require viewer-bound state.
  const oauthAgent = new Agent(result.session)
  return {
    session: result.session,
    oauthAgent,
    appState: result.state ?? null,
  }
}

export async function startLogin(identifier: string) {
  const appState = crypto.randomUUID()

  // Optional app-local correlation (library validates OAuth state internally).
  sessionStorage.setItem(
    `oauth:state:${appState}`,
    JSON.stringify({ startedAt: Date.now(), identifier }),
  )

  await client.signIn(identifier, { state: appState })
  throw new Error('unreachable: browser is redirected')
}

export async function restoreByDid(did: string) {
  const session = await client.restore(did)
  return new Agent(session)
}

export async function logout(did: string) {
  await client.revoke(did)
}
```

### Read vs write usage

```ts
// READ — use publicAgent. Works for any account on any PDS, no scope needed.
const profile = await publicAgent.app.bsky.actor.getProfile({ actor: did })
const lists = await publicAgent.app.bsky.graph.getLists({ actor: did })

// WRITE — use oauthAgent. Requires the matching repo:<nsid>?action=... scope.
await oauthAgent.com.atproto.repo.createRecord({
  repo: oauthAgent.did!,
  collection: 'app.bsky.graph.list',
  record: {
    $type: 'app.bsky.graph.list',
    purpose: 'app.bsky.graph.defs#curatelist',
    name: 'My list',
    createdAt: new Date().toISOString(),
  },
})
```

> **Why not just call `oauthAgent.app.bsky.actor.getProfile()`?** It works on bsky.social-hosted accounts but routes through the user's PDS, which acts as an AppView proxy. Third-party PDSes (eurosky.social, self-hosted, Cocoon, etc.) do not always implement that proxy correctly — the call returns `401 Unauthorized` even though OAuth succeeded. `publicAgent` calls the public AppView directly and works for every account regardless of which PDS hosts them.

## 4) App bootstrap

```ts
// src/main.ts
import { publicAgent, initSession, startLogin } from './oauth'

const form = document.querySelector<HTMLFormElement>('#login-form')!
const status = document.querySelector<HTMLElement>('#status')!

form.addEventListener('submit', async (event) => {
  event.preventDefault()
  const fd = new FormData(form)
  const identifier = String(fd.get('identifier') || '').trim()
  if (!identifier) return

  await startLogin(identifier)
})

const active = await initSession()
if (active) {
  // Profile read goes through publicAgent — robust across all PDS deployments.
  const profile = await publicAgent.app.bsky.actor.getProfile({
    actor: active.oauthAgent.did!,
  })
  status.textContent = `Signed in as @${profile.data.handle}`
} else {
  status.textContent = 'Not signed in'
}
```

## 5) Notes

- `BrowserOAuthClient` handles PAR, PKCE (`S256`), DPoP, and nonce retries.
- Use HTTPS in production.
- Keep identifier/DID cache freshness short during login (`<= 10 minutes`).
