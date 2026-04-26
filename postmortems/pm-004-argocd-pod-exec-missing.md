# PM-004: ArgoCD pod terminal exec missing from UI

**Date:** 2026-04-15
**Severity:** Low
**Status:** Resolved

## Problem Summary
The pod terminal execution feature was missing from the ArgoCD UI, preventing direct debugging from the web console.


## Problem
I was unable to exec into pods via the ArgoCD UI. The "Terminal" tab and execution options were completely missing from the interface, even though the feature was believed to be enabled.


## Causes

### Missing RBAC Permissions
While the execution feature was enabled at the server level, the ArgoCD RBAC policy did not explicitly grant the `exec` action to the user role. ArgoCD requires specific policy entries to allow the `create` action on `exec` sub-resources.


## How I diagnosed it



## Fixes

### Enable Server Execution
Ensured `server.exec.enabled` is set to `true` in `argocd-values.yaml`.

### Configure RBAC Policy
Updated `rbac.policy.csv` to explicitly allow the execution command. Added the following policy snippet to the ArgoCD configuration:

```
p, role:admin, exec, create, */*, allow
```

### Set Default Role
Confirmed `policy.default` is set to `role:admin` (or the appropriate role being used) to ensure the permissions are applied to the active session.


## Lessons / Ongoing Considerations

### Security vs. Convenience
Granting `exec` via RBAC should be handled with caution in multi-tenant environments. While `role:admin` is appropriate for a private lab or cluster admin, more granular policies should be used for restricted users.

### Feature Dependency
Remember that ArgoCD UI features often have a two-tier requirement: the feature flag in the server settings and the permission flag in the RBAC policy.
