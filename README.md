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

## Optional: Authenticated Origin Pulls (Cloudflare origins)

If this site sits behind Cloudflare and you already restrict inbound 80/443
to Cloudflare's IP ranges (e.g. with
[cf-origin-firewall](https://github.com/1chunghu/cf-origin-firewall)), one gap
remains: an IP allowlist verifies *who delivered the packet*, not *whose zone
it came through* — anyone can point their own Cloudflare zone at your origin
IP and arrive from the same edge ranges, skipping your WAF and rate limits.
Authenticated Origin Pulls (mTLS) closes most of that gap by making nginx
verify Cloudflare's client certificate on every origin pull.
Companion write-up (zh-TW):
[鎖了 IP 還不夠](https://1chung.net/blog/authenticated-origin-pulls/)

1. Cloudflare Dashboard → SSL/TLS → Origin Server → **Authenticated Origin
   Pulls** → enable the zone-wide switch. The free tier uses Cloudflare's
   shared client certificate — nothing to generate or upload.
2. Fetch the CA and place it where the *container* can actually read it:
   <https://developers.cloudflare.com/ssl/static/authenticated_origin_pull_ca.pem>
   (current CA expires **2029-11-01**; set a reminder to re-fetch).
3. Uncomment the `ssl_client_certificate` / `ssl_verify_client` lines in
   `nginx.conf`. Unlike `add_header`, these inherit normally — declared once
   at http level, they cover every server block.
4. Verify both directions: a request to port 443 **without** a client
   certificate must fail with `400` ("No required SSL certificate was sent");
   requests through Cloudflare must stay `200`.

Scope note: the shared certificate proves the connection came from a
Cloudflare edge — not that it came through *your* zone (an attacker can enable
AOP on their own zone and present the same shared cert). Zone-level custom
certificates close that too; either way, keep the unknown-Host catch-all
(`00-default.conf` → `444`) and treat AOP as one layer, not a silver bullet.

**Two deployment gotchas, learned the hard way:**

1. **Know exactly which host directory your cert mount maps to.** Two
   similarly named `certs/` directories on the host, one mounted into the
   container and one not — the CA landed in the wrong one, and nginx went
   into a crash-restart loop on the next restart, taking every site down.
2. **`docker exec nginx nginx -t` can pass against a stale file.** When the
   config is a single-file bind mount and you overwrite it with `cp`/`sed -i`,
   a new inode is created — but the running container still reads the old one,
   so the "syntax is ok" validated the *previous* config, not your change.
   Only a `-t` re-run *after* `docker restart` proves anything; or test the
   new file in a throwaway container before touching production.

## Note

These security headers are, by design, fully public — every visitor's browser
already reads them on every response. Publishing the config leaks nothing an
attacker couldn't see with `curl -I`. Real infrastructure details (internal IPs,
management ports, backend service names) are **not** in these files and should
never be committed.

## License

[MIT](LICENSE)
