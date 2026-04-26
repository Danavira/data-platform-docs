# PM-003: ClickHouse Helm chart 403 from Docker Hub OCI registry

**Date:** 2026-04-14
**Severity:** Medium
**Status:** Resolved

## Problem Summary
Attempts to fetch the ClickHouse Helm chart via Docker Hub's OCI registry resulted in `403 Forbidden` errors during ArgoCD synchronization.


## Problem
My ArgoCD instance failed to sync the ClickHouse application. When attempting to resolve Helm chart version `9.4.4` from `registry-1.docker.io/bitnamicharts/clickhouse`, the process was terminated by a `403 Forbidden` response from the registry.


## Causes

### Docker Hub Auth Restrictions
Docker Hub has increased restrictions on anonymous pull requests. ArgoCD's unauthenticated attempt to resolve the chart digest was rejected during the authentication token exchange phase.

### Registry Dependency
The application was reliant on `registry-1.docker.io`, which is subject to more aggressive rate-limiting and authentication requirements than other public mirrors.


## How I diagnosed it



## Fixes

### Registry Migration
Changed the `repoURL` in `clickhouse-application.yaml` from `oci://registry-1.docker.io/bitnamicharts/clickhouse` to `oci://ghcr.io/bitnamicharts/clickhouse`. Bitnami officially mirrors their charts to GitHub Container Registry (GHCR), which allows for public, unauthenticated access.

### Compatibility Hardening
Included ArgoCD compatibility values to disable Helm hooks for user creation and database migrations, ensuring the lifecycle is managed correctly by ArgoCD's sync engine.


## Lessons / Ongoing Considerations

### Bitnami Audit
I need to audit the cluster for other Bitnami-sourced charts. Any application still pointing to `registry-1.docker.io` is a candidate for a similar `403` failure and should be migrated to `ghcr.io`.

### Authentication vs. Mirroring
While I could configure Docker Hub credentials as a repository secret in ArgoCD to bypass these limits, GHCR is currently the simpler path as it avoids rate-limit overhead entirely.

### Mirror Reliability
While GHCR is stable, I should monitor Bitnami's official documentation to ensure GHCR remains a primary, first-class distribution point for their charts long-term.
