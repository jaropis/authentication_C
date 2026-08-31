# Week 2: Parse bounded HTTP requests

## Goal

Replace the fixed-response toy with a small, deliberately limited HTTP/1.1 parser. Route three `GET` targets and return `400`, `404`, or `405` without crashing on malformed input.

Time budget: 5-8 hours.

## Start here

Begin with the completed Week 1 server. From the repository root:

```bash
mkdir -p week-02/tests week-02/notes
cp -R week-01/src week-02/
cp week-01/Makefile week-02/
```

Do not copy the compiled `authlab` binary. Confirm the copied baseline before refactoring:

```bash
make -C week-02 clean all
week-02/authlab & server_pid=$!
curl --retry 20 --retry-connrefused --retry-delay 0 --retry-max-time 5 --silent --output /dev/null http://127.0.0.1:8080/
curl --silent http://127.0.0.1:8080/
kill "$server_pid"
```

It should still print `Hello, world!`. At this point the server reads only once and does not understand request boundaries; that is the behavior you are replacing.

## Define the supported protocol

Write the limits and non-features in `notes/protocol-scope.md` before coding:

- HTTP/1.1 request headers must end with `\r\n\r\n`.
- Only one request is handled per connection; every response says `Connection: close`.
- This week only `GET` requests with no body are supported.
- Chunked transfer encoding, trailers, upgrades, and persistent connections are unsupported.
- Header bytes are capped at 16 KiB, header count at 64, method at 15 characters, and request target at 2047 characters.
- Header names compare case-insensitively; values are not silently lowercased.

An explicit small subset is safer than accidentally pretending to support all of HTTP/1.1.

## Milestone 1: Separate HTTP code

Create:

```text
week-02/
├── Makefile
├── notes/protocol-scope.md
├── src/
│   ├── http.c
│   ├── http.h
│   └── main.c
└── tests/
    └── test_http.c
```

In `http.h`, define bounded structures. A suitable shape is:

```c
#define HTTP_MAX_HEADERS 64

struct http_header {
    char name[64];
    char value[4096];
};

struct http_request {
    char method[16];
    char target[2048];
    char path[1024];
    char version[16];
    struct http_header headers[HTTP_MAX_HEADERS];
    size_t header_count;
};
```

Return a parser result that distinguishes at least `complete`, `incomplete`, and `malformed`. Do not make callers infer those states from one ambiguous integer.

Update the `Makefile` so application source files build into `authlab` and `make test` builds and runs `tests/test_http.c` against `src/http.c`.

## Milestone 2: Read a complete header block

Replace the single `recv` with an accumulation loop:

1. Keep a byte count separate from buffer capacity.
2. Search only the received bytes for the four-byte terminator `\r\n\r\n`.
3. Call `recv` again while the terminator is absent and capacity remains.
4. Return `400 Bad Request` if the peer closes early or the limit is reached without a terminator.
5. Never scan uninitialized memory and never assume received bytes contain a trailing NUL.

Implement a small length-aware delimiter search rather than using `strstr` on untrusted nonterminated bytes. Add a receive timeout, such as five seconds with `SO_RCVTIMEO`, and document that a production server needs a more scalable slow-client defense.

## Milestone 3: Parse without overrunning fields

Parse in this order:

1. Find the first CRLF and split the request line into exactly three nonempty fields.
2. Reject unsupported versions; accept only `HTTP/1.1` for this lab.
3. Reject a method or target that cannot fit its destination including the NUL terminator.
4. Require the target to begin with `/`; split its path from an optional `?query` for routing.
5. Parse each subsequent header at its first colon.
6. Reject a missing/empty header name, whitespace before the colon, control bytes, too many headers, or overlong fields.
7. Trim optional spaces/tabs around the header value.
8. Require exactly one nonempty `Host` header for HTTP/1.1.

Avoid `strtok`: it hides empty fields, mutates global parsing state, and does not enforce your byte limits clearly. It is fine to copy a validated span into fixed-size fields with an explicit length check.

Add a lookup function such as:

```c
const char *http_get_header(const struct http_request *request,
                            const char *name);
```

Use `strcasecmp` or an equivalent ASCII case-insensitive comparison for header names.

## Milestone 4: Centralize responses and route requests

Create one response helper that accepts status, content type, body pointer, and body length. It must calculate `Content-Length`, use CRLF, call the Week 1 `send_all`, and add `Connection: close`.

Implement this routing table:

| Request               | Result                                     |
| --------------------- | ------------------------------------------ |
| `GET /`               | `200` and `Hello, world!`                  |
| `GET /hello`          | `200` and `Hello from authlab!`            |
| `GET /does-not-exist` | `404 Not Found`                            |
| `POST /`              | `405 Method Not Allowed` plus `Allow: GET` |
| malformed input       | `400 Bad Request`                          |

Do not route by prefix: `/hello-there` is not `/hello`.

## Parser tests

Make `tests/test_http.c` call the parser directly with byte arrays. Include at least:

- a valid request with two headers;
- header-name case variation (`hOsT`);
- a target containing a query string;
- incomplete headers;
- bare LF separators;
- missing method, target, or version;
- missing `Host`;
- more than 64 headers;
- an overlong method, target, header name, and header value;
- an embedded NUL or control byte;
- a header with no colon.

Run after every parser change:

```bash
make -C week-02 test
```

Run once with sanitizers:

```bash
make -C week-02 clean
make -C week-02 test CFLAGS='-std=c11 -g -Wall -Wextra -Wpedantic -Werror -fsanitize=address,undefined'
```

## Protocol and attack lab

Start the normal build, then exercise the network boundary:

```bash
make -C week-02 clean all
week-02/authlab
```

From another terminal:

```bash
curl -i http://127.0.0.1:8080/
curl -i http://127.0.0.1:8080/hello
curl -i http://127.0.0.1:8080/does-not-exist
curl -i -X POST http://127.0.0.1:8080/
printf 'BROKEN\r\n\r\n' | nc 127.0.0.1 8080
printf 'GET / HTTP/1.1\r\nHost localhost\r\n\r\n' | nc 127.0.0.1 8080
```

Record the exact status line for each request. Then send a header larger than 16 KiB and verify a `400` response or clean connection close, never a crash.

## End-of-week working state

At the end of Week 2, you should have a warning-free server that accumulates bounded header bytes, parses a narrowly documented HTTP/1.1 subset, and routes `GET /`, `GET /hello`, and unknown paths correctly. Malformed requests result in `400`, unsupported methods in `405`, and missing routes in `404`.

Verify:

```bash
make -C week-02 clean all test
week-02/authlab & server_pid=$!
curl --retry 20 --retry-connrefused --retry-delay 0 --retry-max-time 5 --silent --output /dev/null http://127.0.0.1:8080/
test "$(curl --silent --output /dev/null --write-out '%{http_code}' http://127.0.0.1:8080/hello)" = 200
test "$(curl --silent --output /dev/null --write-out '%{http_code}' http://127.0.0.1:8080/missing)" = 404
test "$(printf 'bad\r\n\r\n' | nc 127.0.0.1 8080 | head -n 1 | tr -d '\r')" = 'HTTP/1.1 400 Bad Request'
kill "$server_pid"
```

Write down why TCP segmentation means the end of one `recv` call is not the end of an HTTP request. This directory is the baseline for Week 3.
