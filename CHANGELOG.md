# Changelog

## 1.0.2 (2026-10-04)

README only: it opens with the before-and-after script and leaves the install steps to the page you bought it from. No code change.

## 1.0.1 (2026-10-04)

- Fixed the `trusted_hops` guidance (README and docstring), which had the failure direction reversed:
  too HIGH, including the default `1` with no proxy in front, lets a client forge its IP and mint a
  new rate-limit bucket per request; too low keys every visitor on your proxy. Apps reached
  directly must pass `trusted_hops=0`. The example app now does, and tests cover the too-high case.
- `asgiwarden csp-hash` now flags `ONCLICK=` / `OnClick=` (attribute names are case-insensitive) and
  defaults `--frame-ancestors` to `'none'`, matching the headers layer.
- The clean HTML 500 page now follows `harden(hsts=..., referrer_policy=...)` instead of the defaults.
- Test fixtures use documentation addresses only.

## 1.0.0 (2026-10-04)

First release.

- Security headers on every HTML page: frame policy, `nosniff`, referrer policy, HSTS.
- A Content-Security-Policy built from the sha256 hashes of a page's inline scripts.
- HEAD answered like GET on FastAPI routes.
- The visitor's real IP behind Cloudflare, with the Cloudflare ranges bundled and a
  `refresh-cloudflare` command to renew them.
- A sliding-window rate limit by IP and by session.
- A request id on every log line, secrets and personal data scrubbed from logs, and throttled
  alerts for uncaught server errors.
- A `harden(app)` call that installs the lot in the right order.
