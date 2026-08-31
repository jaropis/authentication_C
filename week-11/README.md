# Week 11: XSS and authentication

## Goal

Make reflected script injection observable, repair it with context-correct HTML escaping, and demonstrate why `HttpOnly` and CSRF tokens limit some consequences without making XSS harmless.

Time budget: 5-8 hours.

## Start here

Begin with the completed Week 10 server:

```bash
mkdir -p week-11/notes week-11/var week-11/lab
cp -R week-10/src week-10/tests week-10/Makefile week-10/schema.sql week-11/
make -C week-11 clean all test
```

Your starting state has session cookies, authorization, and CSRF-protected state changes. It already displays usernames and email addresses, so dynamic HTML output is now part of the authentication boundary.

Use only disposable local data and keep the server on loopback. The deliberately vulnerable route must not remain vulnerable at the end of the week.

## Map browser interpretation

In `notes/xss-model.md`, distinguish:

- untrusted bytes in an HTTP request;
- a C string containing those bytes;
- bytes emitted into an HTML text node;
- the browser parsing those bytes as markup and script;
- script running with authlab's origin and browser privileges.

HTML escaping is contextual. A function correct for an HTML text node is not automatically safe for a JavaScript string, CSS, URL, or unquoted attribute. This week, keep dynamic data only in HTML text nodes and quoted attribute values.

## Milestone 1: Add a reflected input route

Add bounded query-string decoding by reusing the form decoder's percent-decoding rules. Implement `GET /echo?message=...` with a maximum decoded message length of 1024 bytes.

For the attack phase, place the decoded message directly between HTML tags without escaping. Try:

```bash
curl --get --data-urlencode 'message=<script>alert(document.domain)</script>' http://127.0.0.1:8080/echo
```

Open the encoded URL in a browser and observe the alert. In developer tools, identify the response bytes and the DOM node the parser created. `curl` cannot execute XSS; the browser's HTML parser and JavaScript engine are essential to the exploit.

Also try an event-handler payload in the lab, because blocking only literal `<script>` would not solve output injection. Do not collect cookies or transmit data anywhere.

## Milestone 2: Implement text-node escaping

Create `src/html.c` and `src/html.h` with a length-aware helper that emits:

| Input | Output   |
| ----- | -------- |
| `&`   | `&amp;`  |
| `<`   | `&lt;`   |
| `>`   | `&gt;`   |
| `"`   | `&quot;` |
| `'`   | `&#39;`  |

The helper must:

1. accept an input pointer and explicit byte length;
2. calculate/check output growth without integer overflow;
3. fail if the destination is too small;
4. never partially produce a response that the caller mistakes for complete;
5. reject embedded NUL/control bytes under your existing text policy.

Add unit tests for every special character, mixed input, empty input, maximum input, insufficient output capacity, and already escaped-looking text such as `&lt;` (which should become `&amp;lt;`).

Use the helper for the echo message, username, email, and every other dynamic HTML value. Centralize rendering enough that a later handler does not accidentally forget the boundary.

## Milestone 3: Repeat and repair

Repeat the exact browser URL. The page should display literal `<script>...` text and no script should execute. Inspect source/DOM and identify the escaped response bytes.

Do not “fix” XSS by deleting angle brackets, searching for attack strings, or disabling JavaScript in your own browser. The defense is to encode untrusted data for the output context.

## Milestone 4: Add a restrictive CSP

Add a response header for HTML pages:

```http
Content-Security-Policy: default-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'; form-action 'self'
```

Remove inline scripts/styles that would force `'unsafe-inline'`. Also add:

```http
X-Content-Type-Options: nosniff
Referrer-Policy: no-referrer
```

Treat CSP as defense in depth. Keep output escaping even when CSP appears to block the demonstration payload. Record which payloads CSP blocks and which underlying encoding defect still exists if escaping is removed.

## Authentication consequence lab

While logged in, inspect `document.cookie`:

- a non-HttpOnly demo cookie may be visible;
- the `session` cookie must not be visible;
- the browser can nevertheless attach the session cookie to same-origin requests initiated by script.

Reason through a same-origin injected script that reads the `/account` DOM, obtains its CSRF token, and submits `/change-email`. CSRF tokens defend against another origin that cannot read them; XSS runs inside the trusted origin and can often read them. You do not need to exfiltrate a credential to prove the point.

Record this matrix:

| Defense       | Helps with                        | Does not stop                         |
| ------------- | --------------------------------- | ------------------------------------- |
| `HttpOnly`    | JavaScript reading session cookie | script issuing authenticated requests |
| CSRF token    | cross-origin form lacking token   | same-origin XSS reading token/DOM     |
| HTML escaping | data becoming executable markup   | bugs in other output contexts         |
| CSP           | some script execution paths       | the need for correct encoding         |

## Tests

Add response-level tests that submit each special character and assert the body contains only its escaped form. Include all current dynamic HTML routes, not just `/echo`.

Run a browser pass and:

- verify no alert/event handler executes after repair;
- verify forms still submit;
- check the developer console for CSP violations caused by legitimate pages;
- verify session and CSRF flows still pass.

## End-of-week working state

At the end of Week 11, `/echo` safely displays hostile-looking text, all dynamic HTML goes through context-appropriate escaping, and HTML responses carry a restrictive CSP. Session cookies remain `HttpOnly`, while your notes explain why XSS can still perform authenticated actions.

Verify:

```bash
make -C week-11 clean all test
sh week-11/tests/test_csrf.sh
sh week-11/tests/test_xss.sh
```

The XSS test must assert escaped response bytes; complete the browser check manually because shell tools do not execute JavaScript. This repaired directory is copied into Week 12.
