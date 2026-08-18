# 1. Cloudflare Tunnel instead of port forwarding

## Status

Accepted

## Context

The core node is a single AlmaLinux host that runs, alongside the reverse
proxy and dashboard, a self-hosted password manager (Vaultwarden). Exposing
any service to the Internet on this host traditionally means forwarding
ports 80/443 on the gateway router straight to the host and relying on the
host's own firewall and TLS stack as the only line of defense.

Vaultwarden is exactly the kind of service where that trade-off is
unattractive: a router misconfiguration, a firewalld rule change, or an
unpatched edge component would put a credentials store directly on the
public Internet's attack surface. The main motivation for choosing a
different ingress model was this single fact — not general aesthetic
preference for "zero trust" as a buzzword.

## Decision

Expose the domain to the Internet exclusively through a Cloudflare Tunnel
(`cloudflared`), which opens an outbound-only connection from the host to
Cloudflare's edge. No inbound port is opened on the router or in `firewalld`
for HTTP/HTTPS traffic; all public ingress traffic reaches Caddy through the
tunnel, not through a forwarded port.

## Consequences

- No inbound firewall rule or router port-forward exists for web traffic —
  removes an entire class of "forgot to close the port" and "router admin
  panel also exposed" mistakes.
- Availability of the site now depends on Cloudflare's edge and the
  `cloudflared` daemon's connectivity, not just on the host's own uptime —
  an outage or misconfiguration in the tunnel is a new single point of
  failure that a bare port-forward wouldn't have had.
- All public traffic is visible to Cloudflare in transit (it terminates at
  their edge before reaching the tunnel) — acceptable here since Cloudflare
  already sits in the TLS path for the DNS-01 challenge and DNS resolution,
  but it is a trust dependency worth naming rather than glossing over.
- The tunnel requires a one-time manual bootstrap outside of Ansible (see
  [ADR-0002](0002-locally-managed-tunnel-over-dashboard.md)) — this is a
  manual step every fresh deployment must remember, and it is not encoded
  in `site.yml` itself.
