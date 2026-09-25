# 12. Camera-wall login exception by client address

## Status

Accepted

## Context

This ADR covers only the `basic_auth` login in front of the camera wall
(`kamery.{{ domain_name }}`, [ADR-0011](0011-live-camera-view-via-go2rtc.md))
and one exception to it: a wall kiosk that has to show the page with nobody
there to type a password. It does not change the path allowlist, the final
403, or how any other service is exposed.

The kiosk lives in its own repository, `ansible-kiosk-template`, whose
ADR-0002 decides that the kiosk stores no credentials and gets in by address
instead, with the address pinned by a DHCP reservation in
`ansible-network-template`. This repository has to implement the Caddy side.

An address exception is only meaningful if Caddy sees the kiosk's real
address. Caddy runs in Docker on a bridge network with published ports, so
that had to be measured, not assumed. On a VM matching production (AlmaLinux
9, Docker CE 29.8.1, firewalld backend, userland proxy on):

- a LAN client over IPv4 reaches Caddy through DNAT, and Caddy's
  `remote_ip` is the client's real address;
- a connection from the Docker host itself (localhost, IPv4 or IPv6) goes
  through `docker-proxy`, and Caddy sees the gateway of the `caddy-ingress`
  network (e.g. `172.18.0.1`); an IPv6 client from outside would take the
  same path when the Docker network has no IPv6;
- a request from another container (`cloudflared`) comes from that
  container's address.

The kiosk sits in VLAN `main` and the host in VLAN `servers`; the router
masquerades only on the WAN, so inter-VLAN traffic keeps its source address,
and the LAN has no IPv6.

## Decision

- **`camera_wall_trusted_ips`**, a list of plain IPv4 addresses, empty by
  default. Empty means no exception: the block renders exactly as before.
- **The exception covers the login and nothing else.** In the `@kamery`
  block a named matcher `@kamery_login not remote_ip <list>` is passed to
  `basic_auth`, so the login applies to every request *not* from a listed
  address. The path allowlist and the final `respond 403` are unchanged and
  apply to trusted addresses too: `/api/streams` and `/api/config` stay 403.
- **`remote_ip`, not `client_ip`.** `remote_ip` is the TCP peer as Caddy
  sees it; it ignores `X-Forwarded-For`, so a client cannot claim the kiosk's
  address in a header.
- **Docker addresses are refused by the `caddy` role.** Before the
  Caddyfile is rendered, an assert rejects anything that is not a single IPv4
  address, loopback, `172.16.0.0/12` (Docker's default pools), and every
  gateway of every Docker network actually present on the host. The gateway
  is the dangerous one: listed, it would let everything Docker proxies —
  host-local and IPv6 traffic — skip the login.

## Consequences

- The kiosk opens the camera wall with no password stored anywhere; every
  other client still logs in. Verified on the test VM with the rendered
  Caddyfile: the trusted address gets 200 for the page and `/api/ws` and 403
  for `/api/streams` and `/api/config`; any other address gets 401; a forged
  `X-Forwarded-For` changes nothing; an empty list renders a valid config
  that asks everyone to log in.
- Whoever takes over the trusted address on the LAN can watch the cameras
  without a password — but gets nothing more than that (see
  `ansible-kiosk-template` ADR-0002). The address must stay pinned by the
  DHCP reservation in `ansible-network-template`.
- The exception rests on two network facts outside this repository: no
  inter-VLAN masquerade and no IPv6 on the LAN. If either changes, Caddy may
  start seeing a router or gateway address instead of the kiosk's, and the
  measurement has to be repeated before the exception can be trusted.
- The assert runs `docker network ls` / `inspect` on every run with a
  non-empty list — read-only, `changed_when: false`, also in check mode.
