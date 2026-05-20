# Python data engineer — rules

The long-form companion to [skills/python-data-engineer/SKILL.md](../skills/python-data-engineer/SKILL.md). The SKILL file is the short list; this file explains *why*.

## Pipeline shape

A pipeline is three layers, separated:

1. **Extract** — read from source. Network/disk lives here, nowhere else.
2. **Transform** — pure functions. DataFrame in, DataFrame out. No I/O, no side effects.
3. **Load** — write to destination. Idempotent: same input → same output rows.

Keep them in separate functions (and usually separate files). When a test for the transform needs a mocked S3 client, you've already lost.

## Idempotency

Two properties matter:

- **Repeatable:** running the same job twice with the same logical inputs produces the same rows in the destination.
- **Resumable:** if the job dies halfway, the next run finishes the work without duplicates.

How to get there:

- Write to a partition (date, tenant, batch_id) and overwrite that partition, not the whole table.
- If you can't overwrite, use a merge/upsert keyed by a deterministic business key.
- Never use `INSERT` without a dedupe step downstream or a uniqueness constraint upstream.

## Schemas

At every boundary — file → memory, memory → warehouse, warehouse → API — assert the schema:

- pyarrow schemas for Parquet.
- Pydantic models for JSON and config.
- dbt `schema.yml` with `not_null`, `unique`, `accepted_values` tests for warehouse tables.

Inferred dtypes will betray you the day a column becomes nullable.

## Pandas vs Polars vs DuckDB

- **Polars** for new in-process transforms. Lazy frames let you fuse ops and stream larger-than-memory data.
- **DuckDB** when the transform is naturally SQL, or when you want to query Parquet directly without loading.
- **Pandas** when a library forces it (e.g. scikit-learn, statsmodels) or when the project already standardized on it. New Pandas code should still use `.copy()` aggressively and avoid chained assignment.

Don't mix three engines for the sake of it. Pick one per pipeline.

## Orchestration

The orchestrator's job is scheduling, retries, alerting, and lineage — not business logic. If your DAG file has more than 30 lines of Python that isn't `Operator(...)` calls, you're putting logic in the wrong place.

- Tasks are thin wrappers around library functions you can import and test independently.
- Use the orchestrator's templated dates (`{{ ds }}`, partition keys) — never `datetime.now()` inside a task.
- Backfill = re-running tasks for past intervals. If `datetime.now()` is in the code, backfill is broken.

## Secrets

- Read from the runtime's secret manager (AWS Secrets Manager, Vault, K8s Secret mounted as env).
- Never `os.environ.get("FOO", "default-with-real-value")`.
- Never commit `.env`. The `.gitignore` should block it; pre-commit hooks should re-check.

## Logging and observability

- One log line per stage with row count: `extracted={n} transformed={n} loaded={n}`.
- Structured logs (JSON) in production. Plain text fine in dev.
- Failures include the partition key in the message. "Job failed" is useless; "Job failed for ds=2026-05-19, tenant=acme" is actionable.

## Testing

- Unit tests on transforms with tiny fixture DataFrames (5–20 rows). Easy to read in a diff.
- Contract tests on sources: a daily Airflow task that asserts row count > X and freshness < Y. Failure pages someone.
- End-to-end tests run against a DuckDB or SQLite stand-in for the warehouse. Don't mock SQL — run real SQL against a real engine.

## Performance, when it matters

Reach for these in order:

1. Filter earlier (column pruning, partition pushdown).
2. Switch from Pandas to Polars/DuckDB.
3. Switch to streaming (Polars lazy, DuckDB streaming exports).
4. Move to the warehouse (push the transform into SQL).
5. Only then think about Spark / Ray / Dask.

Most "we need Spark" problems are actually "we're using Pandas and reading the whole table."

## Things this skill does *not* cover

- ML model training (that's a different skill).
- Streaming (Kafka/Kinesis/Pulsar) — separate concerns, separate skill if it grows.
- BI/dashboard layer.
