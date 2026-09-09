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

The CSP is deployed as `Content-Security-Policy-Report-Only`, and HSTS as
`Strict-Transport-Security "max-age=300"` with no `includeSubDomains` and no
`preload`.

## Consequences

- The proxied apps are completely unaffected — no CSP, no new headers, no
  behavior change. The headers protect the one handler that can safely carry
  them and nothing else. The cost is that this is not a blanket policy: any
  future service that should have security headers needs them added to its
  own handler explicitly.
- CSP starts in Report-Only: violations are reported (once a `report-to`
  sink is wired up) but nothing is blocked, so a mistake in the policy
  cannot take the site down. Switching to the enforcing
  `Content-Security-Policy` header is a separate, deliberate change to be
  made only after the Report-Only phase shows the policy is clean.
- HSTS `max-age` is intentionally low (300s) and grows in stages. A too-high
  `max-age` shipped by mistake pins every visitor's browser to HTTPS-only
  for that duration with no way to retract it server-side; ramping up
  (300s -> hours -> days -> a year) keeps the blast radius of a TLS
  misconfiguration small while confidence builds.
- `preload` is deliberately omitted. Adding the domain to the browser
  preload list is easy; removal takes months to propagate through browser
  releases. Preload is a one-way door not worth walking through for a
  homelab domain at this stage.
- `includeSubDomains` is omitted for the same reason — it would force HSTS
  onto every subdomain (Grafana, Vaultwarden, the LAN-only device proxies)
  as a side effect of a header set for the apex static site.
