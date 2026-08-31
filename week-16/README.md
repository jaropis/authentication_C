# Week 16: OAuth 2.0 Authorization Code with PKCE

## Goal

Act as an OAuth client, obtain delegated access through Authorization Code Flow with PKCE, and call one harmless resource API. Do not mistake the access token for proof that a user authenticated to authlab.

Time budget: 7-10 hours, including provider registration.

## Start here

Begin with the completed Week 15 application and token lab:

```bash
mkdir -p week-16/notes week-16/var
cp -R week-15/src week-15/tests week-15/tools week-15/Makefile week-15/schema.sql week-15/Caddyfile week-16/
make -C week-16 clean all test jwt-lab
```

Your starting authlab has local password/session login and a JWT learning tool. OAuth adds delegated authorization; it does not replace either merely because tokens appear in the flow.

Install HTTP and JSON client libraries:

```bash
brew install curl jansson pkg-config
pkg-config --modversion libcurl jansson
```

Choose either a standards-compliant local mock for automated tests or a real provider that permits a localhost HTTPS redirect. Register this exact redirect URI when possible:

```text
https://localhost:8443/oauth/callback
```

Never put a provider client secret in source, shell scripts, notes, or Git. If one is required, load it from an environment variable entered outside recorded commands.

## Name all four roles

In `notes/oauth-flow.md`, identify for your chosen provider:

- resource owner: the person granting access;
- client: authlab;
- authorization server: issues code/token after grant;
- resource server: accepts the access token for a scoped API.

Draw every front-channel browser redirect and back-channel HTTPS request. Annotate which party can read each value.

## Milestone 1: Configure provider metadata

Load explicit startup configuration:

```text
OAUTH_AUTHORIZATION_ENDPOINT
OAUTH_TOKEN_ENDPOINT
OAUTH_RESOURCE_ENDPOINT
OAUTH_CLIENT_ID
OAUTH_REDIRECT_URI
OAUTH_SCOPE
```

Require HTTPS endpoints except an explicitly enabled loopback mock. Do not take endpoint URLs from an incoming request. Keep scope minimal and choose a read-only resource for the lab.

## Milestone 2: Create a one-use transaction

Add an `oauth_transactions` table or short-lived server-side store containing:

```text
transaction_id, state_hash, pkce_verifier, created_at, expires_at, used_at
```

For `GET /oauth/start`:

1. Generate an opaque transaction ID and independent `state` value with `randombytes_buf`.
2. Generate 32 random verifier bytes and Base64URL-encode without padding; the result is a valid 43-character PKCE verifier.
3. Compute `SHA256(ASCII(code_verifier))` with libsodium and Base64URL-encode it as `code_challenge`.
4. Store only what the callback needs; storing a digest of `state` limits disclosure.
5. Set a short-lived `oauth_tx` cookie with `Secure; HttpOnly; SameSite=Lax; Path=/oauth`.
6. Redirect to the fixed authorization endpoint with percent-encoded `response_type=code`, client ID, exact redirect URI, scope, state, challenge, and `code_challenge_method=S256`.

Use libcurl's URL API or another structured URL builder. Do not concatenate unescaped query values.

The transaction lifetime should be a few minutes and it must be consumed once.

## Milestone 3: Validate the callback

Implement `GET /oauth/callback`:

1. Handle provider `error` parameters without reflecting them unescaped.
2. Require exactly one bounded `code` and `state`.
3. Resolve the `oauth_tx` cookie to one unexpired, unused transaction.
4. Hash and constant-time compare returned state.
5. Mark/consume the transaction so callback replay fails.
6. Send the authorization code to the token endpoint over verified HTTPS with the original verifier, client ID, and exact redirect URI.

If a confidential client secret is required, send it only according to provider metadata and keep it out of logs. Never put access tokens in URLs.

Use libcurl with certificate verification enabled, response-size/time limits, redirect handling disabled or tightly constrained, and a bounded write callback. Parse successful JSON with Jansson. Treat malformed or unexpected token responses as failure.

## Milestone 4: Use delegated access

Keep the access token in short-lived server-side state for the lab. Call the configured resource endpoint using:

```http
Authorization: Bearer <access-token>
```

Request one read-only resource covered by the exact scope. Display a minimal, HTML-escaped subset of the result. Do not dump the token response, bearer token, refresh token, or arbitrary provider JSON into logs/pages.

If the provider issues a refresh token, document its greater lifetime and storage/revocation requirements but do not persist it until you have designed those controls.

Critically, do not create an authlab user/session based solely on the access token or resource response. The access token's audience is normally the resource server, and OAuth itself does not standardize a client-facing identity assertion. Week 17 adds OIDC for that.

## Attack and failure exercises

Use the mock provider to make each case deterministic:

- callback with missing/wrong state;
- correct state paired with the wrong transaction cookie;
- callback replay after transaction consumption;
- code exchange with the wrong verifier;
- redirect URI differing by one character;
- token endpoint with invalid TLS or oversized/malformed JSON;
- access token with insufficient scope rejected by the resource server;
- access token pasted into authlab's session cookie rejected as meaningless.

Record which component rejects each case and which security property is preserved.

## Automated and live checks

Build a local mock or use a maintained provider test fixture for CI-style checks. It should record no real secrets and verify the code challenge against the verifier. Test state expiry and one-use behavior with an injected clock.

Then run one live flow through Caddy if provider registration is available. In browser developer tools, follow redirects but redact codes/state from notes. Confirm the code travels through the browser while token exchange is server-to-server.

## End-of-week working state

At the end of Week 16, `/oauth/start` creates a bounded one-use state/PKCE transaction, `/oauth/callback` validates it and exchanges the code over verified HTTPS, and authlab can call one minimally scoped resource API. OAuth completion does not create a local authenticated session.

Verify against the local mock:

```bash
make -C week-16 clean all test
sh week-16/tests/test_oauth_flow.sh
```

The test must prove wrong state, wrong verifier, and callback replay all fail. Be able to explain exactly what authority was delegated and why the resulting access token is not automatically an identity credential for authlab. Copy this state into Week 17.
