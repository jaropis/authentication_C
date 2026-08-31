# Authentication from First Principles in C

## Course purpose

This course is designed for a programmer who already knows some C and wants to understand web authentication at a level below ordinary frameworks.

The central project is to build a small web server in C and progressively add:

* HTTP request handling
* forms and cookies
* persistent users
* password authentication
* server-side sessions
* authorization
* CSRF protection
* password reset
* account and session management
* rate limiting
* OAuth 2.0 and OpenID Connect
* security testing

The objective is **not** to produce a production-grade web framework.

The objective is to understand precisely what frameworks are doing on your behalf.

By the end of the course, concepts such as `HttpOnly`, session fixation, CSRF tokens, password hashing, bearer tokens, OAuth authorization codes, refresh tokens, and OIDC ID tokens should no longer feel like disconnected security terminology. They should fit into a coherent model of authentication.

---

# Prerequisites

You should be comfortable with basic C:

```c
struct
pointer
malloc()
free()
char *
arrays
functions
files
```

You should also have basic familiarity with:

* command-line tools
* compilation with `gcc` or `clang`
* TCP/IP at a conceptual level
* SQL at a very elementary level
* HTML forms

Detailed knowledge of networking, cryptography, or web security is **not** assumed.

---

# Recommended tools

Use:

* C11 or newer
* `gcc` or `clang`
* `curl`
* a web browser
* SQLite
* SQLite C API
* libsodium or another mature cryptographic library
* AddressSanitizer
* Valgrind where available
* optionally Wireshark
* optionally Caddy or nginx for TLS termination

Recommended compiler flags during development:

```bash
-Wall -Wextra -Wpedantic -Werror
```

and periodically:

```bash
-fsanitize=address,undefined
```

The server should initially bind only to:

```text
127.0.0.1
```

Do not expose the experimental server directly to the Internet.

---

# Philosophy of the course

Three rules govern the entire course.

## Rule 1: Implement protocols, not cryptographic primitives

You will implement:

```text
sessions
cookies
login flows
authorization
CSRF protection
token handling
```

You will **not** implement:

```text
SHA-256
AES
Argon2
random-number generators
TLS
```

Use established cryptographic libraries.

## Rule 2: Observe everything

Whenever possible, inspect the actual protocol messages.

Use:

```bash
curl -v
```

and browser developer tools.

Do not merely write:

```text
set cookie
```

Observe the actual:

```http
Set-Cookie: session=...
```

header.

## Rule 3: Break your own server

For every security mechanism, first understand the insecure version.

The sequence should often be:

```text
implement
→ attack
→ explain why the attack works
→ repair
→ attack again
```

---

# Course structure

Suggested duration:

**14 weeks**

Suggested workload:

**5–8 hours per week**

Each week consists of four components:

1. theory
2. implementation
3. protocol inspection
4. attack/defense exercise

The complete project grows incrementally throughout the course.

---

# Week 1 — TCP and the smallest possible HTTP server

## Questions

What actually happens when a browser connects to a web server?

What is the relationship between:

```text
TCP
HTTP
web server
browser
```

?

## Theory

Study:

* IP addresses
* TCP ports
* sockets
* client/server model
* TCP connections
* request/response protocols
* HTTP as text carried over TCP

Understand the sequence:

```text
socket()
↓
bind()
↓
listen()
↓
accept()
↓
recv()
↓
send()
↓
close()
```

## Implementation

Write a server that listens on:

```text
127.0.0.1:8080
```

and always returns:

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 13

Hello, world!
```

Do not build an HTTP parser yet.

## Experiments

Connect using:

```bash
curl -v http://localhost:8080/
```

Then use:

```bash
nc localhost 8080
```

and manually type:

```http
GET / HTTP/1.1
Host: localhost

