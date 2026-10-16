# asgiwarden — One hardening kit for FastAPI and Starlette

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/asgiwarden**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

Security headers, the real visitor IP behind Cloudflare, a rate limit, request ids, log redaction and throttled error alerts, installed in the right order with one harden(app) call.

## Get it

Buy asgiwarden at https://dominares.org/tools/asgiwarden: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python package and the asgiwarden command, with an example app, a before-and-after script and the full test suite.

Requirements: Python 3.10 or newer and Starlette, which installs with it. FastAPI is optional.

This page documents version 1.0.2. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

`examples/before_after.py` shows the whole kit on a live app without starting a server; it needs the `dev` extra (the test dependencies).

### What it does

`harden(app)` installs, in the one order that works, on any FastAPI or Starlette app:

| Layer | What it does |
|---|---|
| Request id and error handler (outermost) | Every response carries `X-Request-ID`; every log line can carry `rid=`; an uncaught exception becomes a clean 500 with that id and one throttled alert. |
| Client IP | Works out the visitor's real address and publishes it as `request.state.client_ip`. Behind Cloudflare it trusts `CF-Connecting-IP` only after checking that the request really came from a Cloudflare edge. |
| Rate limit (if you give rules) | Sliding window by IP and by session id, per path prefix. Over the limit: `429` with `Retry-After`. |
| Page headers | On `text/html` responses only: `X-Frame-Options: DENY` plus `frame-ancestors 'none'`, `nosniff`, `Referrer-Policy`, `Strict-Transport-Security`. A route's own value is never overwritten. |

and then gives every GET-only route a HEAD, so `curl -I` and uptime monitors stop getting 405.

```python
from fastapi import FastAPI
from asgiwarden import RateLimitRule, configure_logging, harden

app = FastAPI()
# ... routes and include_router calls ...

configure_logging("myapp", "logs")           # redacting handlers; optional rotating file
harden(
    app,
    service="myapp",
    on_alert=lambda payload: notify(payload),  # your email, chat or pager call
    frameable_paths=["/embed/chat"],           # pages any other site may frame
    rate_limit_rules=[RateLimitRule("/api/chat", ip=(30, 60), session=(10, 60))],
    trusted_hops=1,                            # exactly how many proxies you control in front of the app (0 if none)
)
```

Call `harden` after the last `include_router` and before the app serves its first request.

#### Headers

Why a default and not a list per route: the next rewrite of a route forgets the list, and the
sign-in page can then be loaded in an invisible iframe on another site. The defaults skip scripts,
stylesheets and downloads, where `nosniff` plus a guessed type could stop a file loading.

A page that sends a `frame-ancestors` naming another site is refused framing, unless its path is in
`frameable_paths`. `frame_policy("'self'")` gives the two headers for a page your own site frames.
`harden_html_headers(headers, path, hsts=..., referrer_policy=...)` is the function underneath.
HSTS is `max-age=31536000` with no `preload` and no `includeSubDomains`; pass your own string, or
`None` to leave it out, as `harden(app, hsts=...)` (`referrer_policy=` works the same way).

#### Content-Security-Policy from the page itself

```python
from asgiwarden import build_csp, csp_html_response

csp_html_response(html, frame_ancestors="'none'")      # an HTMLResponse whose CSP matches its own body
build_csp(html, script_hosts=["https://cdn.example"], style_unsafe_inline=True)   # just the string
```

`script-src` is `'self'` plus one `'sha256-...'` per inline script. The defaults name nothing but
`'self'`; each extra host or `'unsafe-inline'` for styles is something you pass in. `report_only=True`
sends the report-only header so you can watch before you enforce. From the command line:
`asgiwarden csp-hash page.html` (or `-` for stdin), with `--frame-ancestors`, `--script-host` and
`--style-unsafe-inline`. The command line defaults `--frame-ancestors` to `'none'`, the same as the
headers layer; `build_csp()` in Python defaults to `'self'`, so pass `frame_ancestors=` explicitly there.

#### The real client IP

```python
from asgiwarden import client_ip, resolve_client_ip
client_ip(request)    # or request.state.client_ip once harden() ran
resolve_client_ip(xff="203.0.113.5, 162.158.91.92", cf_connecting_ip="203.0.113.5")
```

The rule: the only address a visitor cannot forge is the one your own proxy appended, the rightmost
`X-Forwarded-For` entry (`trusted_hops` counts proxies you control; `0` means none, and then the TCP peer
is used and `X-Forwarded-For` is ignored). If that entry is a Cloudflare edge, `CF-Connecting-IP` wins;
otherwise the header is ignored, so nobody can forge and rotate it to get unlimited rate-limit buckets
(provided `trusted_hops` matches your real proxies; with no proxy in front, pass `trusted_hops=0`).
Pass `trust_cloudflare=False` if you are not behind Cloudflare at all.

**Refreshing the Cloudflare ranges.** The ranges are bundled (fetched 2026-10-04) and never fetched at
request time. When Cloudflare publishes new ones:

