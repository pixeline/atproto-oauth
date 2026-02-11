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
import { Agent } from '@atproto/api'
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
    scope:
      'atproto rpc:app.bsky.actor.getProfile?aud=did:web:api.bsky.app#bsky_appview blob:*/* account:email transition:generic',
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

    const agent = new Agent(oauthSession)
    const profile = await agent.getProfile({ actor: agent.did })

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
    const agent = new Agent(oauthSession)
    const profile = await agent.getProfile({ actor: agent.did })

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

## 4) Notes

- This pattern is preferred over pure SPA token storage for long-lived sessions.
- TMB and Client Assertion Backend are alternatives, but BFF is the default recommendation.
- Keep token/session records in durable encrypted storage in production.
