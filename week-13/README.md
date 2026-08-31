# Week 13: Brute force, enumeration, and timing

## Goal

Measure the login endpoint as an adversary, add bounded attempt tracking, reduce username-dependent behavior, and use constant-time library comparisons for fixed-length secret tokens.

Time budget: 6-9 hours.

## Start here

Begin with the completed Week 12 application:

```bash
mkdir -p week-13/notes week-13/var week-13/tools
cp -R week-12/src week-12/tests week-12/Makefile week-12/schema.sql week-13/
make -C week-13 clean all test
```

Your starting responses hide obvious “unknown user” text, but an attacker can submit unlimited guesses and may distinguish account paths by status, response size, or time.

## Measure before changing behavior

Create `tools/measure_login.sh` or a small C client that performs many disposable local requests and records:

- HTTP status;
- response byte length;
- curl `time_total`;
- known-user/wrong-password versus unknown-user samples.

Use enough samples to see distributions, not one request. Alternate cases to reduce warm-up/order effects. Do not send high request volumes anywhere except your loopback lab.

Write in `notes/adversarial-login.md` which signals are equal and which differ. Network timing is noisy; the purpose is to identify gross differences and learn why “same message” is not the entire enumeration defense.

## Milestone 1: Add a dummy password hash

At startup, obtain a valid libsodium password-hash string for a fixed dummy password and keep it in process memory. For an unknown normalized username, call `crypto_pwhash_str_verify` against that dummy hash with the submitted password, discard the result, and return the same invalid-credentials response.

Requirements:

- use the same password length validation before account lookup;
- perform one expensive verify for both known-wrong and unknown accounts;
- never create a database user for the dummy value;
- keep database errors distinct as internal failures;
- do not claim exact constant-time behavior: database/cache paths can still differ.

Rerun the measurement and compare distributions. Document residual differences rather than promising they are eliminated.

## Milestone 2: Design throttling trade-offs

Use a lab policy such as:

```text
maximum 5 failed attempts per normalized identifier per 15 minutes
maximum 30 failed attempts per source IP per 15 minutes
successful login clears/reduces the identifier bucket
```

Discuss before implementing:

- per-username limits can let attackers lock out victims;
- per-IP limits affect users behind shared NAT and attackers can rotate addresses;
- global limits permit broad denial of service;
- progressive sleeps block this single-threaded server and reduce availability;
- no one dimension is a complete defense.

The exact numbers are lab choices, not production recommendations.

## Milestone 3: Persist bounded attempts

Add a table such as:

```sql
CREATE TABLE IF NOT EXISTS login_attempts (
    id INTEGER PRIMARY KEY,
    identifier TEXT NOT NULL,
    source_ip TEXT NOT NULL,
    attempted_at INTEGER NOT NULL,
    succeeded INTEGER NOT NULL CHECK (succeeded IN (0, 1))
);

CREATE INDEX IF NOT EXISTS login_attempt_identifier_time_idx
    ON login_attempts(identifier, attempted_at);

CREATE INDEX IF NOT EXISTS login_attempt_ip_time_idx
    ON login_attempts(source_ip, attempted_at);
```

Store no password, session ID, reset token, or complete request body. Bound identifier/IP lengths. Obtain source IP from the accepted peer address; Week 14 explains the trusted-proxy complication.

Use prepared queries to count attempts in a time window and periodically delete old rows. Pass an injected current time into policy code so boundary tests require no sleeping.

Apply the policy uniformly even when the username does not exist. After the limit, return `429 Too Many Requests` with a bounded `Retry-After`. Decide whether the body remains generic and document what the status reveals.

Also rate-limit forgot-password requests with a separate, intentionally conservative policy so attackers cannot use reset delivery as spam or enumeration.

## Milestone 4: Compare tokens in constant time

Create `tools/timing_compare.c` to run many comparisons where a candidate differs at byte 0, the middle, or the final byte. Compare naive early-exit code with `sodium_memcmp` for equal fixed lengths. Prevent compiler removal of the work and treat the results as illustration, not a precise remote exploit.

Audit secret-token comparisons:

- CSRF token bytes;
- reset-token digests when comparison occurs in C;
- any future state, nonce, or PKCE verifier values.

First validate exact format and length, then call `sodium_memcmp`. Do not use constant-time comparison for ordinary usernames, paths, or roles; they are not secrets and clarity matters more.

Password verification already belongs to `crypto_pwhash_str_verify`; do not replace it with a manual comparison.

## Automated policy tests

Cover:

- attempts 1 through the limit receive ordinary invalid-credential behavior;
- the next attempt is throttled;
- identifier and IP buckets work independently;
- unknown identifiers consume limits too;
- window boundary and `Retry-After` use the injected clock;
- old rows are pruned without deleting active-window rows;
- a valid login under policy succeeds and follows the documented reset behavior;
- no password or credential appears in `login_attempts`;
- malformed requests do not create unbounded rows;
- concurrent transactions cannot both bypass the final available attempt if concurrency is later introduced.

Avoid assertions on millisecond timing in normal tests; they are flaky. Test work-path decisions directly and keep timing measurement observational.

## Attack and defense lab

Run a loop that makes six wrong attempts against one disposable account and record statuses. The sixth should be `429` under the example policy. Then try an unknown identifier and confirm it follows the same policy.

Inspect:

```bash
sqlite3 week-13/var/authlab.db 'SELECT identifier, source_ip, attempted_at, succeeded FROM login_attempts ORDER BY id;'
```

Confirm the table contains only expected metadata. Advance time through the test clock, not the system clock, and prove the window recovers.

## End-of-week working state

At the end of Week 13, login does comparable password-hash work for known-wrong and unknown accounts, response content remains generic, failed attempts are bounded by identifier and source IP, reset requests are throttled, and fixed-length secret comparisons use libsodium.

Verify:

```bash
make -C week-13 clean all test
sh week-13/tests/test_rate_limit.sh
week-13/tools/timing_compare
```

Save a short before/after timing summary without claiming perfect indistinguishability. Be able to explain how throttling itself can harm availability. This state is copied into Week 14.
