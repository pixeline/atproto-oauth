# Client Metadata Reference

AT Protocol uses OAuth Client ID Metadata Documents. `client_id` is a URL to this JSON.

## Recommended Location and Naming

Preferred:
- `https://app.example.com/oauth-client-metadata.json`

Why this path:
- matches atproto conventional client_id style
- Authorization Server UIs can present a simpler host-first display for this convention

Do not use:
- non-HTTPS discoverable URLs (except localhost loopback dev exception)
- redirecting metadata URLs
- mismatched `client_id` value and fetch URL

## Required Metadata Fields

Include all of these:
- `client_id` (string)
- `grant_types` (array)
- `scope` (string)
- `response_types` (array)
- `redirect_uris` (array)
- `dpop_bound_access_tokens` (boolean, must be `true`)

AT Protocol profile requirements:
- `scope` must include `atproto`
- `grant_types` must include `authorization_code`
- `response_types` must include `code`

## Recommended Optional Fields

Provide when available:
- `client_name`
- `client_uri`
- `logo_uri`
- `tos_uri`
- `policy_uri`

Security note:
- treat display fields as untrusted unless client is explicitly trusted

## Client Authentication Modes

### Public clients

Use for SPA/mobile/CLI:
- `token_endpoint_auth_method: "none"`
- no client secret
- no private signing key on device

### Confidential clients

Use for server-side/BFF:
- `token_endpoint_auth_method: "private_key_jwt"`
- `token_endpoint_auth_signing_alg: "ES256"`
- include either `jwks` or `jwks_uri`

Client assertion JWT requirements:
- header: `alg=ES256`, `kid=<key-id>`
- body:
  - `iss=<client_id>`
  - `sub=<client_id>`
  - `aud=<authorization-server-issuer>`
  - `jti=<random unique id>`
  - `iat=<unix timestamp>`
- sign a fresh JWT per PAR/token request (no reuse)

## Redirect URI Rules

### Web apps

Use HTTPS redirect URIs hosted by app-controlled domains.

### Native apps

Allowed patterns:
- loopback IP redirect URI, e.g. `http://127.0.0.1:<port>/callback`
- private-use custom scheme, e.g. `com.example.app:/oauth/callback`
- app/universal links over HTTPS

Custom scheme rule (discoverable clients):
- scheme should match `client_id` hostname in reverse domain order
- example:
  - `client_id`: `https://app.example.com/oauth-client-metadata.json`
  - `redirect_uri`: `com.example.app:/oauth/callback`

## Localhost Development Exception

AT Protocol supports loopback development client IDs.

Pattern:
- `http://localhost?scope=<url-encoded-scope>&redirect_uri=<url-encoded-loopback-uri>`

Notes:
- use only for local development/testing
- redirect URI should use explicit loopback IP (`127.0.0.1` or `[::1]`) for callback listeners

## Validation Checklist

Validate metadata before starting auth:
- `client_id` exactly equals URL used to fetch metadata
- `dpop_bound_access_tokens` is `true`
- `scope` includes `atproto`
- `grant_types` contains `authorization_code`
- `response_types` contains `code`
- `redirect_uris` non-empty and valid for app type
- confidential clients have `private_key_jwt` + key material (`jwks` or `jwks_uri`)

## Minimal Public Example

```json
{
  "client_id": "https://app.example.com/oauth-client-metadata.json",
  "client_name": "Example SPA",
  "client_uri": "https://app.example.com",
  "redirect_uris": ["https://app.example.com/oauth/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "atproto rpc:app.bsky.actor.getProfile?aud=did:web:api.bsky.app#bsky_appview",
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "application_type": "web",
  "dpop_bound_access_tokens": true
}
```

## Minimal Confidential Example

```json
{
  "client_id": "https://app.example.com/oauth-client-metadata.json",
  "client_name": "Example BFF",
  "client_uri": "https://app.example.com",
  "redirect_uris": ["https://app.example.com/oauth/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "atproto transition:generic",
  "response_types": ["code"],
  "token_endpoint_auth_method": "private_key_jwt",
  "token_endpoint_auth_signing_alg": "ES256",
  "jwks_uri": "https://app.example.com/oauth/jwks.json",
  "application_type": "web",
  "dpop_bound_access_tokens": true
}
```
