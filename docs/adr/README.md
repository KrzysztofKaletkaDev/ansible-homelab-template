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
