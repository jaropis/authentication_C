# Week 17: OpenID Connect

## Goal

Add the OIDC identity layer to the authorization-code flow: discover provider metadata, validate an ID token and nonce completely, map `(issuer, subject)` to a local user, then create an ordinary authlab session.

Time budget: 8-12 hours. This is the most integration-heavy week.

## Start here

Begin with the completed Week 16 OAuth client:

```bash
mkdir -p week-17/notes week-17/var
cp -R week-16/src week-16/tests week-16/tools week-16/Makefile week-16/schema.sql week-16/Caddyfile week-17/
make -C week-17 clean all test
```

Your starting code validates state and PKCE and can use an access token at a resource server, but deliberately does not log anyone into authlab. Use an OpenID Connect provider that supports discovery, Authorization Code Flow, PKCE, and your localhost HTTPS redirect.

Use a mature JOSE library for JWS/JWK verification. One concrete macOS stack is the Latchset José C library:

```bash
brew install curl jansson jose pkg-config
pkg-config --modversion libcurl jansson jose
```

Pin and document actual library versions. If Latchset José is unavailable in your package manager, choose another maintained C JOSE library with JWK/JWS support and update the build/API instructions explicitly; do not hand-roll RSA/ECDSA verification or JSON canonicalization.

## Milestone 1: Discover and pin the issuer

Configure one exact expected issuer and fetch:

```text
<issuer>/.well-known/openid-configuration
```

Using bounded libcurl/Jansson code, require:

- HTTPS except an explicit loopback mock;
- metadata `issuer` exactly equals configured issuer;
- authorization, token, and `jwks_uri` endpoints use acceptable HTTPS URLs;
- `S256` PKCE and `code` response type are supported;
- an expected ID-token signing algorithm is advertised.

Do not let callback parameters select an issuer or metadata URL. Cache valid discovery/JWKS documents for a bounded duration and response size; refresh JWKS once when an otherwise valid token references an unknown `kid`.

## Milestone 2: Add an OIDC nonce

Extend each OAuth transaction with an independent random `nonce` (store its digest if desired). Include the raw nonce in the authorization request and add `openid` to scope.

Keep purposes separate:

- `state` binds the browser callback to the client transaction;
- PKCE binds code redemption to the initiator holding the verifier;
- `nonce` binds the returned ID token to this authentication request.

Never reuse one random value for all three fields.

## Milestone 3: Validate the ID token in strict order

After code exchange, treat the ID token as untrusted compact input. Use the JOSE/JSON libraries to:

1. Enforce size/segment limits and parse the protected header.
2. Reject `alg=none` and algorithms outside the configured allowlist.
3. Require a bounded `kid` when the provider's key set needs it.
4. Select exactly one compatible signing key from the trusted issuer's JWKS.
5. Verify the JWS signature before trusting claims.
6. Parse claims and require expected JSON types.
7. Require exact `iss` match.
8. Require `aud` to contain your client ID; if multiple audiences exist, enforce the provider/profile's `azp` rule.
9. Require `now < exp`, acceptable `iat`, and `nbf` if present, with a small documented skew.
10. Constant-time compare the returned `nonce` with the transaction nonce.
11. Require a nonempty stable `sub`.

Validate `at_hash`/`c_hash` when they are present and required by the provider/profile/library; do not improvise their algorithm rules. Never use decoded-but-unverified claims.

An access token and ID token are not interchangeable: their audiences, consumers, and purposes differ.

## Milestone 4: Map external identity safely

Add:

```sql
CREATE TABLE IF NOT EXISTS oidc_identities (
    issuer TEXT NOT NULL,
    subject TEXT NOT NULL,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at INTEGER NOT NULL,
    PRIMARY KEY (issuer, subject)
);
```

Permit OIDC-created users to have no local password hash, or refactor password credentials into their own table. Use an explicit migration/table rebuild if changing the existing `NOT NULL` constraint.

Only the pair `(iss, sub)` is the external identity key. Do not auto-link by email, display name, or access-token contents. Email can change and may be unverified; two issuers can use the same subject string.

On first valid OIDC login, create a local `user` role and identity mapping transactionally. On later logins, resolve the same mapping. Design account linking as a separate re-authenticated, CSRF-protected operation rather than silently merging accounts.

## Milestone 5: Return to the local session model

After full ID-token validation and identity mapping:

1. Consume the OIDC transaction.
2. Create a fresh ordinary Week 8 session and CSRF token.
3. Set the same secure opaque session cookie.
4. Redirect to `/profile`.

Do not use the ID token itself as the browser's long-lived authlab cookie. Local logout deletes the local session; it does not necessarily log the user out of the identity provider. Document provider logout as a separate protocol concern.

Do not persist ID/access tokens unless a concrete feature needs them. If persisted, they become credentials requiring encryption, lifetime, and revocation design.

## Adversarial validation matrix

Your mock provider tests should produce correctly signed tokens with one defect at a time:

- wrong signature, unknown `kid`, disallowed algorithm;
- wrong issuer;
- audience missing client ID;
- wrong/missing `azp` for multiple audiences;
- expired, future-issued, or not-yet-valid token;
- wrong/missing nonce;
- missing/empty subject;
- valid token from another transaction replayed at this callback;
- key rotation where refreshed JWKS validates the new key;
- unverified email matching an existing local account (must not link).

No failure may create an identity mapping or local session. Keep detailed causes in test/internal diagnostics, with generic browser-facing failure.

## Live protocol inspection

Run one provider login through Caddy and trace:

```text
authlab redirect -> provider authentication/consent
-> code/state callback -> token request with verifier
-> ID token validation -> local session cookie -> /profile
```

Redact codes, state, nonce, cookies, and tokens. Decode a disposable ID token only in the lab and identify `iss`, `sub`, `aud`, `exp`, and `nonce`; then prove the application checks rather than merely prints each claim.

## End-of-week working state

At the end of Week 17, authlab discovers one pinned issuer, validates authorization state/PKCE plus ID-token signature and claims, maps `(issuer, subject)` to a local user, and creates the same revocable opaque session used by password login. Invalid tokens create no local state.

Verify:

```bash
make -C week-17 clean all test
sh week-17/tests/test_oidc_flow.sh
```

The test suite must include every adversarial claim case above and one complete success ending at authenticated `/profile`. Be able to explain what OIDC adds to OAuth and why local sessions still exist afterward. Copy this state into Week 18.
