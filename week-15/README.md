# Week 15: Tokens and JWT

## Goal

Inspect, issue, tamper with, and verify a signed JWT in a separate lab tool. Compare its operational properties with authlab's opaque sessions without replacing the simpler working session architecture.

Time budget: 5-8 hours.

## Start here

Begin with the completed Week 14 HTTPS deployment:

```bash
mkdir -p week-15/notes week-15/var week-15/tools
cp -R week-14/src week-14/tests week-14/tools week-14/Makefile week-14/schema.sql week-14/Caddyfile week-15/
make -C week-15 clean all test
```

Your starting application uses an opaque random cookie whose meaning and validity live in SQLite. Keep that authentication path intact. JWT work belongs in `tools/jwt_lab.c` and its tests until you can justify a different architecture.

Install a mature JOSE/JWT library and JSON parser rather than implementing signing:

```bash
brew install libjwt jansson pkg-config
pkg-config --modversion libjwt jansson
```

If the installed libjwt API differs from examples you find, use the headers/examples shipped with that exact version. Do not copy an unverified cryptographic implementation into the course.

## Understand the compact form

Write in `notes/jwt-model.md`:

```text
BASE64URL(protected header)
.
BASE64URL(claims JSON)
.
BASE64URL(signature bytes)
```

Base64URL is reversible encoding, not encryption. The signature covers the encoded header and payload; it does not conceal either.

Define these claims for the lab:

| Claim  | Lab meaning                                      |
| ------ | ------------------------------------------------ |
| `iss`  | exact issuer identifier, `https://authlab.local` |
| `sub`  | stable user identifier, not display name         |
| `aud`  | intended verifier, `authlab-jwt-lab`             |
| `iat`  | issue time                                       |
| `exp`  | hard expiry a few minutes later                  |
| `role` | authorization snapshot used only for discussion  |

Claims are statements. Verification establishes who signed the bytes; application policy decides whether those statements are acceptable now.

## Milestone 1: Decode without trusting

Add an `inspect` mode to `jwt_lab` that:

1. Requires exactly three nonempty dot-separated segments.
2. Enforces a small maximum token/segment size.
3. Base64URL-decodes the first two segments with a library routine, such as libsodium's URL-safe no-padding decoder.
4. Parses both as JSON with Jansson and rejects duplicate/invalid structure according to your chosen parser policy.
5. Pretty-prints header and claims under a banner saying `UNVERIFIED`.

Do not let successful decoding grant access. `inspect` should work on a token whose signature is nonsense, because inspection and authentication are separate operations.

## Milestone 2: Issue a signed demonstration token

Use libjwt to issue an HS256 token with the claims above. The HMAC key must be at least 32 random bytes. For interactive work, load it from a protected environment/file outside version control; tests may use an explicitly labeled non-secret fixture key.

Do not:

- hardcode a deployment key in source;
- print the key;
- place passwords, session IDs, reset tokens, or personal secrets in claims;
- invent signing code from SHA/HMAC primitives;
- treat HS256 as the right choice for a multi-service public-key architecture.

Add a `make jwt-lab` target and an invocation that prints one disposable token to the terminal. Remember that terminal output/history can persist; use only synthetic claims.

## Milestone 3: Verify signature and claims

Add a `verify` mode using the high-level library. Configure an explicit algorithm allowlist containing only the algorithm you issued. Never accept `none`, never choose a verification primitive solely from an untrusted `alg` header, and never confuse an asymmetric public key with an HMAC secret.

Only after signature verification succeeds, enforce:

- exact `iss` string;
- `aud` contains exactly/intentionally `authlab-jwt-lab`;
- nonempty expected `sub` type;
- `iat` is not implausibly in the future;
- `now < exp`, with at most a small documented clock skew;
- required claims have the expected JSON types.

Return different internal/test reasons but one generic external invalid-token result.

## Milestone 4: Tamper and observe

Perform these experiments:

1. Decode a valid token and read its role. No key is needed.
2. Change `role` in the decoded payload, Base64URL-encode it, and retain the old signature.
3. `inspect` still displays the attacker-chosen role.
4. `verify` must reject the modified signing input.
5. Issue an already expired token; signature verification may pass, but claim validation must reject it.
6. Verify with the wrong issuer, audience, and key; each must fail.

This separates encoding, signature validity, and application acceptance.

## Session-versus-token exercise

Make a table in `notes/jwt-model.md` comparing:

| Event                       | Opaque session                   | self-contained JWT without extra state          |
| --------------------------- | -------------------------------- | ----------------------------------------------- |
| server lookup               | required                         | often avoided                                   |
| immediate logout/revocation | delete row                       | needs denylist/key strategy or waits for expiry |
| role change                 | next DB-backed request sees it   | old claim persists until replacement/expiry     |
| payload visibility          | random identifier reveals little | claims are readable                             |
| signing-key compromise      | does not mint DB rows            | can mint accepted tokens until key response     |

Revoke an authlab session and prove replay fails immediately. Then “revoke” a valid lab JWT without a denylist and observe it still verifies until `exp`. Do not solve that by pretending the trade-off does not exist.

## Tests

Use fixture keys and injected times to cover:

- valid signature and all required claims;
- changed header, payload, or signature;
- missing/wrong-type claims;
- wrong issuer/audience;
- before, at, and after expiry;
- future `iat` outside skew;
- `alg=none` and unexpected algorithm rejection;
- oversized/malformed Base64URL and JSON;
- payload remains readable without a key.

Keep parser fuzz/sanitizer checks on the untrusted compact input.

## End-of-week working state

At the end of Week 15, authlab still authenticates with revocable server-side sessions. A separate warning-free `jwt_lab` can inspect untrusted claims, issue a signed synthetic token through libjwt, and accept it only after signature plus issuer/audience/time validation. Tampering remains visible but fails verification.

Verify:

```bash
make -C week-15 clean all test jwt-lab
sh week-15/tests/test_jwt_lab.sh
```

The test must include a modified payload that decodes successfully and verifies unsuccessfully. Be able to state what a JWT signature proves and what it does not. Copy this directory into Week 16.
