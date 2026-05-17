# Architecture Decision Records

ADRs record significant decisions made about the platform — what was decided, why, and what was ruled out.

## Index

### Platform

| Title | Status |
|---|---|
| [ArgoCD for GitOps deployment](platform/adr-argocd-gitops-deployment.md) | Accepted |
| [Infisical for secrets management](platform/adr-infisical-secrets-management.md) | Accepted |
| [Sandboxed container runtimes](platform/adr-sandboxed-container-runtimes.md) | Accepted |

### Batch ELT

| Title | Status |
|---|---|
| [ClickHouse deduplication strategy](batch-elt/adr-clickhouse-deduplication-strategy.md) | Accepted |

## Template

```markdown
# ADR-NNN: Title

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-NNN

## Context

What is the problem or situation that requires a decision? What constraints exist?

## Decision

What was decided?

## Alternatives considered

What else was evaluated, and why was it ruled out?

## Consequences

What becomes easier or harder as a result of this decision?
What follow-up work does this create?
```

## Statuses

- **Proposed** — under discussion, not yet in production
- **Accepted** — in effect
- **Deprecated** — no longer applies but kept for history
- **Superseded** — replaced by a later ADR (link to it)
