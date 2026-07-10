# nginx-hardening-template

A minimal, production-tested nginx configuration for a static site that scores
**A+** on [securityheaders.com](https://securityheaders.com) and hits **100/100**
on Lighthouse — without an `unsafe-inline` script policy.

Distilled from a real personal-site deployment. Drop it in, replace the
`example.com` placeholders, and adjust the CSP to your own origins.

## What it does

| Layer | Control |
|-------|---------|
| Transport | HSTS (1 year), TLS 1.2/1.3 only, no weak ciphers |
| Framing | `X-Frame-Options: DENY` + CSP `frame-ancestors 'none'` (clickjacking) |
| Content | Strict CSP with **no `unsafe-inline` on script-src** — the whole point |
| Sniffing | `X-Content-Type-Options: nosniff` |
| Referrer | `strict-origin-when-cross-origin` |
| Features | `Permissions-Policy` denies camera / mic / geolocation / USB / payment … |
| Info leak | `server_tokens off` (no version in `Server:` header) |
| Host spoofing | catch-all `default_server` returns `444` on unknown Host |
| Caching | 1-year immutable cache for versioned assets, 7-day for root assets |

## The two non-obvious gotchas

**1. `add_header` does not inherit once a block redeclares it.** In nginx, the
moment a `server` or `location` block sets *any* `add_header`, it stops
inheriting the ones from the `http` block. So the site block re-lists the full
header set rather than relying on inheritance — a partial list silently drops
the rest. This is the single most common way "I set HSTS but it's missing on
some pages" happens.

**2. `unsafe-inline` is only removable if you externalise inline code.** The A+
grade hinges on `script-src 'self'` with no `'unsafe-inline'`. That requires
moving every inline `<script>` and inline event handler (`onload=...`) out to
external `.js` files. `style-src` here keeps `'unsafe-inline'` because the site
uses inline `style="width:…"` for data-bar widths; that does not affect the
grade. If your markup has no inline styles either, drop it for a perfect policy.

## Files

| File | Role |
|------|------|
| `nginx.conf` | http-level baseline: `server_tokens off`, baseline headers, logging |
| `conf.d/00-default.conf` | catch-all `default_server` → `444` on unknown Host |
| `conf.d/site.conf` | the site: HTTPS redirect, full header set, CSP, cache rules |

## Usage

1. Replace every `example.com` with your domain, and fix the cert paths.
2. Edit the CSP `script-src` / `connect-src` to match your real origins
   (self-hosted analytics, any API your frontend calls) — or tighten to
   `'self'` only if you have none.
3. `nginx -t` to validate, then reload.
4. Verify at [securityheaders.com](https://securityheaders.com) and in your
   browser devtools (watch for `securitypolicyviolation` console events).

## Note

These security headers are, by design, fully public — every visitor's browser
already reads them on every response. Publishing the config leaks nothing an
attacker couldn't see with `curl -I`. Real infrastructure details (internal IPs,
management ports, backend service names) are **not** in these files and should
never be committed.

## License

[MIT](LICENSE)
