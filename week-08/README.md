# Week 8: Session security

## Goal

Rotate identifiers at authentication, enforce idle and absolute expiration, and make logout invalidate the server-side credential rather than merely hiding it in one browser.

Time budget: 5-8 hours.

## Start here

Begin with the completed Week 7 session system:

```bash
mkdir -p week-08/notes week-08/var
cp -R week-07/src week-07/tests week-07/Makefile week-07/schema.sql week-08/
make -C week-08 clean all test
```

Use a fresh database or write an explicit migration. Your starting state issues bearer credentials and checks one expiry timestamp, but has no logout and no idle timeout.

## Define the lifecycle before coding

In `notes/session-lifecycle.md`, define these transitions:

```text
password verified -> new active session
active + recent use -> active session
idle deadline passed -> invalid session
absolute deadline passed -> invalid session
logout -> deleted session
password reset (Week 12) -> all user sessions deleted
```

Choose lab values that do not require waiting during manual work, for example 30 minutes idle and 8 hours absolute. Tests must inject timestamps rather than sleep.

## Milestone 1: Represent both deadlines

Change the session schema for a fresh database to include:

```sql
created_at INTEGER NOT NULL,
last_seen_at INTEGER NOT NULL,
idle_expires_at INTEGER NOT NULL,
absolute_expires_at INTEGER NOT NULL
```

If preserving an existing database, write a numbered migration and run it transactionally. Do not silently assume `CREATE TABLE IF NOT EXISTS` changes an old table.

A session is valid only when:

```text
now < idle_expires_at AND now < absolute_expires_at
```

Use one boundary convention everywhere and test equality at each deadline.

## Milestone 2: Make time testable

Do not scatter direct `time(NULL)` calls across database and HTTP code. Pass the current Unix time into session functions, or wrap time behind a tiny clock function that tests can replace.

Add tests for:

- one second before each deadline;
- exactly at each deadline;
- one second after each deadline;
- a nonsensical clock value or integer overflow while adding lifetimes.

Check timestamp addition before performing it. A security expiry must not wrap into a valid distant date.

## Milestone 3: Refresh idle activity safely

After successful lookup, extend the idle deadline without moving the absolute deadline. To avoid a database write on every request, update only when `last_seen_at` is older than a small threshold, such as five minutes.

Use a conditional prepared update that still requires the session to be unexpired. If another request deleted the row, do not recreate it. This server is sequential today, but preserving the invariant makes the code correct when concurrency arrives.

Delete expired rows periodically or at startup. Expired rows must already be rejected even if cleanup has not run.

## Milestone 4: Prevent fixation by construction

Every successful login must generate a new random identifier after password verification. It must never:

- accept a session ID supplied in the login form;
- promote a cookie value into an authenticated row;
- reuse an existing anonymous/pre-login identifier;
- derive a new identifier from the old one.

If a session cookie arrived with the login request, delete its server-side row after successful authentication, then insert the new session and issue only the new value. Wrap deletion/insertion in a transaction if partial completion would violate your chosen policy.

For the attack exercise, temporarily change login to reuse a supplied cookie and show that a token chosen or known by an attacker remains valid after the victim logs in. Restore fresh random generation and show the old value fails. Never retain the vulnerable version.

## Milestone 5: Implement real logout

Add `POST /logout`:

1. Read the current session cookie.
2. Delete the matching server-side row with a prepared statement.
3. Return a cookie with the exact same name/path and `Max-Age=0` to remove the browser copy.
4. Return `303 See Other` to a public page.

Deletion should be idempotent: logging out with no/invalid session still clears the browser cookie and does not reveal whether a token existed.

Logout is state-changing and currently lacks CSRF protection. Record that gap prominently; Week 10 repairs it.

## Session tests

Expand `tests/test_sessions.sh` or the C test harness:

- two successful logins produce two different active sessions;
- a cookie present before login is not the cookie issued afterward;
- idle expiry and absolute expiry each cause `401`;
- activity advances only idle expiry and never absolute expiry;
- logout deletes the row;
- the copied pre-logout cookie fails after logout;
- repeated logout remains safe;
- browser-cookie deletion alone is shown not to invalidate a database row in a controlled test.

Use an injected test clock or direct session-function calls, never a long `sleep`.

## Attack and verification lab

Login and retain two copies of the same cookie:

```bash
cp /tmp/authlab-session.txt /tmp/authlab-stolen.txt
curl -i -b /tmp/authlab-session.txt -c /tmp/authlab-session.txt -X POST http://127.0.0.1:8080/logout
curl -i -b /tmp/authlab-stolen.txt http://127.0.0.1:8080/profile
```

The last response must be `401`. Query the sessions table by user ID and confirm the logged-out row is gone.

Then delete a browser/curl cookie without calling logout while its row still exists. Re-present a saved copy and observe that it still works. This demonstrates why client deletion is not server invalidation.

## End-of-week working state

At the end of Week 8, login always rotates to a newly generated session ID; valid sessions obey independent idle and absolute limits; activity cannot extend the absolute limit; and `POST /logout` deletes the row and expires the browser cookie.

Verify:

```bash
make -C week-08 clean all test
sh week-08/tests/test_sessions.sh
```

The integration test must include `login -> save token -> logout -> replay saved token -> 401`. Be able to explain why rotation prevents fixation but cannot protect a token stolen after login. Copy this state into Week 9.
