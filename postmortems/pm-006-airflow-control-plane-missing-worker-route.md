# PM-006: Airflow migration jobs hanging due to missing control-plane → worker-1 flannel route

**Date:** 2026-05-16
**Severity:** High
**Status:** Resolved

## Problem Summary

Airflow had been completely down for 18 days — scheduler, webserver, and triggerer all stuck in `Init:CrashLoopBackOff`. The root cause was that the control-plane node was missing a flannel route and ARP entry for `danavira-worker-1`, where Postgres lives. ArgoCD schedules migration jobs on the control-plane, so `airflow db migrate` could never reach the database. Every sync attempt silently hung at the DB connection step, leaving no useful error in the pod logs.

## Problem

After a previous round of fixes (PM-005), the worker-to-worker flannel ARP entries had been made permanent and worker connectivity was restored. However Airflow remained broken. The ArgoCD hook jobs (`airflow-run-airflow-migrations`, `airflow-create-user`) had been in a `Failed` state since the original deployment 18 days prior, held in place by the `argocd.argoproj.io/hook-finalizer` which prevented them from being deleted and re-run.

When the jobs were finally unblocked and re-run, they scheduled on the control-plane (`10.42.1.x`) rather than the worker nodes. Postgres runs on `danavira-worker-1` at `10.42.3.19`. The control-plane had no route to `10.42.3.0/24` and no ARP entry for worker-1's flannel VTEP — so the DB connection hung silently with no error.

## Causes

### Missing flannel route and ARP entry on the control-plane for worker-1

The control-plane's flannel routing table only contained entries for worker-2's pod CIDR (`10.42.0.0/24`) and its own (`10.42.1.0/24`). Worker-1's pod CIDR (`10.42.3.0/24`) was absent entirely:

```
10.42.0.0/24 via 10.42.0.0 dev flannel.1 onlink   # worker-2 ✓
10.42.1.0/24 dev cni0 proto kernel                 # local ✓
10.42.2.0/24 via 10.42.2.0 dev flannel.1 onlink   # (extra CIDR) ✓
# 10.42.3.0/24 — MISSING ✗
```

Similarly, `ip neigh show dev flannel.1` on the control-plane had no entry for `10.42.3.0` (worker-1's VTEP IP). Without the ARP entry the kernel cannot resolve the next-hop MAC, so VXLAN encapsulation never happens and packets are silently dropped.

PM-005 only fixed the worker-to-worker paths — the control-plane → worker-1 path was left broken, likely because PM-005 was diagnosed entirely from the worker nodes and the control-plane path had appeared to work at the time.

## How I diagnosed it

1. ArgoCD migration jobs were in `Failed` state with age 18 days — they had never re-run since initial deployment
2. Attempted `kubectl delete job` — jobs were stuck due to `argocd.argoproj.io/hook-finalizer`; patched finalizers off both jobs
3. ArgoCD sync then got stuck on `waiting for deletion of hook /Secret/airflow-fernet-key` — patched that finalizer too, restarted the ArgoCD application controller, force-cleared the operation via JSON patch
4. Deleted and force-recreated the `airflow` namespace to get a clean state
5. After resync, migration jobs appeared as `1/1 Running` for over 12 minutes — migrations don't take that long, confirmed they were hung
6. `k logs` on the migration job pod returned empty — connection was hanging before any output
7. Ran a test pod in the `airflow` namespace to connect to `postgres.postgres.svc.cluster.local:5432` — hung with no response
8. `k exec -n postgres postgres-0 -- psql -U postgres` worked — ruled out Postgres itself
9. Suspected CNI issue based on PM-005; checked worker ARP tables — all entries `PERMANENT`, workers fine
10. Checked `k get pods -o wide` — migration jobs were on `danavira-control-plane`, Postgres on `danavira-worker-1`
11. `ip route show | grep 10.42` on the control-plane — `10.42.3.0/24` completely absent
12. `ip neigh show dev flannel.1` on the control-plane — no entry for `10.42.3.0`

## Fixes

Added the missing route and permanent ARP entry on the control-plane:

```bash
# On danavira-control-plane
sudo ip neigh replace 10.42.3.0 lladdr a6:7a:a9:1f:05:8d dev flannel.1 nud permanent
sudo ip route add 10.42.3.0/24 via 10.42.3.0 dev flannel.1 onlink
```

Migration jobs completed immediately after. Airflow scheduler, webserver, and triggerer came up once the jobs finished.

## Lessons / Ongoing Considerations

### Always check the control-plane routes, not just the workers
PM-005 established a debugging checklist for flannel issues focused on worker nodes. The control-plane is also a flannel participant and its routing table can be incomplete independently of the workers. The checklist should be run on all nodes, not just the ones where the affected pods are scheduled.

### ArgoCD hook jobs scheduling is opaque
The migration jobs are ArgoCD sync hooks with no `nodeSelector`, so Kubernetes scheduled them on the control-plane by default. There is no indication in the ArgoCD UI or pod events that the job ended up on a node that cannot reach the database. A hanging connection produces no log output and no error — it just looks like the job is running slowly.

Consider adding a `nodeSelector` or `affinity` to the migration job in `values.yaml` to pin it to a node closer to Postgres, or at minimum ensure the control-plane always has complete flannel routes to all worker pod CIDRs.

### Flannel route persistence remains unresolved
As noted in PM-005, the manually added flannel routes and ARP entries are in-memory only and will be lost on node reboot or k3s-agent restart. Switching to `host-gw` flannel backend (via `--flannel-backend=host-gw`) would eliminate VXLAN entirely and make routing stable. This is still outstanding.

### VTEP reference (updated)
| Node | flannel.1 IP | VTEP MAC |
|---|---|---|
| danavira-control-plane | 10.42.1.0 | b2:b9:bc:ce:b2:d4 |
| danavira-worker-1 | 10.42.3.0 | a6:7a:a9:1f:05:8d |
| danavira-worker-2 | 10.42.0.0 | f6:62:f4:8d:b6:33 |
