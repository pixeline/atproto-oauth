# OAuth Flow Reference (AT Protocol)

Use this as the canonical procedure for from-scratch implementations.

## 1) Inputs and Session State

Accept one login input:
- handle (`alice.example.com`)
- DID (`did:plc:...` or `did:web:...`)
- service URL/hostname (`https://pds.example.com`)

Persist per authorization attempt:
- generated `state`
- expected Authorization Server `issuer`
- PKCE `code_verifier`
- DPoP keypair
- app-local state (optional)

Persist per authenticated session:
- `sub` DID
- `iss` (Authorization Server issuer)
- `aud` (resolved PDS URL)
- granted `scope`
- `access_token`, `refresh_token`, expiry
- DPoP nonces (track separately for Authorization Server and Resource Server)

## 2) Identity and Discovery Chain

If input is a handle:
1. Resolve handle -> DID:
- DNS TXT `_atproto.<handle>` containing exactly one `did=<did>` value
- fallback `https://<handle>/.well-known/atproto-did`
2. Resolve DID -> DID document.
3. Enforce bidirectional verification: DID document must claim same handle in `alsoKnownAs` (`at://<handle>`).

If input is a DID:
1. Resolve DID -> DID document.

If input is a service URL/hostname:
1. Skip account binding at start.
2. Discover Authorization Server from service metadata directly.

DID resolution:
- `did:plc`: `GET https://plc.directory/<did>`
- `did:web`: `GET https://<host>/.well-known/did.json`

From DID document:
- Extract PDS service endpoint (`#atproto_pds`, type `AtprotoPersonalDataServer`)

From PDS:
- `GET https://<pds-host>/.well-known/oauth-protected-resource`
- read `authorization_servers` (atproto expects one Authorization Server)

From Authorization Server:
- `GET https://<issuer>/.well-known/oauth-authorization-server`
- validate metadata and required endpoints:
  - `authorization_endpoint`
  - `token_endpoint`
  - `pushed_authorization_request_endpoint`
- verify `issuer` origin matches metadata URL origin

### Discovery fetch safeguards

All discovery fetches above target untrusted remote servers. Generated code must enforce:
- **HTTPS only** (no `http://` redirects except localhost dev).
- **No cross-origin redirects** — reject any redirect that changes the domain or scheme of a well-known metadata URL.
- **Response size cap** (e.g., 512 KB) and **read timeout** (e.g., 10 s).
- **Content-Type `application/json`** required for JSON metadata.
- **SSRF prevention**: block private/internal IP ranges when resolving user-supplied handles or DIDs.
- **Schema validation**: validate JSON structure before extracting fields; reject malformed documents.
- **Strict field extraction**: only use documented fields; ignore unknown keys.

## 3) Authorization Request (PAR + PKCE + DPoP)

Generate per-attempt values:
- random `state` (unique, single-use)
- PKCE verifier/challenge (`S256` only)
- new DPoP keypair (per auth session)

Build PAR payload:
- `client_id`
- `redirect_uri`
- `response_type=code`
- `scope` (must include `atproto`)
- `code_challenge`
- `code_challenge_method=S256`
- `state`
- `login_hint` (when starting from handle or DID)
- confidential only:
  - `client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer`
  - `client_assertion=<signed JWT>`

Send:
- `POST <pushed_authorization_request_endpoint>`
- `Content-Type: application/x-www-form-urlencoded`
- include DPoP proof

Nonce behavior:
- first call may fail with `401` + `error=use_dpop_nonce`
- read `DPoP-Nonce`, rebuild DPoP proof, retry

On success:
- receive `request_uri`
- redirect browser to:
  - `<authorization_endpoint>?client_id=<client_id>&request_uri=<request_uri>`

## 4) Callback Processing

Expected callback query params:
- `state` (required)
- `code` (required unless error)
- `iss` (issuer; required when server indicates support)
- `error` (on failure)

Process strictly:
1. Load and consume state record exactly once (replay prevention).
2. Verify callback `iss` matches expected issuer in state record.
3. If `error` is present, fail flow and surface details.
4. Exchange `code` at token endpoint with:
- `grant_type=authorization_code`
- `code`
- `redirect_uri`
- `code_verifier`
- confidential client assertion fields when applicable
- DPoP proof (with current Authorization Server nonce)

## 5) Token Response Validation

Treat token response as untrusted until all checks pass.

Required checks:
- `token_type` is `DPoP`
- `sub` is a DID
- `scope` is present (required by atproto profile)
- `sub` matches expected account DID
  - if flow started with handle/DID: compare to expected DID
  - if flow started with service URL: resolve `sub` DID now and verify DID -> PDS -> Authorization Server chain
- Authorization Server issuer consistency remains valid

Then:
- persist tokens and session metadata
- expose only granted permissions to app features

## 6) Authorized API Requests

For PDS requests:
- `Authorization: DPoP <access_token>`
- `DPoP: <proof-jwt>` with `ath` claim bound to access token

If `use_dpop_nonce` is returned:
- read `DPoP-Nonce`
- retry once with new DPoP proof
- keep nonce cache updated per server

## 7) Refresh Flow

Refresh request:
- `POST <token_endpoint>`
- `grant_type=refresh_token`
- `refresh_token=<token>`
- DPoP proof
- confidential clients include client assertion

After refresh:
- validate response exactly like initial token exchange (`sub`, `scope`, `token_type`, issuer chain)
- update stored tokens atomically

## 8) Security-Critical Failure Conditions

Fail closed when any condition occurs:
- callback `state` missing/unknown/reused
- callback `iss` mismatch
- PAR unavailable or skipped
- PKCE method not `S256`
- `sub` mismatch or unverifiable
- Authorization Server chain mismatch
- token `scope` missing
- DPoP nonce replay or irrecoverable nonce errors

## 9) Cache Guidance for Auth Flows

Use aggressive freshness for login paths:
- identity cache lifetime `<= 10 minutes`
- avoid stale reads during active auth
- in browser runtimes, use DNS-over-HTTPS or resolver helper service for TXT lookups