```

## Learning outcome

You should be able to explain:

> HTTP does not somehow "enter a C program." The operating system gives the program bytes received through a TCP socket.

---

# Week 2 — HTTP requests and responses

## Theory

Study HTTP request structure:

```http
GET /hello?name=Alice HTTP/1.1
Host: localhost:8080
User-Agent: curl/...
Accept: */*
```

Understand:

* request method
* request target
* protocol version
* headers
* body
* status codes

Study at least:

```text
200
302
400
401
403
404
405
500
```

Pay special attention to the difference between:

```text
401 Unauthorized
403 Forbidden
```

The naming of `401` is historically unfortunate: conceptually it usually means **not authenticated**.

## Implementation

Create a structure such as:

```c
struct http_request {
    char method[16];
    char path[1024];
    struct header headers[64];
    size_t header_count;
    char *body;
    size_t body_length;
};
```

Implement minimal parsing.

Support:

```text
GET /
GET /hello
GET /does-not-exist
```

## Security topic

Learn why parsing network input is dangerous in C.

Study:

* bounds checking
* malformed requests
* integer overflow
* buffer overflow
* maximum request sizes

## Exercise

Send deliberately malformed requests.

The server should return `400` rather than crash.

---

# Week 3 — Routing, forms, POST and application state

## Theory

Study:

```text
GET
POST
Content-Type
Content-Length
```

Understand HTML form submission.

For now use:

```text
application/x-www-form-urlencoded
```

## Implementation

Add:

```text
GET  /login
POST /login
```

The HTML form:

```html
<form method="POST" action="/login">
    <input name="username">
    <input name="password" type="password">
    <button>Log in</button>
</form>
```

For now, simply print the submitted username to the terminal.

Do not perform authentication yet.

## Important question

At what point does a password become secret?

Trace it:

```text
keyboard
→ browser
→ HTTP request body
→ socket
→ C buffer
→ application logic
```

This observation will become important once HTTPS is introduced.

---

# Week 4 — Cookies and browser state

This is the first major authentication week.

## Theory

HTTP is fundamentally stateless.

Study:

```http
Set-Cookie:
Cookie:
```

Understand the distinction between:

```text
cookie name
cookie value
cookie attributes
```

Study:

```text
Path
Domain
Expires
Max-Age
HttpOnly
Secure
SameSite
```

Do not merely memorize them.

Understand what attacker capability each attribute addresses.

## Implementation

Add:

```text
GET /cookie-test
```

Response:

```http
Set-Cookie: demo=hello
```

Then observe the browser sending:

```http
Cookie: demo=hello
```

Add a second cookie with:

```text
HttpOnly
SameSite=Lax
```

## Exercise

Use browser JavaScript:

```javascript
document.cookie
```

Observe which cookies JavaScript can see.

## Key question

Why does `HttpOnly` matter?

Answer in terms of XSS and session theft.

---

# Week 5 — Users, databases and password storage

## Theory

Introduce SQLite.

Create:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL
);
```

Study password storage.

Understand why these are wrong:

```text
plaintext password
MD5(password)
SHA256(password)
SHA256(password + salt)
```

Understand the purpose of:

* salts
* expensive password hashing
* memory hardness
* Argon2id
* bcrypt
* scrypt

You do not need to understand the internals of Argon2.

You do need to understand **why password hashing differs from ordinary hashing**.

## Implementation

Add:

```text
GET  /register
POST /register
```

Use a mature password-hashing library.

Store only a password hash representation.

Inspect the database manually.

Verify that the original password cannot be recovered from it.

## Exercise

Register two users with the same password.

Compare stored hashes.

Explain why they differ.

---

# Week 6 — Authentication

Now implement genuine password authentication.

## Theory

Define:

### Identification

> "I claim to be Alice."

### Authentication

> "Prove that you are Alice."

### Authorization

> "Alice, are you allowed to perform this operation?"

These are different problems.

## Implementation

Implement:

```text
POST /login
```

Process:

```text
receive username
↓
find user
↓
verify submitted password against password hash
↓
authentication succeeds or fails
```

Do **not** yet keep the user logged in.

Simply respond:

```text
Authentication successful
```

or:

```text
Invalid credentials
```

## Security exercise

Initially implement different errors:

```text
unknown username
wrong password
```

Notice that this permits **username enumeration**.

Then change the external response to:

```text
Invalid username or password
```

while optionally recording detailed internal diagnostics.

---

# Week 7 — Sessions

This is arguably the most important week of the course.

## Central question

Once authentication succeeds:

> How does request number 37 know that request number 12 successfully authenticated Alice?

## Theory

Study server-side sessions.

Conceptual database:

```text
sessions

session_id
user_id
created_at
expires_at
```

The browser receives:

```http
Set-Cookie: session=<random-token>
```

Then sends:

```http
Cookie: session=<random-token>
```

The cookie contains **a session identifier**, not the password.

## Implementation

After successful login:

1. generate a cryptographically secure random session identifier
2. store it
3. associate it with a user
4. send it in a cookie

Add:

```text
GET /profile
```

Pseudo-code:

```c
session_id = request_cookie(req, "session");

session = database_find_session(session_id);

if (!session)
    return unauthorized();

user = database_find_user(session.user_id);
```

## Exercise

Log in using the browser.

Copy the session token.

Use it manually with:

```bash
curl --cookie "session=..." http://localhost:8080/profile
```

You have just demonstrated an important fact:

> Whoever possesses a conventional session token can impersonate its owner.

This is a **bearer credential**.

---

# Week 8 — Session security

Study the attacks that naturally follow from Week 7.

## Topics

### Session theft

What happens if an attacker obtains the token?

### Session fixation

What happens if the attacker can cause the victim to authenticate using a session identifier already known by the attacker?

### Session expiration

Distinguish:

```text
idle timeout
absolute timeout
```

### Logout

What does logout actually mean?

It should generally invalidate the server-side session, not merely delete a browser cookie.

## Implementation

Add:

```text
POST /logout
```

Implement:

* session expiration
* session deletion
* generation of a fresh session identifier after authentication

## Experiment

Copy a valid session token.

Logout.

Try the copied token again.

It should no longer work.

---

# Week 9 — Authorization

Authentication answers:

> Who are you?

Authorization answers:

> What can you do?

## Database change

Users have roles:

```text
user
admin
```

Add:

```text
GET /admin
```

Logic:

```c
if (!current_user)
    return_401();

if (current_user->role != ROLE_ADMIN)
    return_403();
```

## Study

Understand:

* role-based access control
* permissions
* principle of least privilege
* default deny

## Important exercise

Create an authorization bug deliberately.

For example, protect the link to `/admin` in HTML but do **not** protect the server route.

Then manually request:

```bash
curl http://localhost:8080/admin
```

The lesson:

> Hiding a button is not authorization.

Authorization must be enforced server-side.

---

# Week 10 — Cross-Site Request Forgery

This is one of the most valuable security topics to implement manually.

Assume you have:

```text
POST /change-email
```

and the browser automatically sends the session cookie.

An attacker creates another web page containing a request targeting your server.

## Question

Why might your server accept it?

Because authentication cookies are often attached automatically by the browser.

## Study

Understand CSRF.

Then understand:

```text
SameSite
CSRF tokens
Origin checking
Referer checking
```

## Implementation

Generate a random CSRF token associated with the session.

Forms contain:

```html
<input type="hidden"
       name="csrf_token"
       value="...">
```

The server verifies it.

## Attack exercise

Create:

```text
evil.html
```

that attempts to perform an authenticated operation against your server.

First demonstrate that the attack works.

Then add CSRF protection.

Then demonstrate that it fails.

---

# Week 11 — XSS and the relationship to authentication

You do not need to become an XSS specialist, but authentication cannot be understood properly without it.

## Theory

Study:

```text
stored XSS
reflected XSS
HTML escaping
Content Security Policy
```

Create a deliberately vulnerable route that echoes user input into HTML.

Then inject:

```html
<script>alert(1)</script>
```

## Authentication connection

Ask:

> What happens if malicious JavaScript can read my session cookie?

Compare:

```text
session cookie without HttpOnly
```

against:

```text
session cookie with HttpOnly
```

Then understand the important limitation:

**HttpOnly does not make XSS harmless.**

The attacker may still issue authenticated requests from the victim's browser.

## Learning goal

Understand why:

```text
XSS
CSRF
cookie security
session authentication
```

are interconnected rather than independent subjects.

---

# Week 12 — Password changes, reset flows and account recovery

Authentication becomes much more interesting once the user forgets the password.

## Implement

```text
POST /change-password
```

Require:

* authenticated session
* current password
* CSRF protection
* new password

Then design:

```text
Forgot password?
```

## Theory

A reset link might contain:

```text
https://example.com/reset?token=<random-value>
```

Study properties of reset tokens:

* unpredictable
* sufficiently long
* short-lived
* single-use
* associated with one user
* invalidated after use

## Exercise

Implement the reset mechanism without actual email.

Print the reset URL to the server console.

Then use the URL manually.

## Important question

Is a password reset token an authentication credential?

Yes.

For its limited purpose, possession of it establishes authority to reset the account.

---

# Week 13 — Brute force, enumeration and timing

Study authentication as an adversarial system.

## Topics

### Brute-force attacks

Add login attempt throttling.

Compare strategies:

```text
per username
per IP
global
progressive delays
temporary account lockout
```

Discuss their disadvantages.

### User enumeration

Compare responses from:

```text
known@example.com
unknown@example.com
```

Inspect:

* response text
* HTTP status
* response timing

### Timing attacks

Study why comparisons of secret values may require constant-time comparison.

Do not attempt to implement sophisticated cryptographic timing attacks.

The objective is to understand the principle.

## Exercise

Write a simple benchmark showing that naïve string comparison can terminate after the first different byte.

Then use an appropriate constant-time comparison function for secret tokens.

---

# Week 14 — TLS and secure transport

So far, your server should have remained local.

Now ask:

> What happens if HTTP traffic crosses an untrusted network?

Without TLS, an observer may see:

```text
passwords
cookies
CSRF tokens
personal data
```

## Theory

Study conceptually:

```text
HTTPS = HTTP over TLS
```

Understand the role of:

* encryption
* integrity
* server authentication
* certificates
* certificate authorities

Do not implement TLS.

Instead place a reverse proxy in front:

```text
Browser
   |
 HTTPS
   |
 Caddy/nginx
   |
 HTTP localhost
   |
 C server
```

## Study cookie behavior

Now understand why:

```text
Secure
```

exists.

---

# Week 15 — Tokens and JWT

Until now you have used opaque sessions:

```text
abc9482c...
```

The server stores their meaning.

Now study self-contained tokens.

## Theory

Understand a JWT structurally:

```text
header.payload.signature
```

Decode one manually.

Understand that:

```text
Base64URL != encryption
```

Anyone possessing the token can generally inspect the payload.

Study claims such as:

```text
sub
iss
aud
exp
iat
```

## Compare architectures

### Server-side session

```text
cookie
  ↓
random identifier
  ↓
database
  ↓
user
```

### Signed token

```text
token
  ↓
verify signature
  ↓
claims
```

## Important discussion

Do **not** conclude:

> JWT is newer, therefore JWT is better.

Study trade-offs:

* revocation
* expiration
* key management
* token leakage
* database lookup requirements
* distributed services

For your small server, conventional sessions are probably simpler and safer.

---

# Week 16 — OAuth 2.0

OAuth terminology should now be easier because you understand authentication credentials and sessions.

## Central idea

OAuth 2.0 is primarily about **delegated authorization**.

Example:

> Allow application A limited access to data controlled by service B without giving application A the user's password.

Study roles:

```text
resource owner
client
authorization server
resource server
```

Study:

```text
authorization code
access token
refresh token
scope
redirect URI
```

Focus on:

```text
Authorization Code Flow with PKCE
```

## Draw the complete flow

```text
User
 |
 | visits application
 v
Client
 |
 | redirect
 v
Authorization Server
 |
 | authenticate + consent
 v
Authorization Server
 |
 | authorization code
 v
Client
 |
 | code + verifier
 v
Authorization Server
 |
 | access token
 v
Client
 |
 | access token
 v
Resource Server
```

## Critical conceptual exercise

Answer:

> Why is an OAuth access token generally not evidence that the user authenticated to my application?

This question leads directly into OpenID Connect.

---

# Week 17 — OpenID Connect

Study OIDC as an identity layer built on OAuth 2.0.

Introduce:

```text
ID token
```

Understand claims such as:

```text
iss
sub
aud
exp
nonce
```

Study:

```text
discovery document
JWKS
signature validation
issuer validation
audience validation
nonce validation
```

## Conceptual comparison

### Password authentication

```text
You authenticate the user yourself.
```

### OAuth

```text
Another service grants delegated access.
```

### OpenID Connect

```text
Another identity provider authenticates the user and provides your application with verifiable identity information.
```

## Optional implementation

Integrate a real OIDC provider into your C server.

Use an HTTP and JSON library rather than writing either parser from scratch.

After successful OIDC authentication, create your ordinary server-side session.

This reveals something important:

```text
OIDC login
      ↓
identity established
      ↓
your local session
      ↓
ordinary authenticated requests
```

OAuth/OIDC does not eliminate sessions automatically.

---

# Week 18 — MFA, passkeys and modern authentication

Do not necessarily implement all of these from scratch.

Study them conceptually after understanding password authentication deeply.

## TOTP

Understand:

```text
shared secret
time interval
HMAC
6-digit code
```

Discuss:

* enrollment
* recovery codes
* replay protection
* clock windows

## WebAuthn / passkeys

Study the conceptual shift.

Passwords:

```text
user knows secret
server verifies derived representation
```

Passkeys:

```text
client possesses private key
server stores public key
client signs challenge
server verifies signature
```

Understand why phishing resistance is much stronger.

Study:

```text
relying party
authenticator
credential
challenge
origin
RP ID
```

At this stage you should be able to understand WebAuthn conceptually without treating it as magic.

---

# Final project

Build a small C application called, for example:

```text
authlab
```

Required routes:

```text
GET  /
GET  /register
POST /register

GET  /login
POST /login
POST /logout

GET  /profile
POST /change-password

GET  /admin

GET  /forgot-password
POST /forgot-password
GET  /reset-password
POST /reset-password
```

Database:

```text
users
sessions
password_reset_tokens
login_attempts
```

Required properties:

* passwords stored using a proper password-hashing algorithm
* unpredictable session identifiers
* sessions expire
* logout invalidates sessions
* session identifiers are regenerated after login
* cookies use appropriate flags
* state-changing authenticated operations require CSRF protection
* authorization is enforced server-side
* password reset tokens are single-use and expire
* login attempts are throttled
* error messages avoid obvious account enumeration
* network input is bounded and validated

---

# Final security review

After completing the server, perform a structured audit.

For every route ask:

## Authentication

Does this route require authentication?

If so, where is that requirement enforced?

## Authorization

Which users may perform the operation?

Could changing an identifier bypass authorization?

## Input

What inputs are attacker-controlled?

What are their maximum lengths?

## SQL

Can user input affect SQL structure?

Use prepared statements everywhere.

## HTML

Can user input appear unescaped in HTML?

## CSRF

Can another website trigger this operation?

## Sessions

What happens if the session identifier is:

* absent?
* invalid?
* expired?
* stolen?
* reused after logout?

## Logging

Could logs accidentally contain:

```text
passwords
session tokens
reset tokens
```

?

They should not.

---

# Suggested attack laboratory

Once your implementation works, create deliberately vulnerable branches in Git.

For example:

```text
vuln/sql-injection
vuln/session-fixation
vuln/no-csrf
vuln/plaintext-passwords
vuln/predictable-session-id
vuln/xss
vuln/broken-authorization
```

For each vulnerability write a short note containing:

```text
1. Vulnerability
2. Attacker prerequisites
3. Exploit
4. Why the exploit works
5. Security property violated
6. Fix
7. Verification of fix
```

This could be one of the most educational parts of the entire course.

---

# Concepts you should be able to explain at the end

Without consulting notes, explain the following.

### Authentication versus authorization

Why are they separate?

### Password hashing

Why can't ordinary fast hashing be used safely?

### Salt

Why does every password hash need one?

### Session identifier

What does it represent?

Why must it be unpredictable?

### Bearer credential

Why does stealing a session cookie usually allow impersonation?

### Session fixation

How does regenerating the identifier after login help?

### `HttpOnly`

What attack does it mitigate?

What attack does it not prevent?

### `Secure`

Why is it meaningful only together with HTTPS?

### `SameSite`

How is it related to CSRF?

### CSRF

Why can a request be authenticated even though the user never intended to make it?

### XSS

Why can XSS undermine authentication security?

### `401` versus `403`

When should each be returned?

### Password reset token

Why is it effectively a temporary credential?

### JWT

What does its signature prove?

What does it not prove?

### OAuth 2.0

What exactly is being delegated?

### OpenID Connect

What does it add to OAuth?

### Access token versus ID token

Why are they conceptually different?

### MFA

Why is adding another password-like secret sometimes less valuable than it appears?

### Passkeys

Why can public-key authentication resist phishing better than passwords?

---

# Recommended study method

For each concept, maintain an authentication notebook with four headings:

```text
MECHANISM
ATTACK
DEFENSE
ASSUMPTIONS
```

For example:

```text
MECHANISM
Session cookie identifies an authenticated session.

ATTACK
Attacker steals cookie through XSS.

DEFENSE
HttpOnly prevents JavaScript from reading the cookie.

ASSUMPTIONS
Browser honors HttpOnly.
Server is still vulnerable to authenticated actions performed through XSS.
```

The fourth category is particularly important.

Security mechanisms are almost never simply "secure."

They are secure **under particular assumptions**.

---

# Optional extension: inspect real systems

After completing the basic course, examine authentication in familiar frameworks.

For example:

```text
Flask
Django
Express
Spring Security
ASP.NET
Axum
Actix
```

Instead of asking:

> How do I log a user in with Django?

ask:

> Where in this system are the mechanisms I implemented myself?

Identify:

```text
password hashing
session creation
session storage
cookies
CSRF
authorization middleware
password reset
OIDC
```

Framework documentation should make substantially more sense after implementing the mechanisms yourself.

---

# Optional extension: authentication architecture

Study more complex real-world architectures:

```text
browser
    ↓
frontend
    ↓
API gateway
    ↓
backend services
    ↓
identity provider
```

Topics:

* centralized identity
* service-to-service authentication
* API keys
* bearer access tokens
* token introspection
* token exchange
* mTLS
* service identities
* zero-trust architectures

Only study these after the fundamentals are solid.

---

# Final objective

The goal of the course is not that you finish saying:

> I know how to implement login in C.

It is that you can look at an authentication architecture and reason from first principles:

```text
Who is asserting an identity?

What evidence supports that assertion?

Who issued the credential?

Who possesses the credential?

What does possession authorize?

How is the credential transported?

How can it be stolen?

How can it be replayed?

When does it expire?

How is it revoked?

What assumptions does the system rely on?
```

Once you habitually ask those questions, authentication stops being a collection of framework features and becomes a comprehensible security protocol.
