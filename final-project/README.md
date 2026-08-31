# Final project: consolidate and audit authlab

## Goal

Turn the Week 18 learning implementation into one reproducible final build, then audit it route by route and credential by credential. Add no new authentication mechanism during this phase; close gaps, document assumptions, and prove required security properties.

Time budget: 10-15 hours across implementation cleanup, testing, and review.

## Start here

Begin only after the Week 18 end-state tests pass:

```bash
mkdir -p final-project/notes final-project/var
cp -R week-18/src week-18/tests week-18/tools week-18/Makefile week-18/schema.sql week-18/Caddyfile final-project/
make -C final-project clean all test
```

Copy configuration examples with placeholder values if you created them. Do not copy `var/authlab.db`, binaries, cookie jars, TLS private keys, TOTP master keys, OAuth client secrets, raw reset tokens, or provider tokens.

Create these review documents as you work:

```text
final-project/
├── README.md
├── SECURITY_REVIEW.md
├── THREAT_MODEL.md
├── TEST_RESULTS.md
├── Caddyfile
├── Makefile
├── schema.sql
├── src/
├── tests/
└── tools/
```

## Phase 1: Freeze the intended surface

Inventory every route in `SECURITY_REVIEW.md`. At minimum, the local-password application must have:

```text
GET  /
GET  /register
POST /register
GET  /login
POST /login
POST /logout
GET  /profile
GET  /account
POST /change-email
POST /change-password
GET  /admin
GET  /forgot-password
POST /forgot-password
GET  /reset-password
POST /reset-password
```

List OAuth/OIDC, MFA, and intentionally educational routes separately. Remove or permanently repair temporary vulnerable routes. Delete debugging endpoints that reveal request bodies, headers, SQL, configuration, tokens, or internal errors.

For each route record method, authentication requirement, authorization rule, CSRF requirement, accepted input limits, output context, success status, and possible failure statuses.

## Phase 2: Rebuild from an empty runtime state

Prove the project can be reproduced without an old database:

1. Build with normal strict flags.
2. Initialize a new database from `schema.sql` or migrations.
3. Start authlab with explicit public-origin, database, cookie, and key configuration.
4. Start Caddy and access only the HTTPS public origin.
5. Run the complete integration suite.
6. Stop both processes and confirm tests clean up temporary state.

The build must not depend on untracked headers, a developer's existing database, or secrets embedded in source.

Document dependency names/versions and startup commands in the final project README. Provide a `.env.example` only with variable names and obviously fake placeholders; never provide live values.

## Phase 3: Audit trust boundaries

In `THREAT_MODEL.md`, draw:

```text
browser
-> Caddy/TLS
-> bounded HTTP parser/router
-> authentication + authorization policy
-> SQLite

authlab
-> OAuth/OIDC provider endpoints

authenticator/TOTP app
<-> browser/user
<-> authlab
```

For each boundary state:

- attacker-controlled bytes crossing it;
- credential or trusted assertion crossing it;
- validation performed immediately after crossing;
- maximum size and timeout;
- replay/expiry/revocation behavior;
- assumptions about browser, proxy, provider, clock, libraries, and host security.

Include out-of-scope threats honestly, such as host compromise, malicious dependencies, denial of service beyond the single-process lab, and production-grade secret/key management.

## Phase 4: Audit credential lifecycles

Create one table for every credential-like value:

| Credential    | Issuer        | Holder                  | Storage                           | Transport               | Expiry           | One-use       | Revocation                      |
| ------------- | ------------- | ----------------------- | --------------------------------- | ----------------------- | ---------------- | ------------- | ------------------------------- |
| password      | user          | user                    | Argon2id representation at server | HTTPS form              | changed/reset    | no            | replace hash                    |
| session ID    | authlab       | browser                 | server row + Secure cookie        | HTTPS cookie            | idle + absolute  | no            | delete row                      |
| CSRF token    | authlab       | browser page/session    | session row + form                | HTTPS body              | session lifetime | no            | delete session                  |
| reset token   | authlab       | user                    | digest at server, raw URL         | HTTPS URL/form          | short            | yes           | consume/delete                  |
| OAuth code    | provider      | browser/client          | transaction state                 | HTTPS redirect          | short            | yes           | provider consumes               |
| access token  | provider      | authlab/resource server | avoid/persist only by design      | HTTPS bearer header     | provider-defined | no            | provider-defined                |
| OIDC ID token | provider      | authlab client          | transient                         | HTTPS token response    | short            | no            | validate then use local session |
| TOTP code     | authenticator | user/authlab            | derived, never stored as code     | HTTPS form              | one time step    | reject replay | advance accepted step           |
| recovery code | authlab       | user                    | digest at server                  | HTTPS form/offline copy | policy           | yes           | mark used                       |

