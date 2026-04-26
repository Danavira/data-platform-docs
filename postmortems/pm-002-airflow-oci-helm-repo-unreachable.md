# PM-002: Airflow OCI Helm repo unreachable

**Date:** 2026-04-12
**Severity:** Medium
**Status:** Resolved

## Problem Summary
The Airflow Helm chart was unreachable from within the cluster, preventing ArgoCD from syncing, even though the repository was accessible from my local machine.


## Problem
ArgoCD failed to install or update the Airflow Helm chart, consistently returning a `context deadline exceeded` timeout. The official Helm repository (`https://airflow.apache.org`) resolves to `archive.apache.org`, which proved to be unreachable from the cluster's network, causing all sync attempts to hang and eventually fail.


## Causes

### Egress Restrictions
A network egress issue specific to the cluster environment blocked traffic to `archive.apache.org`.

### Lack of OCI Support
Apache Airflow does not currently publish an OCI-compliant Helm chart, which prevented a direct switch to a different distribution protocol.


## How I diagnosed it
I started by verifying the ArgoCD sync status, which showed a timeout error. I then attempted to manually fetch the chart using `helm repo update` and `helm template` from within a pod in the cluster. Both commands failed with the same `context deadline exceeded` error, confirming the issue was network-related rather than specific to ArgoCD. I used `curl -v` to trace the connection to `archive.apache.org`, which confirmed that the connection was being blocked.


## Fixes

### Private Mirroring
Manually pulled the Airflow Helm chart to my local machine, pushed it to a personal GHCR repository (`ghcr.io/danavira/helm-charts`), and updated the ArgoCD Application source to point to this mirror.

### ArgoCD Compatibility Tuning
Updated `values.yaml` to include ArgoCD-specific configurations recommended by Airflow documentation. This involved disabling Helm hooks for user creation and database migrations, as these jobs conflict with ArgoCD's synchronization logic.


## Lessons / Ongoing Considerations

### Manual Update Lifecycle
Future Airflow upgrades will require a manual "pull-and-push" workflow to the GHCR mirror before the `targetRevision` can be bumped in the ArgoCD Application.

### Network Transparency
Local connectivity to a repository does not guarantee cluster connectivity; egress pathing should be verified early when `context deadline exceeded` errors occur during chart downloads.
