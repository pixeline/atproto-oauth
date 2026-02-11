# Go BFF Example (`indigo` OAuth)

Use `github.com/bluesky-social/indigo/atproto/auth/oauth` for server-side confidential or public-BFF flows.

## 1) Dependencies

```bash
go get github.com/bluesky-social/indigo/atproto/auth/oauth
go get github.com/bluesky-social/indigo/atproto/identity
go get github.com/bluesky-social/indigo/atproto/atcrypto
go get github.com/gorilla/sessions
```

## 2) BFF server skeleton

```go
package main

import (
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
	"time"

	"github.com/bluesky-social/indigo/atproto/atcrypto"
	"github.com/bluesky-social/indigo/atproto/auth/oauth"
	"github.com/bluesky-social/indigo/atproto/identity"
	"github.com/gorilla/sessions"
)

type Server struct {
	CookieStore *sessions.CookieStore
	OAuth       *oauth.ClientApp
	Dir         identity.Directory
}

func main() {
	scopes := []string{
		"atproto",
		"rpc:app.bsky.actor.getProfile?aud=did:web:api.bsky.app#bsky_appview",
		"blob:*/*",
	}

	// Production discoverable client_id.
	cfg := oauth.NewPublicConfig(
		"https://app.example.com/oauth-client-metadata.json",
		"https://app.example.com/oauth/callback",
		scopes,
	)

	// Optional: upgrade to confidential client auth.
	// Private key must be P-256 multibase.
	if raw := ""; raw != "" {
		priv, err := atcrypto.ParsePrivateMultibase(raw)
		if err != nil {
			panic(err)
		}
		if err := cfg.SetClientSecret(priv, "primary"); err != nil {
			panic(err)
		}
	}

	// Use a durable ClientAuthStore implementation.
	// Reuse cookbook/go-oauth-web-app/sqlitestore.go as-is.
	store, err := NewSqliteStore(&SqliteStoreConfig{
		DatabasePath:              "oauth_sessions.sqlite3",
		SessionExpiryDuration:     90 * 24 * time.Hour,
		SessionInactivityDuration: 14 * 24 * time.Hour,
		AuthRequestExpiryDuration: 30 * time.Minute,
	})
	if err != nil {
		panic(err)
	}

	oauthClient := oauth.NewClientApp(&cfg, store)
	srv := &Server{
		CookieStore: sessions.NewCookieStore([]byte("replace-with-strong-secret")),
		OAuth:       oauthClient,
		Dir:         identity.DefaultDirectory(),
	}

	http.HandleFunc("GET /oauth-client-metadata.json", srv.ClientMetadata)
	http.HandleFunc("GET /oauth/jwks.json", srv.JWKS)
	http.HandleFunc("POST /oauth/login", srv.Login)
	http.HandleFunc("GET /oauth/callback", srv.Callback)

	slog.Info("listening", "addr", ":8080")
	_ = http.ListenAndServe(":8080", nil)
}

func (s *Server) ClientMetadata(w http.ResponseWriter, r *http.Request) {
	meta := s.OAuth.Config.ClientMetadata()
	if s.OAuth.Config.IsConfidential() {
		u := fmt.Sprintf("https://%s/oauth/jwks.json", r.Host)
		meta.JWKSURI = &u
	}
	meta.ClientName = ptr("Example Go BFF")
	meta.ClientURI = ptr(fmt.Sprintf("https://%s", r.Host))
	_ = meta.Validate(s.OAuth.Config.ClientID)
	_ = writeJSON(w, meta)
}

func (s *Server) JWKS(w http.ResponseWriter, _ *http.Request) {
	_ = writeJSON(w, s.OAuth.Config.PublicJWKS())
}

func (s *Server) Login(w http.ResponseWriter, r *http.Request) {
	if err := r.ParseForm(); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}
	identifier := r.FormValue("identifier")
	redirectURL, err := s.OAuth.StartAuthFlow(r.Context(), identifier)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}
	http.Redirect(w, r, redirectURL, http.StatusFound)
}

func (s *Server) Callback(w http.ResponseWriter, r *http.Request) {
	sessData, err := s.OAuth.ProcessCallback(r.Context(), r.URL.Query())
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	oauthSess, err := s.OAuth.ResumeSession(r.Context(), sessData.AccountDID, sessData.SessionID)
	if err != nil {
		http.Error(w, err.Error(), http.StatusUnauthorized)
		return
	}

	cookie, _ := s.CookieStore.Get(r, "example-oauth")
	cookie.Values["account_did"] = sessData.AccountDID.String()
	cookie.Values["session_id"] = sessData.SessionID
	_ = cookie.Save(r, w)

	_ = writeJSON(w, map[string]string{
		"did":   oauthSess.Data.AccountDID.String(),
		"scope": oauthSess.Data.Scope,
	})
}

func ptr(v string) *string { return &v }

func writeJSON(w http.ResponseWriter, v any) error {
	w.Header().Set("Content-Type", "application/json")
	return json.NewEncoder(w).Encode(v)
}
```

## 3) CLI pattern (public client)

For CLI apps, use loopback callback listeners and hosted metadata:
- start local callback server on random port
- set `CallbackURL` to `http://127.0.0.1:<port>/callback`
- call `StartAuthFlow()`
- open system browser
- call `ProcessCallback()` with callback query params

Reference implementation:
- `cookbook/go-oauth-cli-app/main.go`

## 4) Notes

- BFF is the preferred confidential deployment pattern.
- Keep `ClientAuthStore` durable and encrypted at rest.
- Keep identity cache short during login and validate issuer/sub chain on token processing.
