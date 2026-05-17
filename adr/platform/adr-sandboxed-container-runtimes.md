# ADR: Sandboxed container runtimes

**Date:** 2026-05-17
**Status:** Accepted

## Context

Sandboxed container runtimes (gVisor/`runsc`, Kata Containers) intercept syscalls between the container and the host kernel — either via a userspace kernel (gVisor) or a lightweight VM per pod (Kata). They protect against container escape exploits by reducing the host kernel's attack surface.

The question was whether to adopt one of these runtimes for the data platform workloads.

## Decision

Not adopted. The existing security posture is sufficient for this cluster's threat model, and the I/O overhead on stateful workloads is not justified.

## Alternatives considered

**gVisor (`runsc`)** — intercepts syscalls in userspace. Meaningful CPU overhead on syscall-heavy, I/O-bound workloads. Postgres, ClickHouse, and MinIO would all be hurt by this. Ruled out.

**Kata Containers** — runs each pod in a lightweight VM. Lower overhead than gVisor but still adds VM boot latency and memory per pod. Same workload concern applies. Ruled out.

**No change (current approach)** — accepted. The cluster is single-tenant; all images and workloads are controlled. Container escape risk is already mitigated by:
- Kyverno policies blocking privileged containers and privilege escalation
- `capabilities: drop: [ALL]` on every pod
- `runAsNonRoot` enforced cluster-wide
- No untrusted or user-submitted code running on the cluster

## Consequences

- Sandboxed runtimes remain worth revisiting if the cluster ever runs untrusted workloads (e.g. user-submitted DAGs, notebook execution, arbitrary job runners).
- No follow-up work required.
