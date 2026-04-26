# PM-001: Control plane node rename

**Date:** 2026-04-12    
**Severity:** High  
**Status:** Resolved

## Problem Summary
Renaming the control plane node caused a cascade of failures across multiple services (Vikunja, Postgres, Prometheus/Grafana, and `metrics-server`) due to orphaned PersistentVolume (PV) node affinities and controller conflicts.


## Problem
I had initially named my control plane node `danavira-vps-01` because I wanted to sequentially name them like:

- `danavira-vps-01`
- `danavira-vps-02`
- `danavira-vps-03`

However, I later realized that it would be better to name them based on their roles. So I decided to rename the control plane node to `danavira-control-plane`, and the worker nodes as `danavira-worker-x`.

After renaming the node from `danavira-vps-01` to `danavira-control-plane`, all stateful services failed to start. Pods remained in `Pending` or `CrashLoopBackOff` states because they were bound to PersistentVolumes that were still hard-coded to look for the old node name. Additionally, `metrics-server` failed due to a label selector mismatch between the built-in k3s addon and the ArgoCD Helm deployment.


## Causes

### PV Node Affinity Immutability
When a PV is created (especially using local storage or certain CSI drivers), Kubernetes automatically adds `nodeAffinity` to the PV spec. This affinity is immutable. Renaming the node made these PVs permanently unschedulable.

### Missing Taints/Tolerations
The Postgres deployment lacked the necessary `tolerations` to move to the worker node once the control plane became unavailable/changed.

### Service Selector Conflict
A conflict between k3s's native `metrics-server` and the ArgoCD-managed version resulted in a Service selector looking for a `k8s-app: metrics-server` label that did not exist on the Helm-managed pods.


## How I diagnosed it



## Fixes

### Stateful Services (Vikunja, Postgres, Prometheus/Grafana)
Scaled deployments to 0, deleted the orphaned PVCs/PVs, and scaled back up. This triggered the creation of new PVs with affinity for the updated node names. (Note: This resulted in acceptable data loss for this environment).

### Postgres Relocation
Added specific `nodeSelector` and `tolerations` to the Postgres manifest to ensure it targets `danavira-worker-1` and can bypass worker taints.

### Metrics-Server Patch
Manually patched the `metrics-server` service selector to remove the `k8s-app` requirement, allowing it to find the Helm-managed pods.


## Lessons / Ongoing Considerations

### Node Renaming Protocol
Treat a node rename as a "Delete and Recreate" operation. Before renaming, stateful data must be backed up or migrated, as PV `nodeAffinity` cannot be updated in place.

### Addon Management
Disable built-in k3s components (like `metrics-server`) via the `--disable` flag if they are going to be managed externally via ArgoCD/Helm to prevent selector merging conflicts.

### Data Persistence
For production environments where data loss is not acceptable, a manual migration of the underlying disk data to new PVs with the correct `nodeAffinity` would be required.
