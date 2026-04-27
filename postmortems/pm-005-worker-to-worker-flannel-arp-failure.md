# PM-005: Worker-to-worker pod networking failure (flannel ARP incomplete)

**Date:** 2026-04-27    
**Severity:** Very High     
**Status:** Resolved    

## Problem Summary
Cross-node pod communication between `danavira-worker-1` and `danavira-worker-2` was broken, causing Airflow (on worker-2) to be unable to connect to Postgres (on worker-1). Airflow scheduler, webserver, and triggerer pods were stuck in `Init:0/2` indefinitely. Root cause was an `INCOMPLETE` ARP/neighbor entry for worker-1's flannel VTEP on worker-2, preventing VXLAN packet encapsulation.

## Problem
After re-provisioning both worker nodes, Airflow pods were stuck in `Init:0/2`. The `wait-for-airflow-migrations` init container runs `airflow db check-migrations`, which connects to Postgres. The connection was hanging — not refusing, not timing out, just hanging indefinitely.

All other Airflow components (statsd, triggerer) were either running or also blocked in init. Postgres itself was healthy and listening on `0.0.0.0:5432`.

## Causes

### Missing flannel ARP neighbor entries
Flannel VXLAN requires four components to all be correct for cross-node pod traffic to flow:

1. **Route** — `10.42.x.0/24 via 10.42.x.0 dev flannel.1 onlink` (tells the kernel to send traffic for remote pods via flannel.1)
2. **ARP/neighbor entry** — `10.42.x.0 lladdr <vtep-mac> dev flannel.1 permanent` (resolves the VTEP IP to a MAC address)
3. **FDB entry** — `<vtep-mac> dst <node-ip>` (tells VXLAN which physical IP to encapsulate to)
4. **Host-level UDP 8472** — reachable between nodes

The FDB entries were correct on both workers. The routes were missing initially but added after k3s-agent restarts. The ARP entries for the worker-to-worker VTEP IPs were `INCOMPLETE` — the kernel could not resolve the MAC for the next-hop VTEP, so packets were silently dropped before even leaving `flannel.1`.

The control-plane ↔ worker paths worked correctly throughout, suggesting the issue was specific to the timing or order of worker registration when the nodes were re-provisioned.

## How I diagnosed it

1. `nc -zv postgres.postgres.svc.cluster.local 5432` from an Airflow pod — hung
2. `nslookup` worked, ruling out DNS
3. `/proc/net/tcp` on Postgres pod confirmed it was listening on `0.0.0.0:5432`
4. `kubectl get networkpolicy` — none in either namespace
5. Test pod on worker-2 trying to reach Postgres pod IP — also hung, ruling out Airflow-specific issue
6. `ip route show | grep 10.42` on both workers — missing each other's pod CIDRs
7. Restarted k3s-agent on both workers — routes for each other still missing
8. UFW `default deny routed` fixed and k3s-agent restarted — still no routes
9. Checked flannel FDB: `bridge fdb show dev flannel.1` — both workers had correct MAC-to-IP entries for each other
10. Manually added routes: `ip route add 10.42.x.0/24 via 10.42.x.0 dev flannel.1 onlink` — changed error from hanging to `No route to host` (progress: VXLAN encapsulation was now attempted)
11. `tcpdump -i flannel.1` — no packets leaving worker-2 despite route existing
12. `ip neigh show dev flannel.1` — revealed `10.42.3.0 INCOMPLETE` on worker-2, which was the root cause

## Fixes

Added permanent ARP neighbor entries for the cross-worker VTEP IPs on both nodes:

```bash
# On worker-2 — to reach worker-1's pods
sudo ip neigh replace 10.42.3.0 lladdr a6:7a:a9:1f:05:8d dev flannel.1 nud permanent

# On worker-1 — to reach worker-2's pods
sudo ip neigh replace 10.42.4.0 lladdr f6:62:f4:8d:b6:33 dev flannel.1 nud permanent
```

Cross-node connectivity was immediately restored.

## Lessons / Ongoing Considerations

### Fix is not persistent
These ARP entries live in memory and will be lost on k3s-agent restart or node reboot. They need to be made persistent. Options:

1. **Startup script** — add a systemd drop-in or `/etc/rc.local` on each worker that re-adds the entries after k3s-agent starts
2. **Switch flannel backend to `host-gw`** — eliminates VXLAN entirely, uses direct L3 routing. Cleaner for this setup since all nodes are on the same `10.3.0.0/22` subnet. Configured via k3s `--flannel-backend=host-gw` flag
3. **Investigate flannel ARP population bug** — may be related to the minor k3s version mismatch between control-plane (`v1.34.5+k3s1`) and workers (`v1.34.6+k3s1`)

### VTEP MAC addresses (for reference)
| Node | flannel.1 IP | VTEP MAC |
|---|---|---|
| danavira-control-plane | 10.42.1.0 | b2:b9:bc:ce:b2:d4 |
| danavira-worker-1 | 10.42.3.0 | a6:7a:a9:1f:05:8d |
| danavira-worker-2 | 10.42.4.0 | f6:62:f4:8d:b6:33 |

### Debugging cross-node pod networking
When a pod-to-pod connection hangs (not refuses), always check in this order:
1. Is the target pod actually listening? (`/proc/net/tcp`)
2. Are there NetworkPolicies? (`kubectl get networkpolicy`)
3. Are the pods on different nodes? (`kubectl get pods -o wide`)
4. Are flannel routes present? (`ip route show | grep 10.42`)
5. Are FDB entries present? (`bridge fdb show dev flannel.1`)
6. Are ARP entries complete? (`ip neigh show dev flannel.1`)
