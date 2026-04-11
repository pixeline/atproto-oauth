# atproto-oauth

> **AI Agent Skill** — Implements AT Protocol (Bluesky) OAuth login correctly, including DID identity resolution, PAR/PKCE/DPoP flows, scope handling, and security-critical validation.
>
> ## When to use this skill
>
> Add this skill to your AI coding assistant when you need to:
>
> - Add **Bluesky / AT Protocol login** to a web app, SPA, mobile app, or CLI tool
> - - Implement `atproto` OAuth from scratch (PAR, PKCE S256, DPoP, client metadata)
>   - - Handle **DID identity resolution** and PDS / Authorization Server discovery
>     - - Configure AT Protocol **OAuth scopes** correctly
>       - - Troubleshoot atproto OAuth flows (nonce errors, callback failures, scope mismatches)
>        
>         - **Do not use this skill** for standard OAuth2/OIDC (Google, GitHub, etc.), Bluesky App Passwords, or any non-atproto login flow.
>        
>         - ## Example prompts that trigger this skill
>        
>         - - *"Add Bluesky login to my Next.js app"*
> - *"Implement AT Protocol OAuth with PKCE and DPoP in Go"*
> - - *"Why is my atproto OAuth callback failing with state mismatch?"*
>   - - *"What scopes do I need to read a user's Bluesky profile?"*
>    
>     - ## Install
>    
>     - ```bash
>       npx skills add pixeline/atproto-oauth
>       ```
>
> ## Compatibility
>
> Works with Cursor, Claude Code, Windsurf, Cline, Codex, and other tools that support the [skills.sh](https://agentskills.io/specification) format.
>
> ## What's included
>
> | File | Purpose |
> |---|---|
> | `SKILL.md` | Core skill instructions (always loaded) |
> | `examples/typescript-spa.md` | TypeScript SPA using `@atproto/oauth-client-browser` |
> | `examples/typescript-bff.md` | TypeScript BFF using `@atproto/oauth-client-node` |
> | `examples/go-bff.md` | Go BFF using `indigo/atproto/auth/oauth` |
> | `examples/python.md` | Python cookbook (no official SDK) |
> | `examples/client-metadata-examples.md` | Client metadata JSON examples |
> | `references/OAUTH-FLOW-REFERENCE.md` | Canonical step-by-step flow reference |
> | `references/CLIENT-METADATA-REFERENCE.md` | Client metadata field-by-field guide |
> | `references/SCOPES-REFERENCE.md` | Full AT Protocol scope taxonomy |
>
> ## Core References
>
> - [AT Protocol OAuth guide](https://atproto.com/guides/oauth)
> - - [AT Protocol OAuth spec](https://atproto.com/specs/oauth)
>   - - [AT Protocol Permissions spec](https://atproto.com/specs/permission)
>     - - [AT Protocol Identity guide](https://atproto.com/guides/identity)
>       - - [Advanced OAuth client implementation guide](https://docs.bsky.app/docs/advanced-guides/oauth-client)
>         - - [Agent Skills specification](https://agentskills.io/specification)
