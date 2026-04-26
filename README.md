# data-platform-docs

Documentation for my self-hosted Kubernetes data platform. The code is split across three repos; this is where I keep the docs that explain the thinking behind it. The documentations here are meant to show my thought process, the decisions I made, the tradeoffs I considered, and the fixes I implemented for the issues I encountered. May it be a reflection of my knowledge and experience thus far.

## Repos

| Repo | Purpose |
|---|---|
| [data-platform-manifests](https://github.com/yasadanavira/data-platform-manifests) | Kubernetes manifests (Kustomize), Helm values, ArgoCD Applications |
| [data-platform-configs](https://github.com/yasadanavira/data-platform-configs) | Environment-specific configuration, Infisical secret references |
| [batch-etl](https://github.com/yasadanavira/batch-etl) | ETL pipeline code: Spark jobs, Airflow DAGs, data generator, schemas |

## What's here

**[architecture/](architecture/)** — How the platform is put together and why I designed it like so.

**[adr/](adr/)** — The significant decisions I made, what alternatives I considered, and why I went the way I did. See the [ADR index](adr/README.md).

**[how-to/](how-to/)** — Step-by-step guides for common tasks.

**[postmortems/](postmortems/)** — Things that broke, how I figured out what went wrong, and what I did to fix it. See the [postmortem index](postmortems/README.md).
