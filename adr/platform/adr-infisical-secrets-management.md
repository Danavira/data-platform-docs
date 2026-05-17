# ADR: Infisical for secrets management

**Date:** 2026-04-27    
**Status:** Accepted

## Context

The platform runs on Kubernetes and needs secrets for: Postgres credentials, ClickHouse credentials, Airflow Fernet key, and any external service tokens. Secrets must be:

- Not committed to Git
- Available to pods at runtime
- Manageable without direct `kubectl` access for every rotation

## Decision

Use Infisical as the secrets backend, with External Secrets Operator (ESO) to sync secrets into Kubernetes `Secret` objects. Application pods consume native Kubernetes secrets — they have no direct Infisical dependency.

Secret references (not values) live in `data-platform-configs`.

## Alternatives considered

**HashiCorp Vault** — more mature and feature-rich (dynamic secrets, PKI, audit log). Operationally heavier to run self-hosted. Ruled out for now given platform scale; revisit if compliance requirements grow.

**Sealed Secrets** — encrypts secrets for Git storage. Simpler than ESO+Infisical but rotation requires re-committing encrypted blobs. Ruled out because it doesn't give a secrets UI or audit trail.

**Raw Kubernetes Secrets in a private repo** — simplest possible approach. Ruled out because secrets in Git (even private) are a bad practice.

## Consequences

- Secret rotation happens in Infisical; ESO syncs to the cluster on a configurable interval without redeployment.
- ESO adds an operator dependency — if ESO is unhealthy, secret sync stops.
- Infisical availability becomes a dependency for cluster bootstrapping (pods won't start if their secrets haven't synced yet).
- All secret references should be documented in `data-platform-configs/docs/`.
