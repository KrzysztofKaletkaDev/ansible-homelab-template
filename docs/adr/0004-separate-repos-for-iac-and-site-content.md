# 4. Separate repositories for infrastructure and site content

## Status

Accepted

## Context

The portfolio site's source (HTML/CSS/JS/build tooling) and this
infrastructure automation (Ansible roles, playbooks, Vault-encrypted
secrets) change for entirely different reasons, on different cadences, and
are reviewed by different concerns — one is content/design work, the other
is infrastructure change with security implications (firewalld rules,
secret handling, container versions). Mixing them in one repository would
force every content tweak through the same review lens as an
infrastructure change, and vice versa.

## Decision

Keep the site's content in its own repository
(`kaletkadev-portfolio-site`), cloned onto the host at deploy time by the
`portfolio` role (`ansible.builtin.git`, pinned to the `main` branch). This
repository (`ansible-homelab-template`) contains only infrastructure code
and references the content repo by URL
(`portfolio_repo_url` in `group_vars/all/vars.yml`).

## Consequences

- Content changes (a new blog post, a design tweak) ship independently of
  infrastructure changes, with their own commit history and review — no
  need to touch Ansible to update the site.
- The infrastructure repo has no visibility into the content repo's build
  process; if the content repo's `main` branch is broken or unbuilt, the
  `portfolio` role will happily clone it as-is (`force: true` on every
  run) and Caddy will serve whatever is there, with no CI gate in this
  repo checking that the content repo's output is valid HTML.
- Pinning to `main` (rather than a tag or commit SHA) means a deploy is not
  perfectly reproducible — re-running `site.yml` days apart can serve
  different content even with an identical Ansible checkout, unlike every
  other role in this repo where versions are explicitly pinned
  (see [ADR-0005](0005-cadvisor-version-pin.md)).
- Two repositories to keep track of instead of one — a minor coordination
  cost when the same person maintains both, but a real one nonetheless.
