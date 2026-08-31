# Week 7: Server-side sessions

## Goal

After a successful password check, create a cryptographically random server-side session, place only its identifier in a cookie, and use it to authenticate `GET /profile`.

Time budget: 6-8 hours.

## Start here

Begin with the completed Week 6 authentication server:

```bash
mkdir -p week-07/notes week-07/var
cp -R week-06/src week-06/tests week-06/Makefile week-06/schema.sql week-07/
make -C week-07 clean all test
```

Use a fresh Week 7 database. Your starting `POST /login` proves a password for one request and returns no authentication cookie. That missing link between requests is what you will build.

## Model the credential

Write this in `notes/session-model.md`:

```text
browser cookie value
-> random opaque session identifier
-> sessions row
-> user_id
-> users row
-> authenticated identity
```

The cookie is not the user, password, or authorization role. It is a bearer credential: anyone who presents a valid identifier is treated as the associated user.

## Milestone 1: Add session persistence

Extend `schema.sql`:

```sql
CREATE TABLE IF NOT EXISTS sessions (
    session_id TEXT PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at INTEGER NOT NULL,
    expires_at INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS sessions_user_id_idx
    ON sessions(user_id);
```

Use Unix timestamps consistently in UTC. For this first version, choose an absolute lifetime such as eight hours. Week 8 adds distinct idle and absolute limits.

Add prepared database functions to create a session and look one up with its user in one query. A useful lookup shape is:

```sql
SELECT u.id, u.username, u.role, s.expires_at
FROM sessions AS s
JOIN users AS u ON u.id = s.user_id
WHERE s.session_id = ?1 AND s.expires_at > ?2;
```

## Milestone 2: Generate an opaque identifier

Use libsodium, which is already initialized:

1. Allocate 32 random bytes.
2. Fill them with `randombytes_buf`.
3. Encode them as 64 lowercase hexadecimal characters with `sodium_bin2hex` into a 65-byte array.
4. Insert the exact encoded identifier into `sessions` with the authenticated user ID and timestamps.
5. Clear temporary random bytes after encoding.

Do not use `rand`, timestamps, counters, usernames, UUID versions not intended as secrets, or a hash of any predictable value. Token length and unpredictability are separate properties.

If insertion reports the fantastically unlikely primary-key collision, generate a new identifier and retry a small fixed number of times. Any other database failure is a `500` and must not set a cookie.

## Milestone 3: Issue the session cookie

After password verification and successful database insertion, return:

```http
Set-Cookie: session=<64-hex-character-token>; Path=/; HttpOnly; SameSite=Lax
```

Do not add `Secure` until HTTPS exists in Week 14. Do not include user ID, role, expiry, or password material in the value. Never log the token.

Use `303 See Other` with `Location: /profile` after the form POST so browser refresh does not resubmit credentials. Confirm curl does not follow the redirect unless `-L` is supplied.

## Milestone 4: Resolve the current user

Create a helper with three outcomes: authenticated user, anonymous request, and internal error. It should:

1. Retrieve the `session` cookie with the Week 4 bounded parser.
2. Require exactly 64 lowercase hexadecimal characters before querying.
3. Query the session/user join with the current time.
4. Return a caller-owned `current_user` containing only required fields.
5. Never trust a username or role from another cookie or request parameter.

Malformed, absent, unknown, and expired identifiers all produce the same anonymous result externally. A database failure remains an internal error.

## Milestone 5: Add the protected route

Implement `GET /profile`:

- no valid session: `401 Unauthorized` with a small generic body;
- valid session: `200 OK` and an HTML-escaped username;
- database error: `500 Internal Server Error`.

Use `Cache-Control: no-store` on authentication responses and authenticated pages. Do not display the session identifier in HTML.

## Tests

Add deterministic tests around session lookup using a temporary database:

- inserted token resolves to the correct user;
- absent, malformed, unknown, and expired tokens are anonymous;
- one changed hex character invalidates the token;
- two logins generate different session IDs;
- registration cannot set `role=admin` through an extra form field;
- `/profile` never accepts `user_id` or `username` parameters as identity.

Do not assert a particular random token value. Assert format, length, uniqueness in a reasonable sample, and lookup behavior.

## Bearer-credential lab

Run a clean server and register/login with a cookie jar:

```bash
cd week-07
rm -f var/authlab.db /tmp/authlab-session.txt
make clean all test
./authlab
```

From another terminal:

```bash
curl --silent --data 'username=alice&password=correct-horse-demo' http://127.0.0.1:8080/register
curl -i -c /tmp/authlab-session.txt --data 'username=alice&password=correct-horse-demo' http://127.0.0.1:8080/login
curl -i -b /tmp/authlab-session.txt http://127.0.0.1:8080/profile
cat /tmp/authlab-session.txt
```

Copy only the session value into a second curl invocation:

```bash
curl -i --cookie 'session=PASTE_TOKEN_HERE' http://127.0.0.1:8080/profile
```

It succeeds without a password because possession is the proof. Treat the jar and copied token as disposable credentials and delete them when finished.

Inspect SQLite only in this controlled lab:

```bash
sqlite3 var/authlab.db 'SELECT length(session_id), user_id, created_at, expires_at FROM sessions;'
```

Normal diagnostics must not dump this table.

## End-of-week working state

At the end of Week 7, valid login creates a fresh opaque session row and sets a host-only `HttpOnly; SameSite=Lax; Path=/` cookie. `/profile` authenticates solely by resolving that cookie through SQLite. Invalid or absent tokens return `401`.

Verify with an integration script in `tests/test_sessions.sh` that starts a temporary database and performs this sequence:

```text
register -> login -> capture cookie -> GET /profile = 200
                               -> alter cookie -> GET /profile = 401
no cookie -------------------------------------> GET /profile = 401
```

Then run:

```bash
make -C week-07 clean all test
sh week-07/tests/test_sessions.sh
```

Be able to explain precisely why stealing the cookie is sufficient for impersonation. This working state is copied into Week 8.
