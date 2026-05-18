# TypeScript BFF Example (`@atproto/oauth-client-node`)

This is the recommended confidential-client pattern (Backend For Frontend).

## 1) Install

```bash
npm install express express-session @atproto/api @atproto/oauth-client-node @atproto/jwk-jose
npm install -D @types/express @types/express-session typescript
```

## 2) Metadata served by backend

Use `https://app.example.com/oauth-client-metadata.json` as `client_id`.

## 3) Server implementation

```ts
// src/server.ts
import crypto from 'node:crypto'
import express from 'express'
import session from 'express-session'
import { Agent, AtpAgent } from '@atproto/api'
import { JoseKey } from '@atproto/jwk-jose'
import {
  NodeOAuthClient,
  NodeSavedSession,
  NodeSavedState,
} from '@atproto/oauth-client-node'

declare module 'express-session' {
  interface SessionData {
    did?: string
    oauthState?: string
  }
}

const stateDb = new Map<string, NodeSavedState>()
const oauthSessionDb = new Map<string, NodeSavedSession>()

const keyset = await Promise.all([
  JoseKey.fromImportable(process.env.OAUTH_PRIVATE_KEY_PEM!, 'primary'),
])

const oauth = new NodeOAuthClient({
  clientMetadata: {
    client_id: 'https://app.example.com/oauth-client-metadata.json',
    client_name: 'Example BFF',
    client_uri: 'https://app.example.com',
    logo_uri: 'https://app.example.com/logo.png',
    tos_uri: 'https://app.example.com/tos',
    policy_uri: 'https://app.example.com/privacy',
    redirect_uris: ['https://app.example.com/oauth/callback'],
    grant_types: ['authorization_code', 'refresh_token'],
    // Granular scopes only. Reads of app.bsky.* go through the public AppView
    // (see publicAgent below), so no rpc: scope is needed for getProfile.
    // Add granular repo:<nsid>?action=... scopes here for any collection the
    // app actually writes (e.g., repo:app.bsky.feed.post?action=create&action=delete).
    scope: 'atproto blob:*/* account:email',
    response_types: ['code'],
    application_type: 'web',
    token_endpoint_auth_method: 'private_key_jwt',
    token_endpoint_auth_signing_alg: 'ES256',
    dpop_bound_access_tokens: true,
    jwks_uri: 'https://app.example.com/oauth/jwks.json',
  },
  keyset,
  stateStore: {
    async set(key, value) {
      stateDb.set(key, value)
    },
    async get(key) {
      return stateDb.get(key)
    },
    async del(key) {
      stateDb.delete(key)
    },
  },
  sessionStore: {
    async set(sub, value) {
      oauthSessionDb.set(sub, value)
    },
    async get(sub) {
      return oauthSessionDb.get(sub)
    },
    async del(sub) {
      oauthSessionDb.delete(sub)
    },
  },
})

// Unauthenticated AppView agent — use for ALL app.bsky.* reads.
// Avoids routing through the user's PDS, which fails on non-bsky.social
// PDS deployments that don't fully implement the AppView-proxy contract
// (returns 401 even though OAuth succeeded).
const publicAgent = new AtpAgent({ service: 'https://public.api.bsky.app' })

const app = express()

app.use(
  session({
    name: 'example.sid',
    secret: process.env.APP_SESSION_SECRET!,
    resave: false,
    saveUninitialized: false,
    cookie: {
      httpOnly: true,
      sameSite: 'lax',
      secure: true,
      maxAge: 1000 * 60 * 60 * 24 * 7,
    },
  }),
)

// Mandatory discoverable endpoints.
app.get('/oauth-client-metadata.json', (_req, res) => {
  res.json(oauth.clientMetadata)
})
app.get('/oauth/jwks.json', (_req, res) => {
  res.json(oauth.jwks)
})

app.get('/oauth/login', async (req, res, next) => {
  try {
    const identifier = String(req.query.identifier || '').trim()
    if (!identifier) {
      res.status(400).json({ error: 'identifier is required' })
      return
    }

    const appState = crypto.randomUUID()
    req.session.oauthState = appState

    const authorizeUrl = await oauth.authorize(identifier, {
      state: appState,
    })

    res.redirect(authorizeUrl.toString())
  } catch (err) {
    next(err)
  }
})

app.get('/oauth/callback', async (req, res, next) => {
  try {
    const url = new URL(`${req.protocol}://${req.get('host')}${req.originalUrl}`)
    const { session: oauthSession, state } = await oauth.callback(
      url.searchParams,
    )

    // Application-level CSRF/correlation check in addition to OAuth checks.
    if (!state || state !== req.session.oauthState) {
      res.status(400).json({ error: 'state mismatch' })
      return
    }

    req.session.did = oauthSession.did

    // Profile read via public AppView, not the OAuth-bound agent.
    // The OAuth agent is reserved for com.atproto.repo.* writes.
    const profile = await publicAgent.app.bsky.actor.getProfile({
      actor: oauthSession.did,
    })

    res.json({
      did: oauthSession.did,
      handle: profile.data.handle,
      grantedScope: oauthSession.tokenSet.scope,
    })
  } catch (err) {
    next(err)
  }
})

