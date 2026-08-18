# 7. Blocky's DNS and metrics ports opened to the LAN

## Status

Accepted

## Context

Blocky must answer DNS queries from every device on the local network,
which means, unlike every other service in this stack, it cannot be
reached exclusively through Caddy's internal `caddy-ingress` Docker
network — DNS resolution has to reach the container directly, on the
host's own network. The `blocky` role therefore opens `53/udp` and
`53/tcp` in `firewalld`, `permanent` and `immediate`, with no source
restriction, so any device on the LAN can query it.

The same role also opens `4000/tcp` for Blocky's own Prometheus metrics
endpoint, which Prometheus scrapes over the host network
(`scrape :4000 on host`, per the architecture diagram) since it isn't
reachable through `caddy-ingress` either. This port carries no secrets,
but it is still an unauthenticated HTTP endpoint reachable by anything on
the LAN, not just by the Prometheus container.

## Decision

Accept both ports open on the default `firewalld` zone for the local
network, with no additional scoping, rather than building a mechanism to
restrict them.

## Consequences

- Any device on the LAN — not just the router doing legitimate DNS lookups
  or the local Prometheus instance — can query Blocky's DNS resolver and
  scrape its metrics endpoint. In a single-operator homelab where the LAN
  itself is the trust boundary, this is judged an acceptable exposure.
- This is real exposure beyond "no unnecessary host exposure," and the
  README is written to say so explicitly rather than imply Blocky is as
  isolated as Vaultwarden or the monitoring stack.
- The concrete alternative not taken: a `firewalld` **rich rule** scoping
  both ports to the known LAN source subnet (e.g.
  `rule family="ipv4" source address="192.168.0.0/24" port port="53" protocol="udp" accept"`)
  instead of the default zone/port opening used today. This was left as a
  documented option rather than implemented, since the current single-LAN
  deployment model does not yet need it.
- If this template is ever deployed on a host reachable from an untrusted
  network segment, this decision needs to be revisited before that
  happens — it is safe under the LAN-only trust model assumed by the rest
  of this repo, and not automatically safe outside it.
