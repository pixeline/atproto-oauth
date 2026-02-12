---
name: atproto-oauth
description: Teaches an AI agent to implement AT Protocol (atproto/Bluesky) OAuth login and authentication flows correctly. Use for atproto, bluesky, oauth, login, authentication, DID identity resolution, client metadata, PKCE/PAR/DPoP, PDS and Authorization Server discovery, scope design, and secure token/session validation.
---

# AT Protocol OAuth Implementation

Implement atproto OAuth using the atproto profile requirements, not generic OAuth defaults.

## Decide App Pattern

Ask: What type of app are you building?

- `Web service (recommended)`: implement a confidential client with Backend For Frontend (BFF).
  Use `token_endpoint_auth_method: private_key_jwt`, JWT client assertions, and server-side token storage.
- `Browser SPA / mobile / CLI`: implement a public client with `token_endpoint_auth_method: none`.
  Use shorter sessions and no client secret on-device.
- `Alternative confidential patterns`: Token-Mediating Backend (TMB) and Client Assertion Backend exist, but prefer BFF unless an explicit requirement says otherwise.

Reference:
- Flow details: `references/OAUTH-FLOW-REFERENCE.md`
- Metadata schema: `references/CLIENT-METADATA-REFERENCE.md`
- Scope taxonomy: `references/SCOPES-REFERENCE.md`

## Implement Discovery Chain

Start from one of: handle, DID, or service URL.

1. Resolve identity.
- Handle -> DID:
  - query DNS TXT `_atproto.<handle>` and parse single `did=<did>` record
  - fallback to `https://<handle>/.well-known/atproto-did`
- DID -> DID document:
  - `did:plc`: fetch from `https://plc.directory/<did>`
  - `did:web`: fetch `https://<host>/.well-known/did.json` (atproto supports hostname-level did:web)
- Enforce bidirectional handle verification:
  - if login started with handle, DID document must claim the same handle in `alsoKnownAs` (`at://<handle>`)

2. Resolve service + Authorization Server.
- Extract PDS endpoint from DID document service entry `#atproto_pds`
- Fetch PDS protected resource metadata:
  - `https://<pds-host>/.well-known/oauth-protected-resource`
- Read `authorization_servers` and resolve Authorization Server metadata:
  - `https://<issuer>/.well-known/oauth-authorization-server`
- Verify `issuer` matches the Authorization Server origin used

3. Apply cache policy for auth flows.
- Keep handle and DID cache lifetimes short (`<= 10 minutes`)
- Disable stale identity data during active login (`allowStale: false` equivalent)
- In browser environments, use DNS-over-HTTPS or a resolver service for TXT lookups

## Build Client Metadata

Host metadata at a stable HTTPS URL, preferably:

- `https://app.example.com/oauth-client-metadata.json`

Use `oauth-client-metadata.json` naming to get cleaner domain display in Authorization Server UI.

Required fields:
- `client_id`
- `grant_types`
- `scope`
- `response_types`
- `redirect_uris`
- `dpop_bound_access_tokens: true`

Recommended optional fields:
- `client_name`
- `client_uri`
- `logo_uri`
- `tos_uri`
- `policy_uri`

Critical rules:
- `client_id` must exactly equal the URL used to fetch metadata
- Always include `atproto` in `scope`
- Always include `authorization_code` in `grant_types`
- Always include `code` in `response_types`
- Set `token_endpoint_auth_method`:
  - `none` for public clients
  - `private_key_jwt` for confidential clients
- For confidential clients, include `jwks` or `jwks_uri` and sign assertions with `ES256`

Local development exception:
- Loopback client_id is allowed via `http://localhost` form (query params carry `scope` and `redirect_uri`)
- Use only for local development and testing

See complete field-by-field guidance in `references/CLIENT-METADATA-REFERENCE.md`.

## Run Authorization Code Flow (Atproto Profile)

Always use:
- Authorization Code
- PKCE with `S256` only
- PAR
- DPoP with server-issued nonces

1. Start authorization session.
- Generate new random `state` per login attempt
- Generate new PKCE verifier/challenge per login attempt
- Generate new per-session DPoP keypair
- Persist `{state -> expected issuer, verifier, dpop key, app state}`

2. Submit PAR request to `pushed_authorization_request_endpoint`.
- Send form-encoded fields including:
  - `client_id`
  - `redirect_uri`
  - `response_type=code`
  - `scope`
  - `code_challenge`
  - `code_challenge_method=S256`
  - `state`
  - `login_hint` (user-entered handle or DID when available)
- Include DPoP proof
- For confidential clients, include `client_assertion_type` and `client_assertion`
- On `use_dpop_nonce`, retry with updated nonce from `DPoP-Nonce`

3. Redirect browser to `authorization_endpoint` with:
- `client_id`
- `request_uri` (from PAR response)

4. Process callback.
- Require `state`
- Verify state exists, is unique, and consume it once
- If `iss` is present, verify it matches expected Authorization Server `issuer`
- Exchange `code` at `token_endpoint` with PKCE verifier + DPoP (+ client assertion for confidential clients)