```bash
asgiwarden refresh-cloudflare --out cloudflare-ranges.json     # fetch, validate, write
asgiwarden refresh-cloudflare --check                          # exit 1 if the bundled list is stale
asgiwarden refresh-cloudflare --check --file cloudflare-ranges.json
```

then load the file at start-up: `harden(app, cloudflare_ranges=CloudflareRanges.from_file("cloudflare-ranges.json"))`.
A fetched page that is not a list of networks (an error page, a truncated list) is refused and the old
file is left untouched. `--check` suits a nightly CI job.

#### Rate limit

`RateLimitRule(path_prefix, ip=(max_events, window_seconds), session=(...), methods=None)`; the first
rule whose prefix matches applies. The session id is read from the `x-session-id` header, or from a
cookie with `session_cookie="sid"`; a request that names no session is limited by IP only. The pieces
are also usable alone: `SlidingWindowLimiter(max_events, window_seconds).hit(key)` (thread-safe, bounded
number of keys) and the pure `evaluate_rate_limit` / `record_request` over a dict you hold.

#### Request ids, redaction and alerts

`configure_logging(name, logs_dir=None, json_lines=False)` replaces the root handlers with ones that
redact every line, including tracebacks: secrets in URL query strings (`key=`, `token=`, `password=` ...),
`Authorization: Bearer` and `x-api-key` headers, and bare Google, OpenAI-style and Telegram bot keys.
`scrub_message(text)` also masks emails and phone-like numbers; `mask_pii(text)` masks emails, phone
numbers and long digit runs and cuts text to a short prefix, for logging what a person typed.

An incoming `X-Request-ID` is kept only if it is 8 to 64 characters of letters, digits and `._:-`;
anything else is replaced. On an uncaught exception `on_alert(payload)` gets the service, exception
class, route template (`/items/{item_id}`, never the real path), method, request id and a scrubbed
message. Never the body, headers or query string. One alert per distinct error per 15 minutes and at most
6 an hour per process (`ErrorAlertThrottle(window_seconds, max_per_hour)`); repeats held back are counted
into the next alert. The callback may be a plain function or an `async def`. The visitor sees JSON on
`/api/` paths and for JSON clients, a small page otherwise (`error_page=` to replace it).

#### Command line

| Command | Does |
|---|---|
| `asgiwarden csp-hash PAGE.html` | Print the CSP for a page; exit 1 on inline event handlers. |
| `asgiwarden refresh-cloudflare [--out F] [--check [--file F]]` | Renew or verify the Cloudflare ranges. |
| `asgiwarden redact [--pii] < app.log` | Mask secrets (and with `--pii`, emails and phone numbers) in text from stdin. |
| `asgiwarden check URL` | GET and HEAD a URL; report a HEAD that fails, and missing frame policy, `nosniff`, HSTS (https only) or `X-Request-ID`. |

### Limits

- **Per process.** The rate limit and the alert throttle keep their state in memory: a restart resets
  them and N workers allow N times the limit. That keeps them independent of any database; use a shared
  store in front if you need an exact global limit.
- **Topology is yours to state.** `trusted_hops` must equal the number of proxies you control, exactly.
  Too high (or the default `1` with no proxy in front) reads an entry the client wrote, so a forged
  `X-Forwarded-For` mints a new rate-limit bucket on every request. Too low keys every visitor on your own
  proxy, so they share one bucket. An app reached directly, with no proxy, must pass `trusted_hops=0`.
  The "forged header cannot mint buckets" protection holds only when `trusted_hops` matches your proxies.
- **Cloudflare's list is third-party data.** The bundled copy is dated; a range added later is treated as
  a direct request: nothing can be forged, but every visitor behind that range is keyed on the edge address. Refresh it on a schedule.
- **HEAD** is added only to routes whose methods are exactly GET, including routes of included routers.
  It does not reach routes inside a mounted sub-application. Call `harden` before the first request is
  served: FastAPI builds its routing on first use.
- **CSP** is computed over the exact body you give it, and does not rewrite your pages. A page that
  changes its inline script per request needs the CSP computed per request (`csp_html_response`).
  Inline event handlers (`onclick=`) cannot be allowed by a hash; `csp-hash` reports them.
  Nonces are not supported.
- **Page headers** apply to `text/html` responses only. HSTS is sent regardless of scheme (browsers
  ignore it over plain http).
- **Redaction** knows the shapes listed above. It cannot find a secret in a shape it has not seen, and
  `mask_pii` targets US and international phone formats and plain emails. Treat both as a safety net
  under not logging secrets, not a replacement for it.
- **Alerts** reach you only through the callback you write. A streamed response that fails after it has
  begun cannot be replaced with a clean 500; the error is logged and alerted, and the connection closes.
- **Not a firewall.** It does not authenticate, validate input or filter attacks. It closes common gaps
  in headers, IP handling, throttling and logs.

Tested with Python 3.10 and 3.14, Starlette 0.27 and 1.7, and FastAPI 0.100 and 0.142. The only runtime
dependency is Starlette; `FastAPI` is needed for the examples and tests (`pip install ".[dev]"`).
Starlette-based frameworks other than FastAPI have not been tested.

---

Docs mirror of the private repository Dominares-Tools/asgiwarden; updated with each release.
