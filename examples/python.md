# AT Protocol OAuth — Python Cookbook

There is no official AT Protocol OAuth SDK for Python. Use this cookbook-style guide to implement the protocol primitives directly. For the authoritative flow specification, see `references/OAUTH-FLOW-REFERENCE.md`.

## Dependencies

```
pip install httpx cryptography PyJWT
```

- `httpx` — async HTTP client
- - `cryptography` — EC key generation (ES256 for DPoP and client assertions)
  - - `PyJWT` — JWT encoding/decoding
   
    - ## 1. Identity Resolution
   
    - ```python
      import httpx

      async def resolve_handle_to_did(handle: str) -> str:
          """Resolve an atproto handle to a DID. Tries DNS TXT first, then HTTPS fallback."""
          # DNS-over-HTTPS fallback (browser-safe and recommended for server use)
          async with httpx.AsyncClient() as client:
              r = await client.get(
                  "https://cloudflare-dns.com/dns-query",
                  params={"name": f"_atproto.{handle}", "type": "TXT"},
                  headers={"Accept": "application/dns-json"},
              )
              r.raise_for_status()
              for answer in r.json().get("Answer", []):
                  data = answer.get("data", "").strip('"')
                  if data.startswith("did="):
                      return data[4:]

          # HTTPS fallback
          async with httpx.AsyncClient() as client:
              r = await client.get(f"https://{handle}/.well-known/atproto-did")
              r.raise_for_status()
              return r.text.strip()


      async def resolve_did_document(did: str) -> dict:
          """Fetch a DID document for did:plc or did:web."""
          async with httpx.AsyncClient() as client:
              if did.startswith("did:plc:"):
                  r = await client.get(f"https://plc.directory/{did}")
              elif did.startswith("did:web:"):
                  host = did.removeprefix("did:web:")
                  r = await client.get(f"https://{host}/.well-known/did.json")
              else:
                  raise ValueError(f"Unsupported DID method: {did}")
              r.raise_for_status()
              return r.json()


      def extract_pds_endpoint(did_doc: dict) -> str:
          """Extract the PDS service endpoint from a DID document."""
          for service in did_doc.get("service", []):
              if service.get("id") == "#atproto_pds":
                  return service["serviceEndpoint"]
          raise ValueError("No #atproto_pds service found in DID document")


      async def discover_authorization_server(pds_url: str) -> dict:
          """Discover Authorization Server metadata from a PDS endpoint."""
          async with httpx.AsyncClient() as client:
              # Step 1: fetch protected resource metadata
              r = await client.get(f"{pds_url}/.well-known/oauth-protected-resource")
              r.raise_for_status()
              pr_meta = r.json()
              auth_servers = pr_meta.get("authorization_servers", [])
              if not auth_servers:
                  raise ValueError("No authorization_servers in protected resource metadata")
              issuer = auth_servers[0]

              # Step 2: fetch Authorization Server metadata
              r = await client.get(f"{issuer}/.well-known/oauth-authorization-server")
              r.raise_for_status()
              as_meta = r.json()

              # Verify issuer
              if as_meta.get("issuer") != issuer:
                  raise ValueError("Authorization Server issuer mismatch")
              return as_meta
      ```

      ## 2. PKCE and State Generation

      ```python
      import os
      import hashlib
      import base64

      def generate_pkce_pair() -> tuple[str, str]:
          """Return (code_verifier, code_challenge) using S256."""
          verifier = base64.urlsafe_b64encode(os.urandom(32)).rstrip(b"=").decode()
          digest = hashlib.sha256(verifier.encode()).digest()
          challenge = base64.urlsafe_b64encode(digest).rstrip(b"=").decode()
          return verifier, challenge

      def generate_state() -> str:
          return base64.urlsafe_b64encode(os.urandom(16)).rstrip(b"=").decode()
      ```

      ## 3. DPoP Key and Proof Generation

      ```python
      import time
      import jwt
      from cryptography.hazmat.primitives.asymmetric.ec import generate_private_key, SECP256K1
      from cryptography.hazmat.primitives.asymmetric.ec import EllipticCurvePrivateKey
      from cryptography.hazmat.backends import default_backend
      from cryptography.hazmat.primitives import serialization

      def generate_dpop_keypair() -> EllipticCurvePrivateKey:
          return generate_private_key(SECP256K1(), default_backend())

      def make_dpop_proof(
          private_key: EllipticCurvePrivateKey,
          htm: str,         # HTTP method (e.g., "POST")
          htu: str,         # HTTP URI
          nonce: str | None = None,
          ath: str | None = None,  # access token hash for resource requests
      ) -> str:
          """Create a DPoP proof JWT."""
          public_key = private_key.public_key()
          pub_numbers = public_key.public_numbers()

          def b64url_int(n: int) -> str:
              length = (n.bit_length() + 7) // 8
              return base64.urlsafe_b64encode(n.to_bytes(length, "big")).rstrip(b"=").decode()

          jwk = {
              "kty": "EC",
              "crv": "P-256",
              "x": b64url_int(pub_numbers.x),
              "y": b64url_int(pub_numbers.y),
          }
          header = {"alg": "ES256", "typ": "dpop+jwt", "jwk": jwk}
          payload = {
              "jti": base64.urlsafe_b64encode(os.urandom(16)).decode(),
              "htm": htm,
              "htu": htu,
              "iat": int(time.time()),
          }
          if nonce:
              payload["nonce"] = nonce
          if ath:
              payload["ath"] = ath

          pem = private_key.private_bytes(
              serialization.Encoding.PEM,
              serialization.PrivateFormat.TraditionalOpenSSL,
              serialization.NoEncryption(),
          )
          return jwt.encode(payload, pem, algorithm="ES256", headers=header)
      ```

      ## 4. PAR Request

      ```python
      async def send_par_request(
          par_endpoint: str,
          client_id: str,
          redirect_uri: str,
          scope: str,
          code_challenge: str,
          state: str,
          dpop_key: EllipticCurvePrivateKey,
          login_hint: str | None = None,
          dpop_nonce: str | None = None,
      ) -> str:
          """Submit a PAR request and return the request_uri. Retries once on use_dpop_nonce."""
          payload = {
              "client_id": client_id,
              "redirect_uri": redirect_uri,
              "response_type": "code",
              "scope": scope,
              "code_challenge": code_challenge,
              "code_challenge_method": "S256",
              "state": state,
          }
          if login_hint:
              payload["login_hint"] = login_hint

          async def attempt(nonce: str | None) -> httpx.Response:
              proof = make_dpop_proof(dpop_key, "POST", par_endpoint, nonce=nonce)
              async with httpx.AsyncClient() as client:
                  return await client.post(
                      par_endpoint,
                      data=payload,
                      headers={"DPoP": proof},
                  )

          r = await attempt(dpop_nonce)
          if r.status_code == 401 and r.json().get("error") == "use_dpop_nonce":
              nonce = r.headers.get("DPoP-Nonce")
              r = await attempt(nonce)
          r.raise_for_status()
          return r.json()["request_uri"]
      ```

      ## 5. Token Exchange

      ```python
      async def exchange_code(
          token_endpoint: str,
          client_id: str,
          redirect_uri: str,
          code: str,
          code_verifier: str,
          dpop_key: EllipticCurvePrivateKey,
          dpop_nonce: str | None = None,
      ) -> dict:
          """Exchange an authorization code for tokens. Retries once on use_dpop_nonce."""
          payload = {
              "grant_type": "authorization_code",
              "code": code,
              "redirect_uri": redirect_uri,
              "client_id": client_id,
              "code_verifier": code_verifier,
          }

          async def attempt(nonce: str | None) -> httpx.Response:
              proof = make_dpop_proof(dpop_key, "POST", token_endpoint, nonce=nonce)
              async with httpx.AsyncClient() as client:
                  return await client.post(
                      token_endpoint,
                      data=payload,
                      headers={"DPoP": proof},
                  )

          r = await attempt(dpop_nonce)
          if r.status_code == 401 and r.json().get("error") == "use_dpop_nonce":
              nonce = r.headers.get("DPoP-Nonce")
              r = await attempt(nonce)
          r.raise_for_status()
          return r.json()
      ```

      ## 6. Token Validation

      ```python
      def validate_token_response(token: dict, expected_did: str | None = None) -> None:
          """
          Validate a token response before trusting it.
          Raises ValueError on any failed check.
          """
          if token.get("token_type", "").lower() != "dpop":
              raise ValueError("token_type must be DPoP")
          if "scope" not in token:
              raise ValueError("scope field missing from token response")
          sub = token.get("sub", "")
          if not (sub.startswith("did:plc:") or sub.startswith("did:web:")):
              raise ValueError(f"sub is not a valid DID: {sub!r}")
          if expected_did and sub != expected_did:
              raise ValueError(f"sub mismatch: expected {expected_did}, got {sub}")
      ```

      ## 7. Full Login Flow (Outline)

      ```python
      async def start_login(handle: str, client_id: str, redirect_uri: str, as_meta: dict) -> dict:
          """
          Begin an OAuth login. Returns session state to persist until callback.
          """
          did = await resolve_handle_to_did(handle)
          did_doc = await resolve_did_document(did)

          # Bidirectional handle verification
          also_known_as = did_doc.get("alsoKnownAs", [])
          if f"at://{handle}" not in also_known_as:
              raise ValueError(f"Handle {handle} not verified in DID document")

          verifier, challenge = generate_pkce_pair()
          state = generate_state()
          dpop_key = generate_dpop_keypair()

          request_uri = await send_par_request(
              par_endpoint=as_meta["pushed_authorization_request_endpoint"],
              client_id=client_id,
              redirect_uri=redirect_uri,
              scope="atproto",
              code_challenge=challenge,
              state=state,
              dpop_key=dpop_key,
              login_hint=handle,
          )

          auth_url = (
              f"{as_meta['authorization_endpoint']}"
              f"?client_id={client_id}&request_uri={request_uri}"
          )

          # Persist this in your session/KV store, keyed by state
          return {
              "state": state,
              "expected_did": did,
              "expected_issuer": as_meta["issuer"],
              "code_verifier": verifier,
              "dpop_key": dpop_key,  # serialize for storage
              "auth_url": auth_url,
          }


      async def handle_callback(
          code: str,
          state: str,
          iss: str | None,
          stored_session: dict,
          token_endpoint: str,
          client_id: str,
          redirect_uri: str,
      ) -> dict:
          """
          Process the OAuth callback. stored_session is what was returned by start_login.
          """
          if state != stored_session["state"]:
              raise ValueError("State mismatch")
          if iss and iss != stored_session["expected_issuer"]:
              raise ValueError("Issuer mismatch in callback")

          token = await exchange_code(
              token_endpoint=token_endpoint,
              client_id=client_id,
              redirect_uri=redirect_uri,
              code=code,
              code_verifier=stored_session["code_verifier"],
              dpop_key=stored_session["dpop_key"],
          )

          validate_token_response(token, expected_did=stored_session["expected_did"])
          return token
      ```

      ## Notes

      - **No official SDK**: unlike TypeScript or Go, Python has no `@atproto/oauth-client` equivalent. Maintain your DPoP nonce cache per server (Authorization Server and Resource Server separately).
      - - **Key serialization**: DPoP keypairs must be serialized (e.g., to PEM or JWK) before storing in Redis or a session backend.
        - - **Refresh flow**: repeat the token exchange pattern with `grant_type=refresh_token` and a fresh DPoP proof. Re-validate the response just like the initial exchange.
          - - **Scope gating**: always read granted scopes from the `scope` field of the token response, never assume optional scopes were granted.
