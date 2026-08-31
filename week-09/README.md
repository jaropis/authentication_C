# Week 9: Server-side authorization

## Goal

Use the authenticated user's database-backed role to enforce access to `GET /admin`, with a clear distinction between unauthenticated (`401`) and authenticated-but-forbidden (`403`).

Time budget: 5-7 hours.

## Start here

Begin with the completed Week 8 server:

```bash
mkdir -p week-09/notes week-09/var
cp -R week-08/src week-08/tests week-08/Makefile week-08/schema.sql week-09/
make -C week-09 clean all test
```

The `users.role` column has existed since Week 5, and session resolution should already return it. Use a fresh database and register two users. The starting server authenticates users but grants no role-specific capability.

## State the policy

Write an authorization matrix in `notes/authorization-policy.md`:

| Route          | Anonymous        | `user`     | `admin`    |
| -------------- | ---------------- | ---------- | ---------- |
| `GET /`        | allow            | allow      | allow      |
| `GET /profile` | `401`            | allow self | allow self |
| `GET /admin`   | `401`            | `403`      | allow      |
| `POST /logout` | idempotent clear | allow self | allow self |

Default deny is the rule: adding a new role or receiving an unrecognized role string must not accidentally grant access.

## Milestone 1: Keep identity and authority trusted

Ensure `current_user.role` originates only from the `users` row reached through a valid session. Never accept role/user ID from:

- form fields or query parameters;
- a separate unsigned cookie;
- a hidden input;
- a request header invented by the client;
- the visibility of a link in HTML.

Registration must always bind the literal default role `user`, regardless of extra submitted fields.

For the lab only, promote one account out of band while the server is stopped:

```bash
sqlite3 week-09/var/authlab.db "UPDATE users SET role = 'admin' WHERE username = 'adminuser';"
```

In a real system this would be a separately authorized administrative operation, not a direct production database command.

## Milestone 2: Centralize route guards

Create helpers with explicit results rather than scattered string comparisons:

```text
require_authenticated(request) -> user or response/error
require_role(request, ROLE_ADMIN) -> user or response/error
```

Represent known roles as an enum after validating the database text. Unknown values map to no privileges. Keep the route handler's business logic unreachable unless the guard succeeds.

Use these response rules:

- no valid current session: `401 Unauthorized`;
- valid user lacking the permission: `403 Forbidden`;
- valid administrator: continue to handler;
- database failure: `500 Internal Server Error`.

Do not redirect API-like authorization failures to login during these tests; explicit statuses make the distinction observable.

## Milestone 3: Implement GET /admin

The route should display a small administrator page only after the role guard passes. HTML-escape the username and set `Cache-Control: no-store`.

You may hide the navigation link from ordinary users as a usability feature, but the route check is the security boundary. Put the guard next to route dispatch so later edits cannot bypass it accidentally.

## Deliberately broken exercise

Temporarily remove the route guard while continuing to hide the Admin link from non-admin HTML:

1. Log in as an ordinary user and confirm there is no visible link.
2. Manually request `/admin` with curl.
3. Observe the secret content returning despite the hidden link.
4. Restore the server-side guard.
5. Repeat the exact curl request and require `403`.

Do not retain or commit the broken variant as the end state. The lesson is that UI visibility communicates policy but cannot enforce it.

## Authorization tests

Use three cookie jars: none, ordinary user, administrator. Cover:

```text
anonymous GET /admin -> 401
user GET /admin      -> 403
admin GET /admin     -> 200
user GET /profile    -> 200
malformed role row   -> 403 (or controlled internal error), never 200
extra role field at registration -> stored role remains user
```

Also test that changing a query parameter such as `?user_id=<admin-id>` does not affect identity or authority.

Think through role changes during an existing session. Because this design loads the role from the database on each request, a demotion takes effect immediately. If you later cache roles in sessions or tokens, revocation behavior changes.

## Protocol lab

After registering and promoting the lab admin, log both users in:

```bash
curl -c /tmp/authlab-user.txt --data 'username=alice&password=correct-horse-demo' http://127.0.0.1:8080/login
curl -c /tmp/authlab-admin.txt --data 'username=adminuser&password=correct-horse-demo' http://127.0.0.1:8080/login
curl -i http://127.0.0.1:8080/admin
curl -i -b /tmp/authlab-user.txt http://127.0.0.1:8080/admin
curl -i -b /tmp/authlab-admin.txt http://127.0.0.1:8080/admin
```

Record status, authenticated identity, policy decision, and response for each. Never print either session token in the notes.

## End-of-week working state

At the end of Week 9, `GET /admin` is protected at the server route. Anonymous requests get `401`, authenticated ordinary users get `403`, and database-backed administrators get `200`. Client-controlled identity or role values have no effect.

Verify:

```bash
make -C week-09 clean all test
sh week-09/tests/test_authorization.sh
```

The integration test must exercise all three status codes against the same route. Be able to explain why authentication is an input to authorization, not a substitute for it. This is the Week 10 baseline.
