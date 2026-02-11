# TypeScript SPA Example (`@atproto/oauth-client-browser`)

This pattern is for public browser clients (no backend session broker).

## 1) Install

```bash
npm install @atproto/api @atproto/oauth-client-browser
```

## 2) Serve metadata at `/oauth-client-metadata.json`

```json
{
  "client_id": "https://app.example.com/oauth-client-metadata.json",
  "client_name": "Example SPA",
  "client_uri": "https://app.example.com",
  "redirect_uris": ["https://app.example.com/oauth/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "atproto rpc:app.bsky.actor.getProfile?aud=did:web:api.bsky.app#bsky_appview blob:*/*",
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "application_type": "web",
  "dpop_bound_access_tokens": true
}
```

## 3) OAuth client setup

```ts
// src/oauth.ts
import { Agent } from '@atproto/api'
import {
  AtprotoDohHandleResolver,
  BrowserOAuthClient,
  OAuthSession,
} from '@atproto/oauth-client-browser'

const client = await BrowserOAuthClient.load({
  clientId: 'https://app.example.com/oauth-client-metadata.json',
  // Browser runtime needs a DNS-capable resolver path.
  handleResolver: new AtprotoDohHandleResolver(
    'https://cloudflare-dns.com/dns-query',
  ),
})

export async function initSession(): Promise<null | {
  session: OAuthSession
  agent: Agent
  appState: string | null
}> {
  // Handles restore + OAuth callback processing.
  const result = await client.init()
  if (!result) return null

  const agent = new Agent(result.session)
  return {
    session: result.session,
    agent,
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

## 4) App bootstrap

```ts
// src/main.ts
import { initSession, startLogin } from './oauth'

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
  const profile = await active.agent.getProfile({ actor: active.agent.did })
  status.textContent = `Signed in as @${profile.data.handle}`
} else {
  status.textContent = 'Not signed in'
}
```

## 5) Notes

- `BrowserOAuthClient` handles PAR, PKCE (`S256`), DPoP, and nonce retries.
- Use HTTPS in production.
- Keep identifier/DID cache freshness short during login (`<= 10 minutes`).
