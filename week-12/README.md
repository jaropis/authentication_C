# Week 12: Password changes and recovery

## Goal

Add an authenticated password change and an unauthenticated, short-lived, single-use reset flow. Treat reset-token possession as narrowly scoped authentication and invalidate existing sessions after either password replacement.

Time budget: 7-10 hours.

## Start here

Begin with the completed Week 11 server:

```bash
mkdir -p week-12/notes week-12/var
cp -R week-11/src week-11/tests week-11/Makefile week-11/schema.sql week-12/
make -C week-12 clean all test
```

Use a fresh database or a transactional migration. Your starting account page has authentication, CSRF protection, and HTML escaping; it has no way to replace a password.

## Milestone 1: Authenticated password change

Add `POST /change-password` and a form on `GET /account`. Require:

- a valid current session;
- a valid CSRF token for that session;
- the current password;
- a new password meeting the Week 5 length policy;
- a confirmation field matching the new password.

Process in this order:

```text
authenticate session -> verify CSRF -> validate new input
-> load current encoded hash -> verify current password
-> hash new password -> update user -> invalidate sessions
```

Use `crypto_pwhash_str` to generate a fresh encoded hash and salt. Clear all submitted password buffers on every path.

Perform the password update and deletion of all sessions for that user in one database transaction. On success, expire the browser cookie and require login again. This simple policy makes stolen pre-change sessions stop working.

Use a generic failure such as `Current password is incorrect` without exposing stored credential details. Do not distinguish confirmation/policy errors from credential errors in logs containing input.

## Milestone 2: Model password-reset credentials

Add:

```sql
CREATE TABLE IF NOT EXISTS password_reset_tokens (
    token_hash TEXT PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at INTEGER NOT NULL,
    expires_at INTEGER NOT NULL,
    used_at INTEGER
);

CREATE INDEX IF NOT EXISTS password_reset_user_idx
    ON password_reset_tokens(user_id);
```

The URL will carry a random raw token; SQLite stores only its digest. Generate 32 random bytes with libsodium, encode them as 64 hex characters for the URL, and calculate a 32-byte digest with `crypto_generichash`. Encode that digest as 64 lowercase hexadecimal characters with `sodium_bin2hex` before binding it to the `token_hash TEXT` column.

Hashing a high-entropy token protects against accidental database disclosure. It does not rescue a predictable token; randomness remains essential.

Choose a short lifetime such as 15 minutes. As with sessions, pass time into testable functions and define validity as `used_at IS NULL AND now < expires_at`.

## Milestone 3: Request a reset without enumeration

Implement:

- `GET /forgot-password`: username/email form;
- `POST /forgot-password`: create a reset credential for a matching user;
- the same external status/body whether or not the account exists.

Use wording such as `If that account exists, reset instructions have been generated.` Keep body, status, and redirect behavior equal. Week 13 measures timing and adds a dummy-work strategy.

Because this course has no email service, print the complete reset URL to the local server terminal only for a known account. Prefix it clearly as development-only behavior. Do not write it to persistent logs, notes, test snapshots, or version control.

Invalidate previous unused reset tokens for the user before creating a new one, or define and test a policy allowing several. The simpler lab policy is one active token per user.

## Milestone 4: Display and consume the reset form

Implement:

```text
GET  /reset-password?token=<raw-token>
POST /reset-password
```

The GET route validates that a token could be used and displays a new-password form. Put the token in a hidden field, never in a cookie. Set `Referrer-Policy: no-referrer`, `Cache-Control: no-store`, and avoid third-party resources on reset pages so the URL is not leaked.

The POST route must:

1. Validate token format and hash it.
2. Find an unexpired, unused row by the hash.
3. Validate and hash the new password.
4. In one immediate transaction, conditionally mark the token used, update the user's password hash, and delete all that user's sessions and other reset tokens.
5. Commit only if exactly one valid token was consumed; otherwise roll back.
6. Clear the auth cookie and redirect to login.

Do not consume the token on GET. Link scanners and browser prefetch can issue GET requests without user intent.

## Milestone 5: Test the credential lifecycle

Use an injected clock and a temporary database. Cover:

- known and unknown reset requests have identical external responses;
- two generated raw tokens differ and only digests are stored;
- wrong, malformed, and expired tokens fail;
- GET does not consume a token;
- the first valid POST changes the password;
- the same raw token fails on the second POST;
- concurrent/double consumption cannot produce two successes;
- old password fails and new password succeeds;
- all old sessions fail after change/reset;
- CSRF is required for authenticated change-password;
- reset POST needs possession of its reset token rather than a session CSRF token.

Do not print raw tokens from automated tests unless output is isolated and deleted.

## Manual recovery lab

1. Register and log in as Alice; save a copy of the session cookie.
2. Request a reset for Alice.
3. Copy the development reset URL from the server terminal into the browser.
4. Submit a new disposable password.
5. Retry the URL; it must fail.
6. Replay the saved old session; `/profile` must return `401`.
7. Verify old password failure and new password success.
8. Request reset for an unknown account and compare the browser-visible response.

Record token properties and outcomes, never token values.

## End-of-week working state

At the end of Week 12, authenticated users can replace a password only with current-password and CSRF proof. A forgotten-password request can issue one short-lived reset URL whose raw token is shown only in the development console, stored only as a digest, consumed once, and followed by invalidation of all sessions.

Verify:

```bash
make -C week-12 clean all test
sh week-12/tests/test_password_change.sh
sh week-12/tests/test_password_reset.sh
```

The reset test must include first-use success, replay failure, expiry failure, and old-session failure. Be able to explain why the reset token is an authentication credential limited to one operation. Copy this state into Week 13.
