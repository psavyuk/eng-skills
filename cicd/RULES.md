# CI/CD — rules

Long-form companion to [skills/cicd/SKILL.md](../skills/cicd/SKILL.md).

## Pipeline shape

```
PR pipeline:        lint  →  unit tests  →  build  →  integration tests  →  artifact (kept)
main pipeline:      [same as PR, replayed]  →  push artifact  →  deploy dev  →  smoke  →  promote staging  →  smoke  →  promote prod
```

Two key properties:

- The **PR pipeline** proves the change *can* ship. Same jobs, same tools, same versions as main.
- The **main pipeline** deploys the artifact the PR pipeline just built. Don't rebuild — promote.

Build-once-promote-everywhere catches a whole class of "works in staging, broken in prod" bugs.

## Triggers

- `pull_request` for the PR pipeline. Filter `paths:` if the repo is a monorepo.
- `push` to `main` for the deploy pipeline.
- `workflow_dispatch` for manual reruns. Always available; gates rollback.
- Avoid `pull_request_target` unless you fully understand the privilege escalation risk (it runs against the target branch with secrets). If you must, never check out the PR head with `actions/checkout` and then run its scripts.

## Caching

Cache the slow, deterministic stuff:

- Package manager downloads (npm, pip, go mod, cargo).
- Compiled dependencies (Rust target/, Go build cache).
- Docker layers (`type=registry,ref=...` or buildx cache).

Don't cache the build output itself across runs — that's an artifact. Cache the inputs to make rebuilding fast.

Cache keys include lockfile hash + OS + arch. Stale caches that don't invalidate are worse than no cache.

## Secrets

- Stored in the platform's secret store (GitHub Encrypted Secrets, GitLab CI/CD Variables protected+masked, Vault, etc.).
- Injected as env vars to the steps that need them. Not workflow-wide.
- Never `echo $SECRET` for debugging. The platform masks known secrets in logs but won't catch transformed forms.
- Cloud auth via OIDC federation. GitHub Actions → AWS IAM Role via web identity, short-lived. No long-lived `AWS_ACCESS_KEY_ID` in secrets.
- Rotate any secret that ever appeared in plaintext, anywhere. Assume it's compromised.

## Container builds

Multi-stage, lean:

```Dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev=false
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
USER node
CMD ["node", "dist/server.js"]
```

- Pin base image by digest in prod (`node:20-alpine@sha256:...`).
- Non-root user. The Dockerfile chooses `USER` before `CMD`.
- Healthcheck only if it tests something meaningful — a `curl localhost:3000/healthz` for an HTTP server, not `true`.
- Build with `docker buildx build --provenance=true --sbom=true` for supply chain attestations.

## Artifacts and promotion

- Each successful build produces exactly one artifact (container image, tarball, wheel) tagged with the git SHA.
- Promotion is metadata: a deploy is "image X by digest Y goes to env Z." It's not a new build.
- Keep artifacts for at least 30 days for prod, 7 days for non-prod. Long enough to bisect a regression.

## Deploys

For Kubernetes: GitOps. CI writes a values bump to a manifests repo; the cluster reconciles via Flux or Argo. The CI runner doesn't have cluster credentials.

For everything else:

- Use the platform's native deploy primitive (CDK pipelines, Cloud Deploy, Cloud Run revision, Lambda alias, etc.).
- Blue/green or canary by default for prod. "Deploy and pray" is fine for dev, not for users.
- Automated rollback on health check failure within N minutes of deploy.

## Tests in CI

- **Lint + types** first — fastest, catches the most embarrassing failures.
- **Unit tests** next — fast, parallel, no network.
- **Integration tests** next — real DB (postgres service container is fine), real Redis, mocked external APIs.
- **E2E** last — slow, flaky-prone; run on PR but allow retry. Block merge only on the deterministic subset.

If a test is flaky three times in a month: quarantine it (move to a non-blocking job) and open a ticket to fix it. Don't normalize re-runs.

## Branch protection

For any repo with prod deploys:

- Require status checks (named, specific — not "all checks").
- Require branches up to date before merge.
- Require ≥1 approving review.
- Require signed commits or DCO.
- Restrict who can push to `main` to bots/admins. Humans go through PRs.
- No force-push to `main`.

## Monorepos

- Path-filter every workflow. Don't run the iOS pipeline because a TypeScript file changed.
- Cache per package, not globally. Globally-shared caches in a monorepo become bottlenecks.
- One artifact per deployable, even if they share code.

## What this skill is *not*

- Application code style (use the relevant per-language skill).
- Cloud infra creation (use terraform-devops).
- Internal developer-tooling repo design.
