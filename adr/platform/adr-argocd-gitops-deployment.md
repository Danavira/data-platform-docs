# ADR: ArgoCD for GitOps deployment

**Date:** 2026-04-11
**Status:** Accepted

## Context

The platform runs on Kubernetes across multiple components (Postgres, ClickHouse, Airflow, Spark Operator, ESO). We need a way to manage deployments that is:

- Auditable (who deployed what, when)
- Reproducible (git is the source of truth)
- Operable without manual `kubectl apply` for each change

## Decision

Use ArgoCD to manage all Kubernetes deployments via GitOps. Manifests live in `data-platform-manifests`. ArgoCD watches this repo and reconciles cluster state to match.

Components deployed via Kustomize overlays (Postgres, ClickHouse, ESO) or Helm (Airflow) — ArgoCD supports both natively.

## Alternatives considered

**Flux** — comparable GitOps tool, also Kubernetes-native. ArgoCD chosen primarily for its UI, which makes it easier to diagnose sync issues without deep Kubernetes knowledge. Functionally equivalent for this use case.

**Plain CI/CD (GitHub Actions running kubectl apply)** — simpler initially but gives up continuous reconciliation (drift detection). Ruled out.

**Helm-only with CI** — doesn't handle non-Helm components (Postgres, ClickHouse use Kustomize). Ruled out.

## Consequences

- All infrastructure changes must go through `data-platform-manifests` — no ad-hoc `kubectl apply` in production.
- ArgoCD itself must be bootstrapped manually (or via a separate bootstrap script) — it can't deploy itself.
- Helm chart upgrades happen by bumping the chart version in `data-platform-manifests`; ArgoCD picks up the diff and syncs.
- ArgoCD adds an operator dependency. If ArgoCD is unhealthy, manual kubectl apply is the fallback.


## Note
Sync waves were implemented as a way to control the order of deployment of the applications. This is done by adding the `argocd.argoproj.io/sync-wave` annotation to the application manifests. The sync wave is a number that is used to determine the order of deployment of the applications. The applications are deployed in increasing order of their sync wave. For example, if an application has a sync wave of 0, it will be deployed before an application with a sync wave of 1.

In this repo, I gave `data-platform-configs-application.yaml` a sync wave of 0, and each `*-application.yaml` gets a sync wave of 1. This is because the `data-platform-configs-application.yaml` contains the secrets and configmaps for the other applications. If it were to be deployed after the other applications, the other applications would not be able to find their environment variables and would fail to start. 
