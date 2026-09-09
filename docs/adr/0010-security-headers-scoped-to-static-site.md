# 10. Security headers scoped to the static site handler

## Status

Accepted

## Context

This ADR covers only the HTTP security response headers
(HSTS, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`,
`Permissions-Policy`, CSP) added to the static portfolio site at the apex
`kaletkadev.com`, and specifically why they are applied in the `handle @root`
block alone rather than site-wide. It does not cover TLS configuration, the
DNS-01 challenge, or the headers of any proxied service (Grafana,
Vaultwarden, Homepage, QNAP, router) — those are left exactly as they are.

The obvious place to set security headers in a `Caddyfile` is the site-wide
block, so every hostname inherits them. That does not work here: a strict
`Content-Security-Policy` in the site-wide block would also apply to the
reverse-proxied apps. Grafana and Vaultwarden both ship their own inline and
bundled scripts and styles and would break under a `script-src 'self'` /
`style-src 'self'` policy tuned for a hand-built static site. Relaxing the
policy enough to keep them working would defeat the point of having it.

The static site, by contrast, serves a fixed, known set of assets and can
take a tight policy.

## Decision

Define a `(portfolio_headers)` snippet and `import` it as the first line of
`handle @root` (the apex static-site handler) only. It is not imported into
`@grafana`, `@vaultwarden`, `@qnap`, `@router`, `@homepage`, or the
site-wide block.

The CSP is enforced as `Content-Security-Policy` (it was rolled out as
`Content-Security-Policy-Report-Only` first — see Consequences). HSTS is
`Strict-Transport-Security "max-age=31536000"` (one year) with no
`includeSubDomains` and no `preload`. `Cache-Control "public, no-transform"`
is set so the Cloudflare edge cannot rewrite the response body and re-inject
scripts the CSP would block.

## Consequences

- The proxied apps are completely unaffected — no CSP, no new headers, no
  behavior change. The headers protect the one handler that can safely carry
  them and nothing else. The cost is that this is not a blanket policy: any
  future service that should have security headers needs them added to its
  own handler explicitly.
- CSP was rolled out Report-Only first: violations were reported but nothing
  was blocked, so a mistake in the policy could not take the site down. It
  ran that way until every reported violation traced back to edge-injected
  code rather than the repo; the move to the enforcing
  `Content-Security-Policy` header is that separate, deliberate change, made
  once the Report-Only phase showed the policy clean.
- HSTS `max-age` was raised in stages rather than starting at a year. It
  began at 300s so a bad TLS config could only pin browsers to HTTPS-only
  briefly with no server-side retraction; it moved to one year (31536000s)
  once the setup proved stable. A too-high `max-age` shipped by mistake is
  the failure this staging guards against.
- The policy assumes Cloudflare's edge does not inject code into the page.
  Turning on Web Analytics, Bot Fight Mode, or Rocket Loader in the
  dashboard will break the enforced CSP — each adds an inline script to the
  HTML that the origin never served in that form. `Cache-Control: no-transform`
  enforces that assumption from code; the dashboard toggles being off is not
  a sufficient guarantee on its own, since they live in the panel, not in
  Git, and anyone with dashboard access (including a future version of the
  operator) can flip them back.
- `no-transform` also disables Cloudflare's other edge transformations,
  including email address obfuscation — the `mailto:` link in the static
  site's `index.html` is now visible in plain text in the page source. This
  is an accepted cost: the alternative is a CSP that the edge can silently
  invalidate.
- HSTS is now `max-age=31536000` (one year), still without `includeSubDomains`
  and without `preload`. `includeSubDomains` stays off because planned k3s
  ingresses on subdomains may not have a valid certificate the moment their
  DNS record appears, and the apex header would otherwise force HTTPS-only on
  them from the start. `preload` stays off because removal from the browser
  preload list requires a submission and then rides out over subsequent
  browser releases — effectively irreversible on a homelab timescale.
