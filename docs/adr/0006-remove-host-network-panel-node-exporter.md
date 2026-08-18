# 6. Removed the host network panel from the node_exporter dashboard

## Status

Accepted

## Context

`node_exporter` runs as a container without `network_mode: host`. Its
network metrics therefore report the container's own network namespace —
its virtual interfaces on the Docker bridge — not the host's physical
interfaces. This was confirmed empirically by comparing the exporter's
reported figures against the host's own `/proc/net/dev`: they did not
match.

Mounting `/proc` into the container with `--path.procfs` does not fix
this, because `/proc/net` is namespace-scoped, not mount-scoped — bind
mounting the host's `/proc` still resolves `/proc/net` through the
container's own network namespace, not the host's.

The alternative fix, running `node_exporter` with `network_mode: host`,
was rejected: it would give the container full access to the host's
network namespace and all its listening sockets, which is a materially
larger exposure than what a metrics-only container needs, purely to make
one dashboard panel accurate.

## Decision

Remove the host network panel from the Grafana `node_exporter` dashboard
(`node-exporter.json.j2`) rather than keep it showing container-namespace
numbers mislabeled as host metrics, and rather than switching to
`network_mode: host` to make the panel accurate.

## Consequences

- The dashboard no longer claims to show host network throughput at all —
  a missing panel instead of a wrong one. Anyone who needs real host
  network figures has to get them another way (e.g. directly on the host).
- CPU, memory, filesystem, and load panels are unaffected and remain
  accurate — this is a narrow gap, not a hole in the monitoring stack.
- `node_exporter` keeps its current network isolation (no
  `network_mode: host`), consistent with the rest of the stack's
  least-privilege posture, at the cost of that one metric.
- If host-level network visibility is ever needed, it will require a
  deliberate follow-up decision (e.g. a different exporter, or accepting
  `network_mode: host` for that one container) — this ADR does not
  resolve that trade-off, only documents why it wasn't taken now.
