# 8. Grafana dashboards provisioned as code with `allowUiUpdates: false`

## Status

Accepted

## Context

This project's core principle is that infrastructure state lives in this
Git repository and is reproducible from a clean checkout. Grafana
dashboards are an obvious place where that principle is easy to violate in
practice: it's fast to click a panel into shape in the UI, and just as easy
to forget it was never captured anywhere else, so a host rebuild silently
loses it.

## Decision

Provision Grafana's datasources and dashboards entirely as code: JSON
dashboard definitions and provisioning YAML are rendered from Jinja2
templates in `roles/monitoring/templates/grafana/`, and the dashboard
provider config sets `allowUiUpdates: false`
(`provisioning/dashboards/dashboard.yml.j2`).

## Consequences

- The JSON files in this repo are the single, unambiguous source of truth
  for every dashboard — a fresh deploy on a new host reproduces the exact
  same dashboards, no manual re-creation step.
- `allowUiUpdates: false` means any edit made in Grafana's UI is
  **discarded** on the next provisioning sync (Grafana restart or
  provisioning reload) — there is no "quickly tweak a panel and save" path
  left. Every change, including small ones, has to go back through editing
  the JSON template and redeploying.
- This trades UI convenience for reproducibility deliberately. Iterating on
  a dashboard is slower (edit JSON, redeploy, reload) than dragging panels
  in the browser, which is a real cost paid on every dashboard change, not
  just the initial setup.
- Anyone who forgets this rule and edits a panel in the Grafana UI expecting
  it to persist will lose that edit without warning at the next
  provisioning sync — a sharp edge inherent to this choice.