Adapt the table to your exact implementation. Any row without a clear expiry/revocation story is a review finding to resolve or document.

## Phase 5: Route-by-route adversarial tests

For every route, test absent, malformed, oversized, duplicate, and unexpected inputs. Then add targeted cases:

### Authentication and sessions

- unknown/wrong login responses remain externally equivalent;
- throttling applies to known and unknown identifiers;
- session tokens are random, rotated, and never logged;
- absent, malformed, expired, logged-out, and stolen-then-revoked tokens fail;
- cookie deletion and server invalidation are tested separately;
- password change/reset invalidates existing sessions.

### Authorization

- anonymous/user/admin requests exercise `401`, `403`, and `200`;
- changing user IDs, role fields, links, query parameters, or headers cannot elevate authority;
- new/unknown roles default to no privilege;
- OIDC claims cannot directly assign local administrator role.

### Injection and browser intent

- SQL metacharacters remain bound data;
- every dynamic HTML text value is escaped;
- CSP is present and legitimate pages produce no required inline-script exceptions;
- every authenticated mutation rejects missing/wrong-session CSRF tokens before changing state;
- CSRF browser lab fails after repair;
- method/path confusion and duplicate fields fail closed.

### Recovery, federation, and MFA

- reset credentials expire, work once, and are stored only as digests;
- OAuth state, PKCE, and callback are transaction-bound and one-use;
- OIDC signature, issuer, audience, time, nonce, and subject validation each have a negative test;
- external identity keys are `(issuer, subject)`, never email alone;
- TOTP challenge/code and recovery-code replay fail;
- MFA attempts are throttled and password-only login creates no full session when enabled.

Record exact commands, test names, and summarized outcomes in `TEST_RESULTS.md`; redact all credential values.

## Phase 6: Memory and parser testing

Build and run the whole test suite under Clang sanitizers:

```bash
make -C final-project clean
make -C final-project test CFLAGS='-std=c11 -g -Wall -Wextra -Wpedantic -Werror -fsanitize=address,undefined'
```

Run parser/form/cookie/query/token unit tests with empty, boundary-size, one-byte-oversize, embedded-NUL, and malformed delimiters. If a fuzzing harness is available, fuzz only local pure parser functions with fixed resource limits and save regression inputs for every crash.

Audit every allocation, SQLite statement, socket, CURL handle, JSON object, and JOSE object for one clear owner and cleanup on every error path. Sanitizer success supplements this reasoning; it does not replace it.

## Phase 7: Deployment assumptions

Confirm and document:

- authlab binds only to `127.0.0.1`;
- only Caddy exposes HTTPS;
- certificate verification is enabled outside the explicit local `curl -k` experiment;
- Secure/HttpOnly/SameSite/Path attributes are correct, including deletion cookies;
- public origin and trusted-proxy behavior come from trusted configuration;
- database and key files have restrictive local permissions;
- logs omit passwords, complete bodies, cookies, reset/OAuth/OIDC/MFA credentials;
- dependency/library failures cause a closed, observable startup/runtime failure.

This remains an educational server, not an Internet-ready framework. State that prominently.

## Optional vulnerable branches

After the clean baseline is committed and tagged locally, create one isolated branch per vulnerability only if it helps your study:

```text
vuln/sql-injection
vuln/session-fixation
vuln/no-csrf
vuln/plaintext-passwords
vuln/predictable-session-id
vuln/xss
vuln/broken-authorization
```

Keep these branches loopback-only and clearly labeled. For each, write vulnerability, attacker prerequisites, exploit, violated property, repair, and verification. Never merge a vulnerable branch into the final baseline.

## End-of-project working state

The final project is complete when a fresh checkout plus documented dependencies can build, initialize an empty database, run behind local Caddy HTTPS, and pass normal plus sanitizer test suites. Every route and credential appears in the security review, every claimed defense has a negative test, and remaining limitations are explicit.

Run the final gate:

```bash
make -C final-project clean all test
sh final-project/tests/test_all.sh
```

Then manually complete the browser-only HTTPS, cookie, CSRF, XSS/CSP, OIDC, and MFA observations and record their redacted results. You should be able to answer for every credential: who issued it, who holds it, what possession proves, where it is valid, how it leaks, when it expires, and how it is revoked.
