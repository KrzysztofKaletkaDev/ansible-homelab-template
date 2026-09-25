# Architecture Decision Records

This directory records the significant architectural decisions behind this
homelab template, in a lightweight MADR-derived format (Status / Context /
Decision / Consequences). Each ADR documents a decision already made and in
force in the code — not a proposal.

| # | Title | Status |
|---|-------|--------|
| [0001](0001-cloudflare-tunnel-over-port-forwarding.md) | Cloudflare Tunnel instead of port forwarding | Accepted |
| [0002](0002-locally-managed-tunnel-over-dashboard.md) | Locally-managed tunnel configuration instead of the Cloudflare dashboard | Accepted |
| [0003](0003-caddy-file-server-for-static-site.md) | Caddy `file_server` instead of a dedicated nginx container | Accepted |
| [0004](0004-separate-repos-for-iac-and-site-content.md) | Separate repositories for infrastructure and site content | Accepted |
| [0005](0005-cadvisor-version-pin.md) | cAdvisor pinned to v0.60.5 from ghcr.io | Accepted |
| [0006](0006-remove-host-network-panel-node-exporter.md) | Removed the host network panel from the node_exporter dashboard | Accepted |
| [0007](0007-blocky-lan-exposed-ports.md) | Blocky's DNS and metrics ports opened to the LAN | Accepted |
| [0008](0008-grafana-dashboards-as-code.md) | Grafana dashboards provisioned as code with `allowUiUpdates: false` | Accepted |
| [0009](0009-www-to-apex-redirect-in-caddy.md) | www-to-apex redirect in Caddy instead of a Cloudflare Redirect Rule | Accepted |
| [0010](0010-security-headers-scoped-to-static-site.md) | Security headers scoped to the static site handler | Accepted |
| [0011](0011-live-camera-view-via-go2rtc.md) | LAN-only live camera view via go2rtc behind Caddy | Accepted |
| [0012](0012-camera-wall-login-exception-by-client-address.md) | Camera-wall login exception by client address | Accepted |
