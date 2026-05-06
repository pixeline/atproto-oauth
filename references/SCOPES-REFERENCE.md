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
- `repo:<nsid>?action=...` (repository record operations on a specific collection — see [Repo Scope Syntax](#repo-scope-syntax))
- `blob:*/*` (blob upload/read by MIME patterns)
- `rpc:<nsid>?aud=<did-web-service-id>` (method-level RPC access)

## Repo Scope Syntax

Granular write access is granted per collection NSID and per action verb.

Grammar:

```
repo:<nsid>?action=<verb>[&action=<verb>...]
```

Each `action=<verb>` is a **separate permission**. Multiple `&action=` clauses **accumulate** — they grant the union of the listed verbs on the named collection. Verbs are:

- `create` — write a new record
- `update` — modify an existing record (only meaningful for mutable record types)
- `delete` — remove a record

### Worked examples

Mutable record type (lists — records can be edited):
```
repo:app.bsky.graph.list?action=create&action=update&action=delete
```

Append-only record type (posts — records are immutable once written; `update` is unnecessary):
```
repo:app.bsky.feed.post?action=create&action=delete
```

List membership (listitems — typically created and removed, never edited):
```
repo:app.bsky.graph.listitem?action=create&action=delete
```

Likes / reposts (single-state interactions):
```
repo:app.bsky.feed.like?action=create&action=delete
repo:app.bsky.feed.repost?action=create&action=delete
```

### Composing multiple collections

Each collection needs its own scope entry. A list-management app touching both `list` and `listitem` requests:

```
repo:app.bsky.graph.list?action=create&action=update&action=delete repo:app.bsky.graph.listitem?action=create&action=delete
```

### When to include `update`

- Include `update` for mutable record types (e.g., `app.bsky.graph.list`, `app.bsky.actor.profile`).
- Omit `update` for append-only types (e.g., `app.bsky.feed.post`, `app.bsky.feed.like`, `app.bsky.feed.repost`). Adding it broadens scope without enabling any real capability.

### Why granular `repo:` scopes beat `transition:generic`

`transition:generic` grants the holder broad PDS-level access on the user's repo — equivalent in practical reach to legacy App Passwords. Granular `repo:<nsid>?action=...` scopes:

- Limit blast radius if a token leaks
- Surface the app's intent to the user during consent
- Force the developer to think through what the app actually writes
- Survive future tightening of `transition:*` semantics

Always prefer composed `repo:` scopes for new clients.

## Practical Scope Examples

Profile read from Bluesky AppView service:
- `rpc:app.bsky.actor.getProfile?aud=did:web:api.bsky.app#bsky_appview`

Media upload support:
- `blob:*/*`

Email access (user may decline):
- `account:email`

## Transitional Scopes (Still Supported)

These are supported for compatibility and migration scenarios.

> **⚠️ `transition:generic` is a legacy escape hatch, not a default.** It is functionally equivalent to App Password-era full repo access. Prefer granular `repo:<nsid>?action=...` scopes for new clients (see [Repo Scope Syntax](#repo-scope-syntax)). Only request `transition:generic` when the app genuinely needs broad PDS access and that need is documented.

- `transition:generic`
  - broad PDS permissions (similar intent to App Password-era broad access)
- `transition:chat.bsky`
  - chat/DM access
  - request with `transition:generic`
- `transition:email`
  - permits account email fields in session-oriented flows

Do not mark these as deprecated in implementation logic. Support them when needed — but do not reach for `transition:generic` as a shortcut to skip the scope inventory.

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
- `repo:app.bsky.feed.post?action=create&action=delete`
- optional `repo:app.bsky.feed.like?action=create&action=delete`
- optional `repo:app.bsky.feed.repost?action=create&action=delete`
- optional `blob:*/*` for media

### List management app (BLM-style)
- `atproto`
- `repo:app.bsky.graph.list?action=create&action=update&action=delete`
- `repo:app.bsky.graph.listitem?action=create&action=delete`
- (reads of profiles/lists/list members go through the public AppView at `https://public.api.bsky.app` — no scope needed)

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
