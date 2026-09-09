# 9. www-to-apex redirect in Caddy instead of a Cloudflare Redirect Rule

## Status

Accepted

## Context

This ADR covers only how `www.kaletkadev.com` is redirected to the apex
`kaletkadev.com`: where that redirect is executed and why. It does not
revisit the choice of a Cloudflare Tunnel for ingress
([ADR-0001](0001-cloudflare-tunnel-over-port-forwarding.md)) or the
locally-managed tunnel config
([ADR-0002](0002-locally-managed-tunnel-over-dashboard.md)) — it extends
those decisions to a new case, the `www` hostname, which previously had no
DNS record at all and so returned an error once one was added and traffic
started reaching the tunnel with no matching ingress rule.

The apex already serves the static portfolio site directly from Caddy
([ADR-0003](0003-caddy-file-server-for-static-site.md)). We want exactly one
canonical hostname, so `www` must redirect to the apex rather than serve a
second copy of the site.

Cloudflare offers a **Redirect Rule** that would perform this redirect at
its edge: faster for the client (one fewer round trip to the origin) and it
would terminate before the request ever entered the tunnel or reached the
host. The alternative is a `redir ... permanent` in Caddy's `Caddyfile`,
reached through the tunnel like every other request.

## Decision

Perform the `www` -> apex redirect in Caddy. `roles/cloudflared` gains an
explicit ingress entry for `www.{{ domain_name }}` pointing at
`https://caddy:443` (not a wildcard — see below), and
`roles/caddy` gains a `@www` matcher with a `redir https://{{ domain_name }}{uri} permanent`
handler, placed before the catch-all `handle` fallback.

A Cloudflare Redirect Rule is not used.

## Consequences

- The redirect is version-controlled in this repo and applied through the
  same Ansible run as every other routing rule, consistent with ADR-0002's
  premise that ingress routing lives in Git and not in a web UI that drifts
  silently. A Redirect Rule would have been a second source of truth for
  routing, outside review and outside `git`.
- The honest cost: the redirect now travels the full path — Cloudflare edge
  -> tunnel -> `cloudflared` container -> Caddy — and back, instead of being
  answered at the edge. For a permanent redirect that browsers cache this is
  a negligible latency hit, but it is a real one, and it means `www` is
  unavailable whenever the tunnel or the host is down, exactly like the apex.
- The `www` DNS route (`cloudflared tunnel route dns <tunnel-name> www.<domain>`)
  is one more manual bootstrap step outside Ansible, in the same category as
  the tunnel login/create/route steps already documented in ADR-0002 — the
  role renders the ingress rule but cannot create the DNS record. The README
  bootstrap section is updated to list both records.
- The ingress entry for `www` is a single explicit hostname, deliberately
  not a wildcard `*.{{ domain_name }}`. A wildcard in the tunnel ingress
  would publicly expose Vaultwarden, Grafana, the QNAP NAS and the router,
  which are today reachable only from the LAN via Blocky rewrites. ADR-0001's
  whole argument rests on the public ingress list being explicit and narrow;
  this change keeps it that way.