app.get('/me', async (req, res, next) => {
  try {
    if (!req.session.did) {
      res.status(401).json({ error: 'not authenticated' })
      return
    }

    const oauthSession = await oauth.restore(req.session.did)
    // Profile read via public AppView, not the OAuth-bound agent.
    // The OAuth agent is reserved for com.atproto.repo.* writes.
    const profile = await publicAgent.app.bsky.actor.getProfile({
      actor: oauthSession.did,
    })

    res.json({ did: oauthSession.did, handle: profile.data.handle })
  } catch (err) {
    next(err)
  }
})

app.post('/oauth/logout', async (req, res, next) => {
  try {
    if (req.session.did) {
      await oauth.revoke(req.session.did)
      req.session.did = undefined
    }
    req.session.oauthState = undefined
    res.status(204).end()
  } catch (err) {
    next(err)
  }
})

app.listen(3000, () => {
  console.log('BFF listening on https://app.example.com')
})
```

## 4) Write path (when the app actually writes records)

Writes go through the OAuth-bound `Agent`, **not** `publicAgent`. Add the matching `repo:<nsid>?action=...` scope to the metadata first.

```ts
app.post('/posts', express.json(), async (req, res, next) => {
  try {
    if (!req.session.did) {
      res.status(401).json({ error: 'not authenticated' })
      return
    }
    const oauthSession = await oauth.restore(req.session.did)
    const oauthAgent = new Agent(oauthSession)

    const result = await oauthAgent.com.atproto.repo.createRecord({
      repo: oauthAgent.did!,
      collection: 'app.bsky.feed.post',
      record: {
        $type: 'app.bsky.feed.post',
        text: String(req.body.text || ''),
        createdAt: new Date().toISOString(),
      },
    })

    res.json({ uri: result.data.uri })
  } catch (err) {
    next(err)
  }
})
```

To enable this endpoint, the metadata `scope` string must include `repo:app.bsky.feed.post?action=create&action=delete` (and any other collection the app writes).

## 5) Notes

- This pattern is preferred over pure SPA token storage for long-lived sessions.
- TMB and Client Assertion Backend are alternatives, but BFF is the default recommendation.
- Keep token/session records in durable encrypted storage in production.
- **Reads vs writes:** `publicAgent` (unauthenticated, `https://public.api.bsky.app`) for `app.bsky.*` reads; `oauthAgent` (`new Agent(oauthSession)`) only for `com.atproto.repo.*` writes and viewer-bound reads. This split avoids `401 Unauthorized` from non-bsky.social PDS deployments where the AppView-proxy contract is incomplete.
- **Login UX:** for existing-account login, start from the user's handle/DID and pass it through as the account identifier/login hint. Do not force a PDS picker unless the user explicitly entered a server hostname. SDKs with target APIs, such as `@atcute/oauth-node-client`, can express this as `authorize({ target: { type: 'account', identifier }, scope })`; reserve `{ type: 'pds', serviceUrl }` plus `prompt: 'create'` for signup.
