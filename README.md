# Authentication from First Principles in C

This repository turns the [course syllabus](docs/syllabus.md) into an incremental implementation lab. Each numbered directory contains a self-contained guide with an explicit starting baseline, concrete milestones, attack/defense experiments, tests, and an end-of-week working state.

The syllabus overview says 14 weeks but contains 18 numbered weeks. These guides follow all 18 numbered modules. Take one module per calendar week unless you intentionally choose a compressed schedule.

## How to work through the course

1. Open the current week's `README.md` and complete its **Start here** commands.
2. Copy only the artifacts named there from the previous week.
3. Implement one milestone at a time and run the narrow test after each change.
4. Complete the protocol-inspection and attack/defense exercise locally.
5. Do not advance until every item in **End-of-week working state** is true.
6. Keep notes in that week's `notes/` directory, with credential values redacted.

Never copy generated binaries, live databases, cookie jars, private keys, provider secrets, raw reset links, or other credentials into the next week. Every server remains bound to `127.0.0.1`; Caddy is the only TLS-facing process introduced by the course.

## Weekly guides

| Module                       | Begin with                  | Finish with                                |
| ---------------------------- | --------------------------- | ------------------------------------------ |
| [Week 1](week-01/README.md)  | empty C program             | fixed HTTP response over TCP               |
| [Week 2](week-02/README.md)  | Week 1 listener             | bounded HTTP parsing and status routing    |
| [Week 3](week-03/README.md)  | parsed GET requests         | POST bodies, forms, and exact routing      |
| [Week 4](week-04/README.md)  | form-capable server         | observable cookie parsing/attributes       |
| [Week 5](week-05/README.md)  | cookie helpers              | SQLite users and Argon2id password hashes  |
| [Week 6](week-06/README.md)  | registered users            | real password verification, no persistence |
| [Week 7](week-07/README.md)  | one-request authentication  | opaque server-side sessions and profile    |
| [Week 8](week-08/README.md)  | basic bearer sessions       | rotation, expiry, and server-side logout   |
| [Week 9](week-09/README.md)  | hardened sessions           | server-enforced role authorization         |
| [Week 10](week-10/README.md) | unprotected state changes   | demonstrated and blocked CSRF              |
| [Week 11](week-11/README.md) | safe request intent         | demonstrated and escaped XSS plus CSP      |
| [Week 12](week-12/README.md) | protected accounts          | password change and single-use recovery    |
| [Week 13](week-13/README.md) | complete credential flows   | measured enumeration and rate limiting     |
| [Week 14](week-14/README.md) | local plaintext transport   | Caddy TLS and Secure cookies               |
| [Week 15](week-15/README.md) | opaque session architecture | separate JWT inspection/verification lab   |
| [Week 16](week-16/README.md) | token fundamentals          | OAuth authorization code + PKCE delegation |
| [Week 17](week-17/README.md) | OAuth client                | validated OIDC identity and local session  |
| [Week 18](week-18/README.md) | local/federated login       | TOTP MFA and complete passkey design       |

After Week 18, use the [final project and security review](final-project/README.md) to consolidate the implementation and audit every route and credential.

## Ground rules

- Implement protocol handling, not cryptographic primitives.
- Keep strict warnings enabled and use sanitizers regularly.
- Use prepared SQL statements and bounded network input everywhere.
- Observe actual HTTP with curl/browser tools; do not rely only on source inspection.
- Demonstrate each vulnerability only on the loopback lab, repair it, and prove the same attack fails.
- Treat session IDs, reset tokens, OAuth values, MFA secrets, and recovery codes as credentials even when they are disposable.
