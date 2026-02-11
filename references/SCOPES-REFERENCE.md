# Scopes Reference

Use least privilege. Always include `atproto`.

## Core Rule

Every atproto OAuth scope string must contain:
- `atproto`

Token response rule:
- `scope` must be present in token responses
- treat returned `scope` as source of truth for granted permissions

## Permission Families

AT Protocol permission scopes are resource/action oriented.

Common families:
- `account:*` (account attributes)
- `identity:*` (identity metadata)
- `repo:*` (repository record operations)
- `blob:*/*` (blob upload/read by MIME patterns)
- `rpc:<nsid>?aud=<did-web-service-id>` (method-level RPC access)

## Practical Scope Examples

Profile read from Bluesky AppView service:
- `rpc:app.bsky.actor.getProfile?aud=did:web:api.bsky.app#bsky_appview`

Media upload support:
- `blob:*/*`

Email access (user may decline):
- `account:email`

## Transitional Scopes (Still Supported)

These are supported for compatibility and migration scenarios.

- `transition:generic`
  - broad PDS permissions (similar intent to App Password-era broad access)
- `transition:chat.bsky`
  - chat/DM access
  - request with `transition:generic`
- `transition:email`
  - permits account email fields in session-oriented flows

Do not mark these as deprecated in implementation logic. Support them when needed.

## Scope String Guidance

- Use space-separated scope values
- Keep scopes stable and explicit; avoid accidental broadening
- For `rpc:` scopes:
  - set a precise NSID where possible
  - set `aud` to the expected service DID
- Avoid wildcard-heavy scopes unless product requirements demand them

## Request vs Granted

After token exchange:
1. Parse returned `scope`.
2. Compare with requested scope set.
3. Feature-gate by granted scopes only.
4. Handle partial grants gracefully (especially `account:email`).

## Recommended Bundles

### Basic sign-in + profile read
- `atproto`
- `rpc:app.bsky.actor.getProfile?aud=did:web:api.bsky.app#bsky_appview`

### Posting app
- `atproto`
- repo create/update scopes required by target collections
- optional `blob:*/*` for media

### DM-capable app (transitional)
- `atproto`
- `transition:generic`
- `transition:chat.bsky`

### Email-aware app
- `atproto`
- `account:email` or `transition:email` (depending on integration path)

## Security Notes

- Never assume scope grants from request intent; always verify token response `scope`
- Reject tokens missing `scope`
- Log scope mismatches for auditability
- Keep per-feature permission checks explicit in code paths
