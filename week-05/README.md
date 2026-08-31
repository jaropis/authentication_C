# Week 5: Users, SQLite, and password storage

## Goal

Persist users in SQLite and register passwords using libsodium's Argon2id-based password-hashing API. Store only the encoded hash string, never the original password.

Time budget: 6-8 hours.

## Start here

Begin with the completed Week 4 server and copy source artifacts, not runtime state:

```bash
mkdir -p week-05/notes week-05/var
cp -R week-04/src week-04/tests week-04/Makefile week-05/
make -C week-05 clean all test
```

Your starting point has safe bounded form and cookie parsing but no database. Do not copy any cookie jar or database from another week.

Install and verify mature dependencies on macOS:

```bash
brew install sqlite libsodium pkg-config
export PKG_CONFIG_PATH="$(brew --prefix sqlite)/lib/pkgconfig${PKG_CONFIG_PATH:+:$PKG_CONFIG_PATH}"
pkg-config --modversion sqlite3 libsodium
```

Homebrew's SQLite is keg-only because macOS includes an older system SQLite. The explicit `PKG_CONFIG_PATH` prevents headers and the linked library from different installations being mixed. Add that export to your shell setup for this course, or encode the same prefix in the `Makefile`. If Homebrew is not your package manager, install development headers through your package manager and keep the same SQLite/libsodium APIs. Do not replace libsodium with a password hash you write yourself.

## Milestone 1: Add explicit dependency flags

Update the `Makefile` to obtain compile/link flags with `pkg-config` where available:

```make
HOMEBREW_SQLITE_PREFIX := $(shell brew --prefix sqlite)
PKG_CONFIG_PATH := $(HOMEBREW_SQLITE_PREFIX)/lib/pkgconfig:$(PKG_CONFIG_PATH)
export PKG_CONFIG_PATH

CPPFLAGS += $(shell pkg-config --cflags sqlite3 libsodium)
LDLIBS += $(shell pkg-config --libs sqlite3 libsodium)
```

If you are not using Homebrew, omit `HOMEBREW_SQLITE_PREFIX` and provide the appropriate `PKG_CONFIG_PATH` externally. Keep warnings and sanitizers usable. Call `sodium_init()` once during startup and fail closed if it returns a negative result.

## Milestone 2: Design and initialize the database

Create `schema.sql`:

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY,
    username TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'user'
        CHECK (role IN ('user', 'admin')),
    created_at INTEGER NOT NULL
);
```

Create `src/db.c` and `src/db.h`. On startup:

1. Open `var/authlab.db` with `sqlite3_open_v2` and explicit read/write/create flags.
2. Enable foreign keys on every connection.
3. Set a short busy timeout.
4. Apply the idempotent schema or fail startup with a useful internal error.
5. Close the database cleanly on normal shutdown paths.

The current working directory affects the relative database path. Run the executable from `week-05`, or accept the path through a documented environment variable such as `AUTHLAB_DB`.

Do not print SQL containing user data or configure a callback that logs passwords/hashes.

## Milestone 3: Validate registration input

Add `GET /register` with a username/password form and `POST /register` with these rules:

- username is 3-64 ASCII characters from letters, digits, `_`, `-`, or `.`;
- normalize the username policy once (for example, lowercase ASCII) before both lookup and storage;
- password is 12-256 bytes for the lab;
- reject embedded NUL/control bytes and malformed form encoding;
- return `400` for invalid input and `409 Conflict` for an already registered username;
- never echo the submitted password.

This is a policy exercise, not a universal username/password policy. Document your exact choices in `notes/registration-policy.md`.

## Milestone 4: Hash through libsodium

Allocate `crypto_pwhash_STRBYTES` bytes for the encoded representation and call:

```c
crypto_pwhash_str(encoded_hash,
                  password,
                  password_length,
                  crypto_pwhash_OPSLIMIT_INTERACTIVE,
                  crypto_pwhash_MEMLIMIT_INTERACTIVE);
```

Important properties to observe:

- libsodium generates a random salt internally;
- the encoded string includes algorithm, parameters, salt, and derived result;
- the encoded string is safe to store as text;
- the password still exists transiently in process memory and must never be logged;
- clear the password buffer with `sodium_memzero` as soon as the handler no longer needs it.

Treat hash failure as `500 Internal Server Error`; do not insert a user without a valid hash.

## Milestone 5: Insert with a prepared statement

Use this sequence for all SQL containing values:

```text
sqlite3_prepare_v2
-> sqlite3_bind_text / sqlite3_bind_int64
-> sqlite3_step
-> sqlite3_finalize
```

Bind the normalized username, encoded hash, role `user`, and current Unix time. Never interpolate form values into SQL. Map the SQLite unique-constraint result to the external `409` response, while keeping detailed diagnostics in the terminal free of secrets.

Add a database test that uses an in-memory SQLite database or a temporary file. Test successful insert, duplicate username, and a username containing a quote to prove it remains data rather than SQL syntax.

## Registration lab

Initialize a clean database and run the server:

```bash
cd week-05
rm -f var/authlab.db
make clean all test
./authlab
```

From a second terminal:

```bash
curl -i http://127.0.0.1:8080/register
curl -i --data 'username=alice&password=correct-horse-demo' http://127.0.0.1:8080/register
curl -i --data 'username=bob&password=correct-horse-demo' http://127.0.0.1:8080/register
curl -i --data 'username=alice&password=another-password' http://127.0.0.1:8080/register
sqlite3 week-05/var/authlab.db 'SELECT id, username, password_hash, role FROM users ORDER BY id;'
```

Confirm Alice and Bob's hash strings differ even though their passwords were the same. Identify the salt/parameter fields without attempting to reverse either password.

Search your source and notes for the demonstration password and remove accidental persistent copies. Shell history is another place secrets can remain; use only disposable lab passwords in these commands.

## Attack and repair exercise

Temporarily construct an SQL string by concatenating a username and try a quote-bearing input. Observe why SQL structure becomes attacker-controlled. Immediately restore prepared statements and repeat the input; it should be stored or rejected solely by username policy, never interpreted as SQL.

Do not leave the vulnerable version in the final directory.

## End-of-week working state

At the end of Week 5, `GET /register` serves a form and `POST /register` validates input, hashes the password with libsodium, and stores a unique user in SQLite through a prepared statement. The database contains encoded password hashes only.

Verify from the repository root with a disposable database:

```bash
make -C week-05 clean all test
rm -f /tmp/authlab-week05.db
(cd week-05 && AUTHLAB_DB=/tmp/authlab-week05.db ./authlab) & server_pid=$!
curl --retry 20 --retry-connrefused --retry-delay 0 --retry-max-time 5 --silent --output /dev/null http://127.0.0.1:8080/
curl --fail-with-body --silent --data 'username=alice&password=correct-horse-demo' http://127.0.0.1:8080/register
test "$(sqlite3 /tmp/authlab-week05.db 'SELECT count(*) FROM users WHERE username = "alice" AND password_hash <> "correct-horse-demo";')" = 1
kill "$server_pid"
rm -f /tmp/authlab-week05.db
```

Be able to explain why a unique salt defeats shared precomputation but does not make a fast hash suitable for passwords. This directory is the Week 6 baseline.
