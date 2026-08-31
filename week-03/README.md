# Week 3: Routing, forms, POST, and state

## Goal

Serve a login form, read a bounded POST body, decode `application/x-www-form-urlencoded` data, and dispatch by exact method/path pairs. Authentication is still intentionally absent.

Time budget: 5-8 hours.

## Start here

Begin with the completed Week 2 parser and router:

```bash
mkdir -p week-03/notes
cp -R week-02/src week-02/tests week-02/Makefile week-03/
make -C week-03 clean all test
```

You should begin with `GET /hello` returning `200`, malformed headers returning `400`, and no request-body support. Do not copy a compiled binary.

## Define the body boundary

Extend the documented protocol subset:

- Accept a decimal `Content-Length` for requests with bodies.
- Cap body length at 64 KiB and reject overflow while parsing the number.
- Reject duplicate `Content-Length` fields for this learning server.
- Reject every `Transfer-Encoding`; chunked bodies are not implemented.
- Reject requests containing both `Content-Length` and `Transfer-Encoding`.
- Read exactly the declared number of body bytes, including bytes already received after the header terminator.
- Return `400` if the connection closes before the declared body arrives.
- Return `413 Payload Too Large` when the configured body limit is exceeded.

Do not wait for the peer to close to find the body boundary. On HTTP/1.1 the peer may be waiting for your response.

## Milestone 1: Give requests owned body storage

Extend `struct http_request` with a body pointer/array and `body_length`. Choose one ownership model and document it:

- either the request owns a fixed 64 KiB array; or
- the server allocates exactly the validated length plus one byte and a cleanup function frees it.

If you allocate, make `http_request_destroy` safe on partially initialized requests and call it on every route and error path. The body is binary input even though this week's form decoder accepts only text.

Add request-reader tests for:

- a body arriving in the same read as headers;
- a body split across multiple reads;
- zero-length body;
- invalid, negative-looking, overflowing, duplicate, and oversized lengths;
- premature connection close.

A `socketpair` test is useful for exercising real reads without reserving a TCP port.

## Milestone 2: Route by method and path

Move dispatch into `src/router.c` and `src/router.h`, or an equivalent focused module. Match the complete method and parsed path.

Implement:

| Request                          | Result                                           |
| -------------------------------- | ------------------------------------------------ |
| `GET /login`                     | HTML login form                                  |
| `POST /login`                    | decode form and acknowledge username             |
| `GET /login?next=/profile`       | same login form; query is not part of route path |
| unsupported method on known path | `405` with an accurate `Allow` header            |
| unknown path                     | `404`                                            |

Keep handlers separate from parser code. The parser decides whether bytes form a request; the router decides which application behavior runs.

## Milestone 3: Serve the form correctly

Return HTML containing:

```html
<form method="POST" action="/login">
  <label>Username <input name="username" autocomplete="username" /></label>
  <label
    >Password
    <input name="password" type="password" autocomplete="current-password"
  /></label>
  <button type="submit">Log in</button>
</form>
```

Use `Content-Type: text/html; charset=utf-8` and calculate `Content-Length` from the actual bytes. The form may be a C string constant this week; do not add a template engine yet.

## Milestone 4: Decode form fields

Create `src/form.c` and `src/form.h` with a length-aware decoder for `application/x-www-form-urlencoded`:

1. Split fields at `&` and names from values at the first `=`.
2. Convert `+` to a space.
3. Decode `%HH` only when both following characters are hexadecimal digits.
4. Reject decoded NUL/control bytes and malformed escapes.
5. Enforce decoded limits: username at most 64 bytes and password at most 256 bytes.
6. Require exactly one `username` and one `password`; reject duplicates rather than guessing which wins.
7. Ignore or reject unknown fields consistently and test that policy.

Check the request `Content-Type`. Accept `application/x-www-form-urlencoded` with an optional `; charset=UTF-8`; return `415 Unsupported Media Type` for other types.

On a valid submission, print only the submitted username to the terminal and return a plain-text `200` response such as `Received login for alice`. Never print the password, complete body, or `Authorization`/`Cookie` headers.

## Application state exercise

Add a process-local request counter and include its value in a response header such as `X-Request-Number`. This is deliberately ephemeral application state:

1. Start the server and make three requests.
2. Observe the number increase.
3. Restart the server and observe it reset.
4. Explain why this cannot persist users and why it would diverge across multiple server processes.

The server is sequential, so concurrency control is not needed yet. Record that assumption.

## Test and inspect

Add form unit tests for spaces, punctuation, `%40`, malformed `%`, missing fields, duplicate fields, and maximum lengths. Then run:

```bash
make -C week-03 clean all test
week-03/authlab
```

In another terminal:

```bash
curl -i http://127.0.0.1:8080/login
curl -i --data 'username=Alice+Smith&password=not-a-real-secret' http://127.0.0.1:8080/login
curl -i --data 'username=alice%40example.com&password=x' http://127.0.0.1:8080/login
curl -i -H 'Content-Type: application/json' --data '{}' http://127.0.0.1:8080/login
curl -i --data 'username=a&username=b&password=x' http://127.0.0.1:8080/login
```

Use `curl -v` once and trace where the password exists: terminal argument, curl process/request buffer, TCP stream, server receive buffer, decoded field, and handler. Because transport is plain HTTP, it is not secret from a network observer; keep the server on loopback.

All password strings in these commands are disposable demonstrations that may remain in shell history. Never substitute a real password.

## End-of-week working state

At the end of Week 3, you should have exact method/path routing, a working `GET /login` HTML form, and a `POST /login` handler that safely reads and decodes a bounded form body. It prints only the username and performs no credential check.

Verify:

```bash
make -C week-03 clean all test
week-03/authlab & server_pid=$!
curl --retry 20 --retry-connrefused --retry-delay 0 --retry-max-time 5 --silent --output /dev/null http://127.0.0.1:8080/
test "$(curl --silent --output /dev/null --write-out '%{http_code}' http://127.0.0.1:8080/login)" = 200
response=$(curl --silent --write-out '\n%{http_code}' --data 'username=alice&password=demo' http://127.0.0.1:8080/login)
printf '%s\n' "$response" | grep -q 'Received login for alice'
printf '%s\n' "$response" | tail -n 1 | grep -q '^200$'
kill "$server_pid"
```

Be able to explain why `Content-Length`, not a C terminator or connection close, determines the body length in this implementation. Copy this state into Week 4.
