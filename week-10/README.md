# Week 10: Cross-Site Request Forgery

## Goal

Demonstrate a browser sending an authenticated state-changing request from another origin, then block it with an unpredictable token bound to the user's session.

Time budget: 6-8 hours.

## Start here

Begin with the completed Week 9 server:

```bash
mkdir -p week-10/notes week-10/var week-10/lab
cp -R week-09/src week-09/tests week-09/Makefile week-09/schema.sql week-10/
make -C week-10 clean all test
```

Your starting server has cookie authentication and authorization, but `POST /logout` changes session state without proving that the page initiating it intended the action. You will also add a deliberately unprotected email change as the visible attack target.

## Milestone 1: Create a state change to protect

Add a nullable, unique `email` column through the fresh schema or an explicit migration. Implement:

- `GET /account`: requires authentication and displays an email-change form;
- `POST /change-email`: requires authentication, validates one email value, updates only the current user's row, and redirects back to `/account`.

For the first attack run only, omit CSRF validation. Still use a prepared `UPDATE users SET email = ?1 WHERE id = ?2`; CSRF and SQL injection are different threats.

## Milestone 2: Build the attack page

Create `lab/evil.html`:

```html
<!doctype html>
<html lang="en">
  <meta charset="utf-8" />
  <title>CSRF lab</title>
  <form method="post" action="http://127.0.0.1:8080/change-email">
    <input type="hidden" name="email" value="attacker@example.test" />
    <button type="submit">Open harmless report</button>
  </form>
</html>
```

Log into authlab in a browser at `http://127.0.0.1:8080`, then serve the lab page from another origin:

```bash
cd week-10/lab
python3 -m http.server 8000 --bind 127.0.0.1
```

Visit `http://127.0.0.1:8000/evil.html` and submit. Ports make the origins different, but both URLs are same-site for SameSite cookie calculations because they share scheme and host. The `SameSite=Lax` cookie can therefore accompany this cross-origin POST. CORS does not stop an HTML form from sending it; CORS mainly controls whether scripts can read responses.

Confirm the email changed without the user visiting authlab's form. Save the request's `Origin`, `Cookie`, and form body in notes with token values redacted.

## Milestone 3: Bind a CSRF token to each session

Add a `csrf_token` column to `sessions`. When creating a session:

1. Generate a separate 32-byte random value with `randombytes_buf`.
2. Encode it as 64 lowercase hexadecimal characters.
3. Store it with the session row.
4. Do not put it in a cookie or URL.

Expose it only inside same-origin HTML forms after authenticating the request:

```html
<input type="hidden" name="csrf_token" value="..." />
```

Hex encoding keeps output simple, but still pass dynamic HTML through the correct escaping helper. A token is secret application state, not authorization by itself; it is useful because the attacker's origin cannot normally read the victim's form.

## Milestone 4: Verify before mutation

For every authenticated state-changing route, use this order:

```text
parse bounded request
-> resolve valid session
-> read exactly one submitted CSRF token
-> validate format and compare with the session token
-> authorize operation
-> mutate state
```

Reject missing, duplicate, malformed, or mismatched tokens with `403` and perform no mutation. Decode fixed-length hex and compare bytes with `sodium_memcmp`; do not use a prefix comparison or stop at the first mismatch.

Add CSRF validation to both `POST /change-email` and `POST /logout`. Every future authenticated POST must use the same guard.

Rotate the CSRF token with the session at login. Token lifetime therefore matches session lifetime.

## Milestone 5: Repeat the attack

Restart with protection enabled and log in again. Submit `evil.html` unchanged:

- the browser may still attach the session cookie;
- the body lacks the session's CSRF token;
- the server must return `403`;
- the stored email must remain unchanged.

Try inventing a 64-character token and copying a token from another login. Both must fail. A valid token used with the wrong session must fail.

Optionally add strict `Origin` validation as defense in depth for browser POSTs: when present, require the exact expected scheme, host, and port. Decide how requests without `Origin` are handled. Do this after the token experiment so the two defenses remain observable separately.

## Automated tests

Create `tests/test_csrf.sh` or an equivalent integration harness covering:

| Session | Token           | Expected                        |
| ------- | --------------- | ------------------------------- |
| none    | none            | `401`                           |
| valid   | missing         | `403`, no change                |
| valid   | malformed       | `403`, no change                |
| valid A | token B         | `403`, no change                |
| valid   | exact token     | redirect/success and one change |
| expired | old exact token | `401`, no change                |

Also verify that logout without a token does not delete the session and logout with the exact token does.

Use a test-only HTML-token extraction helper or direct database fixture; do not weaken the production endpoint to reveal tokens as JSON merely for tests.

## End-of-week working state

At the end of Week 10, the browser lab first demonstrates a cross-origin authenticated email change, then fails after repair. Every authenticated state-changing route checks a random per-session CSRF token before mutation; authentication cookies remain `SameSite=Lax` as an additional browser control.

Verify:

```bash
make -C week-10 clean all test
sh week-10/tests/test_csrf.sh
```

The test must prove both response status and unchanged database state for invalid tokens. Be able to explain how a request can carry valid authentication while lacking user intent. This protected state is copied into Week 11.
