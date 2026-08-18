# 5. cAdvisor pinned to v0.60.5 from ghcr.io

## Status

Accepted

## Context

A deployment attempt with an unpinned/older assumption about cAdvisor
failed outright: the host's Docker installation uses the containerd
snapshotter for storing image layers, and cAdvisor versions prior to
`v0.54.0` cannot read container storage metrics under that layout
(upstream issue [google/cadvisor#3643](https://github.com/google/cadvisor/issues/3643),
fixed in [PR #3709](https://github.com/google/cadvisor/pull/3709)). The
first deploy also failed by trying to pull a non-existent
`ghcr.io/google/cadvisor:v0.49.1` — cAdvisor moved its published image
registry from `gcr.io` to `ghcr.io` starting at `v0.53.0`, so an image
reference combining an old version number with the old registry (or vice
versa) simply does not exist.

## Decision

Pin `cadvisor_version` to `v0.60.5` in `group_vars/all/vars.yml.example`
and pull the image from `ghcr.io/google/cadvisor` (not `gcr.io`). This
version and registry combination is deliberate and satisfies both
constraints: it postdates the containerd-snapshotter fix (`>= v0.54.0`)
and postdates the registry move (`>= v0.53.0`).

## Consequences

- Container per-container metrics in Grafana are correct and complete
  under this host's containerd-backed Docker setup — the concrete failure
  mode that motivated the pin does not recur.
- The version is a hard-coded default, not "whatever is latest" — upgrading
  cAdvisor requires a deliberate, informed change to
  `group_vars/all/vars.yml` (and, per the pitfall documented in
  `CLAUDE.md`, to the real ignored `vars.yml` on the controller, not just
  the `.example` file), not an automatic pickup of new releases.
- Anyone tempted to "clean up" this version pin or registry as unnecessary
  hard-coding needs to know the specific upstream issue/PR behind it —
  this ADR and the inline context in `CLAUDE.md` exist so that knowledge
  isn't lost the next time someone touches this role.
