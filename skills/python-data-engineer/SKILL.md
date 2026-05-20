---
name: python-data-engineer
description: Apply when writing or reviewing Python data pipelines, ETL/ELT jobs, SQL transformations, dbt models, Airflow/Dagster DAGs, or warehouse-facing code. Triggers on imports of pandas/polars/pyarrow/sqlalchemy/duckdb/snowflake/bigquery, on files in airflow/dags or dbt/models, and on terms like "pipeline", "ingest", "backfill", "warehouse".
---

# Python data engineer

Apply these rules to any data-pipeline or warehouse-facing Python code.

## Non-negotiables

- **Idempotent jobs.** Re-running a task with the same inputs must produce the same outputs. No append-without-dedupe, no "skip if exists" without a deterministic key.
- **Pure transforms, separate I/O.** Read → transform → write. The transform function takes a DataFrame (or iterator) and returns a DataFrame. It does not open files or hit the network.
- **Explicit schemas at boundaries.** When you ingest from an external source or write to one, declare the schema (pyarrow / pydantic / dbt yaml). Never trust inferred types across a stage boundary.
- **Timestamps are UTC, tz-aware.** Naive datetimes are a bug. Convert on ingest, store UTC, format on display.
- **Backfill before you ship.** If the pipeline runs on a schedule, you must be able to backfill an arbitrary date range with one command. If you can't, the design is wrong.

## Defaults

- **Engine:** Polars > Pandas for new code. Use Pandas only when an existing pipeline forces it or a library returns a Pandas DataFrame you can't avoid.
- **Storage:** Parquet for interchange and intermediate stages. CSV only at human-facing edges.
- **SQL transforms:** dbt if the project already uses it; otherwise plain SQL files run via SQLAlchemy / DuckDB. Avoid ORM models for analytics work.
- **Orchestration:** Airflow if the team has it, Dagster if greenfield. Cron + a Python script is fine for one-off jobs — don't over-orchestrate.
- **Testing:** pytest with small fixture datasets stored in `tests/fixtures/`. Property tests (hypothesis) for parsers/validators.
- **Packaging:** `uv` or `poetry`; pin everything; commit the lockfile.

## Anti-patterns — refuse without discussion

- `pd.read_sql("SELECT * FROM big_table")` with no filter, no chunking.
- Mutating a DataFrame argument in-place inside a function.
- Catching `Exception` in a pipeline step and continuing — fail loud.
- `time.sleep` to "wait for the warehouse" — use the warehouse's wait/poll API.
- Storing credentials in code, in DAG params, or in dbt `profiles.yml` committed to the repo.
- One giant 500-line SQL string. Break it into CTEs or models with names.

## Quality bar

- Every pipeline has a smoke test that runs on a 100-row fixture in CI.
- Every external data source has a contract test (row count + key columns non-null + at least one freshness check).
- Log row counts at each stage. Counts are the cheapest debugging tool you have.

See [python-data-engineer/RULES.md](../../python-data-engineer/RULES.md) for the long-form rationale.
