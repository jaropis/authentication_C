# Week 4: Cookies and browser state

## Goal

Implement cookie parsing and `Set-Cookie` response generation, then observe which cookie decisions belong to the server and which are enforced by the browser.

Time budget: 5-8 hours.

## Start here

Begin with the completed Week 3 server:

```bash
mkdir -p week-04/notes
cp -R week-03/src week-03/tests week-03/Makefile week-04/
make -C week-04 clean all test
```

Your starting server handles bounded forms but has no durable client state. A second request is unrelated to the first except for process-local state such as the request counter.

## Learn the two directions

Write these in `notes/cookie-model.md`:

```http
Set-Cookie: demo=hello; Path=/; SameSite=Lax
Cookie: demo=hello
```

`Set-Cookie` is a response instruction to a user agent. `Cookie` is a later request header containing selected name/value pairs. Attributes such as `HttpOnly` are not echoed in the request.

For each attribute, record both its rule and the threat it addresses:

| Attribute             | Browser behavior to investigate                         |
| --------------------- | ------------------------------------------------------- |
| `Path`                | limits when a cookie is attached by URL path            |
| `Domain`              | controls eligible hosts; omit it for a host-only cookie |
| `Max-Age` / `Expires` | controls persistence, not server-side validity          |
| `HttpOnly`            | hides the cookie from browser JavaScript APIs           |
| `Secure`              | sends the cookie only over secure transport             |
| `SameSite`            | limits some cross-site requests and mitigates some CSRF |

Do not set `Secure` on the active lab cookie yet: the server is plain HTTP, so a conforming browser would not return it. Week 14 introduces HTTPS.

## Milestone 1: Parse the Cookie request header

Create `src/cookie.c` and `src/cookie.h`. Add a function that retrieves one cookie value into a caller-provided bounded buffer.

Parsing rules for this lab:

1. Read the `Cookie` header by case-insensitive header lookup.
2. Split cookie pairs on semicolons.
3. Trim optional spaces/tabs around each pair.
4. Split at the first `=`; values may contain later `=` bytes.
5. Match names exactly and case-sensitively.
6. Reject control bytes and values that exceed the destination.
7. Treat duplicate names as malformed rather than choosing one.

Do not URL-decode cookie values automatically. Your later random session tokens will use a cookie-safe hex or Base64URL alphabet.

Add tests for no header, one pair, multiple pairs, whitespace, empty value, `=` inside a value, duplicate name, and overlong value.

## Milestone 2: Generate Set-Cookie safely

Extend the response representation so it can emit multiple response headers without allowing CR/LF injection. A cookie helper should validate the name and value against a conservative safe alphabet and append attributes selected by explicit options.

Never combine two cookies into one `Set-Cookie` field. Send two separate header lines.

For this week, emit:

```http
Set-Cookie: demo=hello; Path=/; SameSite=Lax
Set-Cookie: protected=browser-only; Path=/; HttpOnly; SameSite=Lax
```

These values are demonstrations, not authentication credentials.

## Milestone 3: Add the observation route

Implement `GET /cookie-test`:

- If `demo` is absent, set both demonstration cookies and return an HTML page saying no demo cookie arrived.
- If `demo` is present, HTML-escape and display its value, then set it again with a short `Max-Age`, such as 600 seconds.
- Include no password or token in the page.

Also implement `GET /cookie-path` and set `path_only=yes; Path=/cookie-path`. Use it to compare requests to `/cookie-path` and `/`.

## curl cookie-jar lab

Start the server, then use a clean cookie jar:

```bash
rm -f /tmp/authlab-cookies.txt
curl -i -c /tmp/authlab-cookies.txt http://127.0.0.1:8080/cookie-test
cat /tmp/authlab-cookies.txt
curl -i -b /tmp/authlab-cookies.txt -c /tmp/authlab-cookies.txt http://127.0.0.1:8080/cookie-test
curl -i -b /tmp/authlab-cookies.txt http://127.0.0.1:8080/
```

In `notes/cookie-model.md`, distinguish what curl stored from what it sent. Delete the temporary jar afterward because later jars may contain credentials.

## Browser lab

Use a normal browser at `http://127.0.0.1:8080/cookie-test`:

1. Inspect the response under Network and locate both `Set-Cookie` lines.
2. Reload and inspect the outgoing `Cookie` header.
3. Inspect stored cookies in the browser's Application/Storage panel.
4. In the JavaScript console, run `document.cookie`.
5. Confirm that `demo` is visible and `protected` is absent because of `HttpOnly`.
6. Delete `demo` in developer tools and reload.

Explain this limitation explicitly: `HttpOnly` can prevent injected JavaScript from reading a credential, but injected JavaScript can still cause same-origin authenticated requests. It reduces one XSS consequence; it does not repair XSS.

## Attribute experiments

Change one property at a time, observe it, then restore the intended final cookies:

- Set `Path=/cookie-path`; inspect where it is sent.
- Set `Max-Age=5`; wait naturally while doing another task, then reload and inspect expiration.
- Set `Secure` over HTTP; observe that it is not useful in this setup.
- Omit `HttpOnly` from `protected`; observe it appear in `document.cookie`, then restore `HttpOnly`.

Do not set a broad `Domain`; a host-only cookie is the safer default.

## End-of-week working state

At the end of Week 4, your server should parse bounded incoming cookie values, emit multiple safe `Set-Cookie` headers, and provide a browser/curl observation route. The protected demonstration cookie has `HttpOnly`, `Path=/`, and `SameSite=Lax`; it does not yet have `Secure` because transport is HTTP.

Verify:

```bash
make -C week-04 clean all test
week-04/authlab & server_pid=$!
curl --retry 20 --retry-connrefused --retry-delay 0 --retry-max-time 5 --silent --output /dev/null http://127.0.0.1:8080/
headers=$(curl --silent --dump-header - --output /dev/null http://127.0.0.1:8080/cookie-test | tr -d '\r')
test "$(printf '%s\n' "$headers" | grep -c '^Set-Cookie:')" -eq 2
printf '%s\n' "$headers" | grep -q '^Set-Cookie: protected=.*HttpOnly.*SameSite=Lax'
kill "$server_pid"
```

Be able to answer: who creates the value, who stores it, who chooses when to attach it, and what each attribute prevents. This is the baseline for Week 5.
