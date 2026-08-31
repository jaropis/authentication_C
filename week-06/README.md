# Week 6: Authenticate passwords

## Goal

Turn `POST /login` into real identification plus password verification. Give every failed login the same external response and deliberately postpone login persistence until Week 7.

Time budget: 5-8 hours.

## Start here

Begin with the completed Week 5 registration system:

```bash
mkdir -p week-06/notes week-06/var
cp -R week-05/src week-05/tests week-05/Makefile week-05/schema.sql week-06/
make -C week-06 clean all test
```

Start with a fresh Week 6 database rather than copying `week-05/var/authlab.db`. Register at least one disposable test user. At this point `/login` only decodes a form and must not set an authentication cookie.

## Write the decision model

In `notes/authentication-model.md`, define:

- identification: the submitted username claims an account;
- authentication: password verification supplies evidence for that claim;
- authorization: deciding what an authenticated identity may do, introduced in Week 9.

Draw the precise login decision:

```text
parse and validate form
-> normalize username
-> query one user record
-> verify submitted password against encoded hash
-> return generic success or failure
```

The client must not be allowed to submit a user ID, role, stored hash, or verification outcome.

## Milestone 1: Fetch one credential record

Add a database function that executes a fixed prepared query similar to:

```sql
SELECT id, username, password_hash, role
FROM users
WHERE username = ?1;
```

Return an explicit result distinguishing `found`, `not found`, and `database error`. Copy SQLite-owned text into caller-owned bounded storage before finalizing the statement, or complete verification while the statement is valid. Never retain a pointer returned by `sqlite3_column_text` after `sqlite3_finalize`.

## Milestone 2: Verify using the encoded representation

For a found user, call:

```c
crypto_pwhash_str_verify(stored_hash,
                         submitted_password,
                         submitted_password_length);
```

Do not extract the salt, hash the input yourself, or compare encoded strings. The library parses the representation and performs the expensive verification.

Clear the submitted password buffer with `sodium_memzero` on success, failure, and internal-error paths. Do not clear the stored hash before the verification call has finished.

Optionally call `crypto_pwhash_str_needs_rehash` after successful verification and record that an upgraded hash should be stored. Implementing automatic rehash is useful but not required this week.

## Milestone 3: Make failures externally indistinguishable

Use one response for both unknown username and incorrect password:

```text
HTTP status: 401
Body: Invalid username or password
```

Keep status, content type, body text, redirect behavior, and headers identical. Internal diagnostics may distinguish database errors from invalid credentials, but must not include submitted passwords or complete hashes.

An actual database failure should return `500`, not masquerade as bad credentials. That is an operational failure, not an authentication decision.

This week removes obvious content-based username enumeration. Timing may still differ because an unknown account has no expensive password verification. Record that known limitation; Week 13 adds a dummy-hash strategy and measurement.

## Milestone 4: Keep success deliberately stateless

For a correct password, return:

```text
HTTP status: 200
Body: Authentication successful
```

Do not set a session cookie yet. Make a second request and observe that the server has no evidence linking it to the successful login. This gap motivates Week 7.

## Automated integration cases

Add a test script or C integration harness that initializes a temporary database and covers:

| Case                               | Expected result                |
| ---------------------------------- | ------------------------------ |
| known username, correct password   | `200`, success text            |
| known username, wrong password     | `401`, generic text            |
| unknown username                   | `401`, exact same generic text |
| malformed form                     | `400`                          |
| wrong content type                 | `415`                          |
| database unavailable after startup | controlled `500`, no crash     |

Compare complete wrong-password and unknown-user responses after removing nondeterministic headers such as request number. They should be byte-for-byte equal.

## Protocol lab

Run with a clean database, register Alice, and try all branches:

```bash
cd week-06
rm -f var/authlab.db
make clean all test
./authlab
```

In a second terminal:

```bash
curl -i --data 'username=alice&password=correct-horse-demo' http://127.0.0.1:8080/register
curl -i --data 'username=alice&password=correct-horse-demo' http://127.0.0.1:8080/login
curl -i --data 'username=alice&password=wrong-password' http://127.0.0.1:8080/login
curl -i --data 'username=nobody&password=wrong-password' http://127.0.0.1:8080/login
```

Inspect the successful response for `Set-Cookie`; there should be none related to authentication. Then request a protected-looking path and explain why the application cannot know it is still Alice.

## End-of-week working state

At the end of Week 6, registration still works and `POST /login` verifies passwords with libsodium. Correct credentials return `200`; unknown users and wrong passwords return the same `401` response. No authentication state survives into another request.

Verify:

```bash
make -C week-06 clean all test
rm -f /tmp/authlab-week06.db
(cd week-06 && AUTHLAB_DB=/tmp/authlab-week06.db ./authlab) & server_pid=$!
curl --retry 20 --retry-connrefused --retry-delay 0 --retry-max-time 5 --silent --output /dev/null http://127.0.0.1:8080/
curl --fail-with-body --silent --data 'username=alice&password=correct-horse-demo' http://127.0.0.1:8080/register >/dev/null
test "$(curl --silent --output /dev/null --write-out '%{http_code}' --data 'username=alice&password=correct-horse-demo' http://127.0.0.1:8080/login)" = 200
unknown=$(curl --silent --include --data 'username=nobody&password=wrong-password' http://127.0.0.1:8080/login | tr -d '\r' | grep -v '^X-Request-Number:')
wrong=$(curl --silent --include --data 'username=alice&password=wrong-password' http://127.0.0.1:8080/login | tr -d '\r' | grep -v '^X-Request-Number:')
test "$unknown" = "$wrong"
kill "$server_pid"
rm -f /tmp/authlab-week06.db
```

Be able to explain why successful password verification authenticates one request but does not, by itself, authenticate request number two. This working directory becomes Week 7.
