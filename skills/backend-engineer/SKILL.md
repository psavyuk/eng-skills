---
name: backend-engineer
description: Apply when writing or reviewing backend services — REST/GraphQL APIs, async workers, database access, auth, observability. Triggers on FastAPI/Flask/Django/Express/Nest/Go-net-http code, on imports of sqlalchemy/asyncpg/prisma/typeorm, on files like routes/, handlers/, services/, and on terms like "endpoint", "migration", "worker", "queue".
---

# Backend engineer

Apply these rules to any service that handles HTTP/RPC requests, talks to a database, or runs async work.

## Non-negotiables

- **Validate at the edge, trust inside.** Parse and validate request payloads into typed objects (Pydantic, zod, struct + validator). Past that boundary, code can assume the data is valid.
- **One transaction per request, by default.** Open at the edge, commit on success, rollback on error. No nested ad-hoc transactions inside service functions.
- **Migrations are forward-only and reversible at the schema level.** Never edit a merged migration. Add a new one.
- **Background work goes to a queue, not a fire-and-forget task.** If the user doesn't need the result inline, it doesn't run in the request thread.
- **No N+1.** Eager-load or batch. If you write a `.filter().all()` inside a loop, you've already shipped a bug.

## Defaults

- **Python:** FastAPI + SQLAlchemy 2.x (async) + Alembic. Pydantic v2 for schemas.
- **Node:** Fastify or Nest + Prisma. zod for validation. Avoid Express for new services — too little structure.
- **Go:** standard `net/http` + `chi` for routing + `sqlc` for the DB layer. No ORM.
- **Auth:** OIDC/OAuth2 via a managed provider (Cognito, Auth0, Clerk). Don't roll your own session store.
- **DB:** Postgres unless there's a reason. Sqlite for local dev/tests is fine.
- **Cache/Queue:** Redis for cache + simple queues. SQS / RabbitMQ / Cloud Tasks once you need DLQs and fan-out.
- **Observability:** structured logs (JSON), OpenTelemetry traces, RED metrics (rate/errors/duration) on every endpoint.

## Anti-patterns — refuse without discussion

- Business logic inside a route handler. Route handlers parse input, call a service, format output. That's it.
- `SELECT *` in production code paths.
- Raw SQL string interpolation. Use parameters. Always.
- Returning DB models directly from an endpoint. Use a response schema; the DB shape is not the API shape.
- Storing passwords, API keys, or PII in logs.
- A migration that does `DROP COLUMN` in the same deploy as the code that stopped using it. Two deploys: stop using it, then drop it.
- `time.sleep` / `setTimeout` as a synchronization mechanism.

## Quality bar

- Every endpoint has at least one integration test that exercises auth + happy path + one error case.
- 4xx responses have a machine-readable error code, not a free-text English message.
- Health endpoint distinguishes liveness (`/healthz`) from readiness (`/readyz`). Readiness checks downstream deps; liveness does not.
- p95 latency budget per endpoint, documented in the route file or the OpenAPI spec.

See [backend-engineer/RULES.md](../../backend-engineer/RULES.md) for the long-form rationale.
