# Week 14: TLS and secure transport

## Goal

Keep the C server on loopback, terminate HTTPS in a mature reverse proxy, set transport-dependent cookie policy, and observe that TLS protects HTTP bytes while they cross the client/proxy boundary.

Time budget: 5-8 hours.

## Start here

Begin with the completed Week 13 local application:

```bash
mkdir -p week-14/notes week-14/var
cp -R week-13/src week-13/tests week-13/tools week-13/Makefile week-13/schema.sql week-14/
make -C week-14 clean all test
```

Your starting auth system still transmits passwords, cookies, CSRF tokens, and reset tokens as plaintext HTTP bytes. It is safe only because it is restricted to loopback. Do not expose it directly to a network.

Install a TLS-capable proxy rather than adding TLS code to C:

```bash
brew install caddy
caddy version
```

## Understand the boundary

Draw in `notes/tls-boundary.md`:

```text
browser/curl
  | HTTPS to localhost:8443 (encrypted, integrity-protected, server authenticated)
Caddy
  | HTTP to 127.0.0.1:8080 (loopback only)
authlab
```

TLS does not make the endpoint logic correct, encrypt data in process memory/database, prevent XSS, or stop a valid credential from being replayed. It protects transport between TLS peers.

## Milestone 1: Configure local TLS termination

Create `week-14/Caddyfile`:

```caddyfile
https://localhost:8443 {
    tls internal
    reverse_proxy 127.0.0.1:8080
}
```

Keep authlab bound to `127.0.0.1:8080`. Start it from `week-14`, then in another terminal:

```bash
cd week-14
caddy run --config Caddyfile
```

Caddy's internal CA is for local development. Run `caddy trust` before the formal verification so curl and the browser can authenticate it; this may require you to approve an OS privilege prompt directly. Alternatively, use curl's `--insecure` only for the initial local observation below, never for the end-of-week gate and never as a general fix for certificate errors.

Inspect:

```bash
curl -vk https://localhost:8443/
```

Identify the TLS negotiation separately from the HTTP request/response printed after it.

## Milestone 2: Give the application a public origin

Add configuration such as:

```text
AUTHLAB_PUBLIC_ORIGIN=https://localhost:8443
AUTHLAB_SECURE_COOKIES=1
```

Parse configuration once at startup and fail on invalid values. Use the public origin for exact `Origin` checks and absolute development reset URLs. Keep form actions and redirects relative where possible.

Do not infer security from arbitrary client-controlled `Host`, `Forwarded`, or `X-Forwarded-Proto` headers. In this fixed lab deployment, explicit configuration is the trusted source.

## Milestone 3: Enable Secure cookies

When secure-cookie mode is enabled, every authentication cookie must include:

```http
Secure; HttpOnly; SameSite=Lax; Path=/
```

The cookie-deletion response must use matching `Path` and `Secure` attributes. CSRF remains required; HTTPS and `Secure` do not express user intent.

Use a clean browser profile or cookie jar:

1. Log in through `https://localhost:8443`.
2. Inspect `Set-Cookie` and confirm `Secure`.
3. Access `/profile` over HTTPS and confirm success.
4. Try the backend directly at `http://127.0.0.1:8080/profile`; the browser must not attach the Secure cookie.
5. Confirm logout over HTTPS invalidates the server row.

Do not place session values in notes or screenshots.

## Milestone 4: Revisit proxy-sensitive security

The TCP peer seen by authlab is now Caddy (`127.0.0.1`), so naïve per-IP rate limiting would group every proxied user together.

For the local lab, choose and document one of these:

- continue treating all traffic as loopback and acknowledge per-client IP limits are unavailable; or
- trust a forwarded client-IP header only because the backend is loopback-only and only the configured proxy is allowed to reach it.

If implementing the second option, reject malformed/multiple values according to a strict policy and never trust forwarded headers when authlab can be reached by untrusted peers. Trusted-proxy configuration is part of the security boundary.

## Transport experiments

Use `curl -v` against direct HTTP and `curl -vk` against HTTPS. Compare what each client reports. Optionally use Wireshark/tcpdump on loopback to contrast:

- client to Caddy on port 8443: TLS records, not readable HTTP fields;
- Caddy to authlab on port 8080: readable HTTP on the trusted local hop.

Do not capture other users' traffic or non-lab interfaces.

Temporarily present a different/untrusted local certificate and observe verification failure. Restore Caddy's trusted local certificate. Explain that encryption without authenticated server identity can still connect a client to the wrong endpoint.

## Regression checks

Run all auth workflows through the HTTPS origin:

- registration/login/profile;
- role authorization;
- CSRF rejection and success;
- logout/replay failure;
- password change/reset URL generation;
- throttling behavior.

Update integration tests to accept a base URL and CA/testing option rather than hardcoding port 8080. Keep a direct-backend test proving the listener remains `127.0.0.1` only.

## End-of-week working state

At the end of Week 14, Caddy serves `https://localhost:8443` with a locally trusted/test certificate and proxies to authlab on loopback. Session cookies carry `Secure`, public-origin checks use HTTPS, and all prior security tests pass through the proxy.

Verify in separate terminals:

```bash
cd week-14
AUTHLAB_PUBLIC_ORIGIN=https://localhost:8443 AUTHLAB_SECURE_COOKIES=1 ./authlab
```

```bash
cd week-14
caddy run --config Caddyfile
```

Then:

```bash
curl --fail-with-body https://localhost:8443/
lsof -nP -iTCP:8080 -sTCP:LISTEN
sh week-14/tests/test_https.sh
```

The listener check must show `127.0.0.1:8080`, never `*:8080`. Be able to identify exactly which hop is encrypted and why `Secure` has meaning only when the browser uses HTTPS. This directory is copied into Week 15.
