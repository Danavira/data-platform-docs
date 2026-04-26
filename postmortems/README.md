# Postmortems

Records of significant issues encountered while building or operating the platform — what broke, how it was diagnosed, and what was learned.

## Index

| # | Title | Date | Severity |
|---|---|---|---|
| [PM-001](pm-001-control-plane-node-rename.md) | Control plane node rename | 2026-04-12 | High |
| [PM-002](pm-002-airflow-oci-helm-repo-unreachable.md) | Airflow OCI Helm repo unreachable | 2026-04-12 | Medium |
| [PM-003](pm-003-clickhouse-helm-docker-hub-403.md) | ClickHouse Helm chart 403 from Docker Hub OCI registry | 2026-04-14 | Medium |
| [PM-004](pm-004-argocd-pod-exec-missing.md) | ArgoCD pod terminal exec missing from UI | — | Low |

## Template

```markdown
# PM-NNN: <short title>

**Date:** YYYY-MM-DD
**Severity:** Low | Medium | High
**Status:** Resolved | Ongoing

## Problem Summary

One paragraph. What broke and what the visible impact was (pipeline stalled,
deploy failed, cluster component unavailable, etc.).

## Problem

More detail on the context — what led up to it, what state things were in.

## Causes

The underlying reasons, not just the surface symptoms. Use subheadings if there
were multiple distinct causes.

## How I diagnosed it

The debugging steps in order — commands run, logs read, what ruled things out,
what led you to the root cause. This is the most valuable section to write in
detail.

## Fixes

What you changed to resolve it. Use subheadings to match the causes if applicable.
Include relevant commands, config snippets, or links to the commit/PR.

## Lessons / Ongoing Considerations

What this informed about the platform design, what you'd do differently, or any
follow-up work it created (link to an ADR if it drove an architectural decision).
```

## Severities

- **High** — data loss risk, cluster-wide outage, or pipeline completely stopped
- **Medium** — single component down, degraded performance, or blocked deployment
- **Low** — minor friction, slow rollout, config mistake with no data impact
