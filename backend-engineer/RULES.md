# Backend engineer — rules

Long-form companion to [skills/backend-engineer/SKILL.md](../skills/backend-engineer/SKILL.md).

## Layering

```
route (HTTP)  →  service (business logic)  →  repository (DB)
```

- **Route** parses input, calls one service function, formats output. ~10 lines.
- **Service** is the unit of business logic. Pure-ish: takes typed inputs, returns typed outputs, may call repositories or other services. This is where tests live.
- **Repository** is the only place that knows about the ORM/SQL. Returns domain objects, not ORM rows.

The temptation to skip the repository is real and usually wrong. The day you swap Postgres for something else, or add a read replica, the lack of a repository will hurt.

## Validation

- Request bodies → schema parse (Pydantic / zod / etc). Reject 422 if invalid.
- Path/query params → same treatment.
- *Inside* the service, the types from the parser flow through. Don't re-validate.
- At the DB boundary, validate again only if the data crossed an untrusted edge (e.g. came back from a cache you don't fully control).

## Transactions

- Default: one transaction per request, opened by middleware, committed on 2xx, rolled back otherwise.
- If a step has to commit independently (e.g. write to outbox + publish), use the outbox pattern, not two transactions racing.
- Long-running work inside a transaction is a deadlock waiting to happen. Move it to a worker.

## Idempotency for write endpoints

Any non-GET that could be retried by the client should accept an `Idempotency-Key` header and store it. Card payments, account creation, anything that triggers a side effect outside your DB. The implementation is small; the bug class it prevents is large.

## Migrations

- Always forward. Rollback in production = a new migration that undoes the change.
- Schema changes are decoupled from code changes by two deploys:
  1. Add column (nullable) + deploy code that writes both old and new.
  2. Backfill.
  3. Deploy code that reads new only.
  4. Drop old column.
- Never run `ALTER TABLE` against a large table without checking lock behavior. Postgres `ADD COLUMN ... DEFAULT` is rewritten as of 11; older versions rewrite the table.

## Errors

- Two layers of errors: domain errors (typed, business-meaningful) and transport errors (HTTP status).
- Service returns `Result[T, DomainError]` (or raises a domain exception). Route translates to status code.
- Don't `try/except Exception: return 500`. Let unhandled errors surface to the framework's error handler so logs and traces capture the stack.

## Async vs sync

- Pick one per service and stick to it. Mixing sync DB drivers under an async framework is the slowest possible architecture.
- If you choose async (FastAPI, Fastify), every blocking call (DB, HTTP, file) must be the async variant or behind `run_in_executor`.

## Background work

- Cron-style: a scheduled task that picks up overdue work from the DB. Simple, debuggable.
- Queue-based: SQS / RabbitMQ / Redis Streams. Required once you need DLQs, fan-out, or strict ordering per key.
- Workers are deployed and observed *separately* from the API. They have their own SLOs.

## Caching

Three rules:

1. Cache at the highest layer that's still correct (CDN > app cache > DB query cache).
2. Every cache entry has a TTL. "Never expires" is a memory leak.
3. Invalidation is part of the write path, not a separate batch job, unless you've explicitly accepted staleness.

## Observability

- **Logs** are for debugging. JSON, with `trace_id`, `request_id`, `user_id` on every line. Never log secrets or PII.
- **Metrics** are for alerting. RED + USE. Per endpoint, per worker, per DB pool.
- **Traces** are for understanding cross-service latency. OpenTelemetry; one span per service hop, one per significant DB call.

## Security defaults

- Auth on every endpoint by default; opt out explicitly with a decorator/middleware for public ones.
- CSRF protection for cookie-auth web apps; not needed for bearer-token API clients.
- Rate limit by user *and* by IP. Different limits.
- SQL: parameters only. ORM-generated queries are fine; string-concatenated WHERE clauses are not.
- Dependency updates: weekly Dependabot/Renovate, security patches merged within 7 days.

## What this skill is *not*

- Frontend (separate skill).
- Pure data pipelines (use python-data-engineer).
- Infra/IaC (use terraform-devops).
