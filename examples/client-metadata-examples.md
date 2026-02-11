# `oauth-client-metadata.json` Examples

Use these as templates and adjust scopes/redirects for your app.

## 1) Public Web SPA

```json
{
  "client_id": "https://app.example.com/oauth-client-metadata.json",
  "client_name": "Example SPA",
  "client_uri": "https://app.example.com",
  "logo_uri": "https://app.example.com/logo.png",
  "tos_uri": "https://app.example.com/tos",
  "policy_uri": "https://app.example.com/privacy",
  "redirect_uris": ["https://app.example.com/oauth/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "atproto rpc:app.bsky.actor.getProfile?aud=did:web:api.bsky.app#bsky_appview blob:*/*",
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "application_type": "web",
  "dpop_bound_access_tokens": true
}
```

## 2) Confidential BFF (Recommended)

```json
{
  "client_id": "https://app.example.com/oauth-client-metadata.json",
  "client_name": "Example BFF",
  "client_uri": "https://app.example.com",
  "logo_uri": "https://app.example.com/logo.png",
  "tos_uri": "https://app.example.com/tos",
  "policy_uri": "https://app.example.com/privacy",
  "redirect_uris": ["https://app.example.com/oauth/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "atproto transition:generic transition:chat.bsky account:email",
  "response_types": ["code"],
  "token_endpoint_auth_method": "private_key_jwt",
  "token_endpoint_auth_signing_alg": "ES256",
  "jwks_uri": "https://app.example.com/oauth/jwks.json",
  "application_type": "web",
  "dpop_bound_access_tokens": true
}
```

## 3) Native App (Custom URI Scheme)

```json
{
  "client_id": "https://mobile.example.com/oauth-client-metadata.json",
  "client_name": "Example Native App",
  "client_uri": "https://mobile.example.com",
  "redirect_uris": [
    "com.example.mobile:/oauth/callback",
    "https://mobile.example.com/oauth/callback"
  ],
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "atproto rpc:app.bsky.actor.getProfile?aud=did:web:api.bsky.app#bsky_appview",
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "application_type": "native",
  "dpop_bound_access_tokens": true
}
```

## 4) Localhost Development Client ID

Loopback development client_id format:

```text
http://localhost?scope=atproto%20rpc%3Aapp.bsky.actor.getProfile%3Faud%3Ddid%3Aweb%3Aapi.bsky.app%23bsky_appview&redirect_uri=http%3A%2F%2F127.0.0.1%3A5173%2Foauth%2Fcallback
```

The metadata represented by that loopback client_id is equivalent to:

```json
{
  "client_id": "http://localhost?scope=atproto%20rpc%3Aapp.bsky.actor.getProfile%3Faud%3Ddid%3Aweb%3Aapi.bsky.app%23bsky_appview&redirect_uri=http%3A%2F%2F127.0.0.1%3A5173%2Foauth%2Fcallback",
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "atproto rpc:app.bsky.actor.getProfile?aud=did:web:api.bsky.app#bsky_appview",
  "response_types": ["code"],
  "redirect_uris": ["http://127.0.0.1:5173/oauth/callback"],
  "token_endpoint_auth_method": "none",
  "application_type": "native",
  "dpop_bound_access_tokens": true
}
```

## Validation Quick Check

- `client_id` must match the exact URL used to fetch metadata.
- `dpop_bound_access_tokens` must be `true`.
- `scope` must include `atproto`.
- `response_types` must include `code`.
- `grant_types` must include `authorization_code`.
- For confidential clients: include `private_key_jwt` + `token_endpoint_auth_signing_alg` + `jwks`/`jwks_uri`.