5. Validate token response before trusting identity.
- Require `token_type=DPoP`
- Require `scope` field and parse granted scopes
- Verify `sub` is a DID and matches expected identity
- If login started from server hostname, resolve DID from `sub` and verify DID -> PDS -> Authorization Server chain

## Handle Scopes Correctly

Always request `atproto`.

Common granular scopes:
- `rpc:app.bsky.actor.getProfile?aud=did:web:api.bsky.app#bsky_appview`
- `blob:*/*`
- `account:email` (optional grant; user may decline)

Transitional scopes (still supported):
- `transition:generic`
- `transition:chat.bsky` (request with `transition:generic`)
- `transition:email`

After token exchange:
- Compare granted scopes in token response to requested scopes
- Do not assume optional scopes were granted
- Gate features at runtime on granted scopes only

Full scope taxonomy is in `references/SCOPES-REFERENCE.md`.

## Handle PDS Backend Variations

Support heterogeneous deployments:
- Official PDS (`ghcr.io/bluesky-social/pds`)
- Goat toolkit based deployments
- Cocoon and other third-party PDS implementations

Do not hardcode topology assumptions.
- A PDS may delegate auth to a separate Authorization Server (entryway pattern)
- Always perform discovery at runtime from well-known metadata endpoints
- Always validate issuer and protected resource relationships

## Use Language-Specific Patterns

- TypeScript SPA:
  - `examples/typescript-spa.md`
  - package: `@atproto/oauth-client-browser`
- TypeScript BFF:
  - `examples/typescript-bff.md`
  - package: `@atproto/oauth-client-node`
- Go BFF:
  - `examples/go-bff.md`
  - package: `github.com/bluesky-social/indigo/atproto/auth/oauth`
- Python:
  - no official SDK; follow cookbook style flow and implement protocol primitives from `references/OAUTH-FLOW-REFERENCE.md`

## Security Checklist

Complete all checks before shipping:

- [ ] Enforce PKCE `S256`; never allow `plain`
- [ ] Use PAR for all authorization requests
- [ ] Use DPoP on PAR, token, and resource requests
- [ ] Rotate and persist DPoP nonces per server (Authorization Server vs Resource Server)
- [ ] Generate unique `state` and PKCE verifier per auth attempt
- [ ] Verify callback `iss` against expected issuer
- [ ] Verify token `sub` DID matches expected account
- [ ] Verify DID -> PDS -> Authorization Server consistency before trusting tokens
- [ ] Verify handle bidirectionally when login starts from handle
- [ ] Require and inspect token response `scope`; feature-gate on granted scopes
- [ ] Treat unverified metadata display fields (`client_name`, `logo_uri`) as untrusted unless client is explicitly trusted
- [ ] Keep identity caches short for auth (`<= 10 minutes`), avoid stale reads during active login

## Implementation Pitfalls and Production Guardrails

These are common failure modes seen in real deployments and how to avoid them.

### 1) Treat OAuth storage adapters as protocol-critical

State/session persistence bugs often surface as OAuth callback failures (not storage errors).

- Normalize KV reads to support both:
  - JSON strings
  - already-deserialized JSON objects
- Verify read-after-write for:
  - state store
  - session store
  - one-time app state mappings (e.g., `returnTo`)
- Ensure callback can read the exact key format written by authorize flow.
- Add an integration test for your chosen backend (Redis/KV) that covers:
  - `set -> get -> del` for OAuth state and session.

### 2) Parse granted scopes from the SDK object, not guessed fields

`client.callback()` typically returns an OAuth session object, not raw token payload.

- Prefer SDK methods (e.g., token info accessor) to obtain granted scopes.
- Do not assume `session.scope` / `session.tokenSet.scope` exists on all SDK/session types.
- Feature-gate from the final granted scope set only.

### 3) Use explicit callback error mapping

Not all callback errors should be 500.

- Map unknown/expired authorization session to a clear client error:
  - `400` + “Login session expired or invalid. Please retry.”
- Keep `500` for true server failures.
- Log the internal reason for operators; return user-safe text to clients.

### 4) Validate runtime env from server execution path

OAuth URL correctness can silently break in production if env vars differ between client and server contexts.

- Confirm effective values at runtime for:
  - app base URL
  - redirect URI
  - client metadata URL / client_id
- Prefer server-safe env variables for server-side OAuth configuration.
- Verify production metadata endpoint reflects production URLs.

### 5) Keep profile/identity display resilient and scope-minimal

Requesting broader scopes is often unnecessary for basic identity display.

- Keep baseline scope minimal (`atproto`) unless extra capabilities are required.
- For display-only identity fields (handle/avatar), use layered resolution:
  1. authenticated profile fetch when available
  2. public appview/profile endpoint fallback
  3. DID document (`alsoKnownAs`) fallback for handle
- Never block login solely because non-critical profile fields are unavailable.

### 6) Add operational smoke tests to release checklist

After each auth-related deployment, verify end-to-end in production:

- `/oauth-client-metadata.json` returns expected `client_id`, `redirect_uris`, `scope`.
- `/api/oauth/login` returns `302` to auth server with PAR `request_uri`.
- Fresh login callback succeeds and creates app session.
- Expired callback returns controlled `400` (not `500`).
- Header identity rendering uses handle/avatar when resolvable.
