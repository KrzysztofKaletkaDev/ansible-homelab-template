# 3. Caddy `file_server` instead of a dedicated nginx container

## Status

Accepted

## Context

The static portfolio site (`kaletkadev.com`) needs nothing beyond serving
pre-built HTML/CSS/JS files over HTTPS. Caddy is already deployed on this
host as the reverse proxy for every other service and already terminates
TLS for the domain. Running a second web server (e.g. nginx) purely to
serve static files would duplicate a capability Caddy already has built in
(`file_server`), for a workload that has no need for a separate process,
its own image build, or its own container lifecycle.

## Decision

Serve the static site directly from the existing Caddy container using its
`file_server` directive, with the site's build output bind-mounted
read-only into the container (`{{ portfolio_dir }}/site:/srv/portfolio:ro`).
No dedicated nginx (or other static-file) container is introduced.

## Consequences

- One fewer container, image, and Compose service to build, update, and
  monitor — less surface area for the "which image tag is this pinned to"
  problem that already bit this repo once with cAdvisor.
- The static site's availability is now coupled to Caddy's — if Caddy is
  down for a reload or rebuild, the portfolio site goes down with it,
  whereas a separate nginx container would have kept serving independently.
- The site's content must exist on disk *before* Caddy's Compose stack
  starts, because the bind mount is read at container start, not
  request time — this is why the `portfolio` role must run before `caddy`
  in `site.yml`, a real ordering constraint documented in `CLAUDE.md`.
- No independent scaling or resource isolation for the static site; it
  shares Caddy's container resource limits (or lack thereof) rather than
  having its own.
