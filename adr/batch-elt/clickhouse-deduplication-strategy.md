# ADR: ClickHouse deduplication strategy

**Date:** 2026-04-27
**Status:** Accepted

## Context

The ETL pipeline reprocesses data in bounded time windows. A given row (identified by its primary key) may be written to ClickHouse multiple times — once per ETL run that touches its window. We need a deduplication strategy that is:

- Correct under re-runs and backfills
- Cheap to write to (high insert throughput)
- Queryable without full-table scanslets call

ClickHouse does not support `UPDATE` or `DELETE` in the traditional OLTP sense.

## Decision

Use `ReplacingMergeTree` with `update_timestamp` as the version column. Higher `update_timestamp` wins during background merges.

At query time, always use one of:
- `SELECT ... FROM db.table FINAL` — forces deduplication at read time, consistent but slower
- `SELECT argMax(col, update_timestamp)` grouped by primary key — more explicit, useful for aggregations

## Alternatives considered

**CollapsingMergeTree / VersionedCollapsingMergeTree** — requires tracking sign column in the application layer. More correct under concurrent writes but significantly more complex to implement and reason about. Ruled out given that our write pattern is append-only from a single Spark job.

**Plain MergeTree with manual dedup at query time** — no background dedup, query cost grows with reprocessing frequency. Ruled out.

**Periodic OPTIMIZE TABLE** — forces merges on a schedule to reduce `FINAL` overhead. Can be added later as an operational improvement if query latency becomes a problem.

## Consequences

- ETL jobs can be re-run safely; the latest write wins.
- All analytical queries must remember to use `FINAL` or `argMax`. Missing this is a silent correctness bug — queries will return duplicates.
- Background merge timing means `FINAL` can be slower than expected on large tables with recent inserts. Monitor merge lag in production.
- `OPTIMIZE TABLE` may be needed periodically for heavily reprocessed tables.
