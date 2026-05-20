---
name: cicd
description: Apply when writing or reviewing CI/CD pipelines, release automation, build scripts, container images, or deploy workflows. Triggers on .github/workflows, .gitlab-ci.yml, Jenkinsfile, Dockerfile, buildkite/, on terms like "pipeline", "deploy", "release", "artifact", "cache", and on changes to package scripts that affect build/test.
---

# CI/CD

Apply these rules to any pipeline, build, release, or deploy automation.

## Non-negotiables

- **A green pipeline means the change can ship.** If green doesn't mean "deployable," fix the pipeline. Skipped or flaky tests are not green.
- **Reproducible builds.** Same commit + same pipeline = identical artifact. Pin tool versions, lock dependencies, pin container base images by digest in production builds.
- **Secrets come from the CI secret store.** Never in a workflow file, never in a printed log, never in an env var that gets echoed by a debug step.
- **Tests run on PR; deploys run on merge.** The PR pipeline proves the change works; the main pipeline ships it. Same job code, different triggers.
- **One artifact, promoted across environments.** Build once, deploy the same artifact to dev → staging → prod. Don't rebuild per env.

## Defaults

- **Platform:** GitHub Actions for GitHub repos; GitLab CI for GitLab. Buildkite if scale or self-hosted runners become a real need.
- **Runners:** managed runners by default; self-hosted only when you need GPUs, specific networks, or volume that's cheaper that way.
- **Caching:** language-native cache (`actions/setup-node` with cache, `actions/setup-python` with `cache: 'pip'`). Docker layer cache via buildx / registry cache.
- **Container builds:** `docker buildx` with multi-stage Dockerfiles, BuildKit cache, multi-arch when needed.
- **Image registry:** GHCR / ECR / GAR — same cloud as the workload. Tags: `git-sha` (immutable), plus a moving `latest`/`main` tag for convenience. Promote by digest, not by tag.
- **Deploys:** GitOps (Flux/Argo) for Kubernetes — CI writes a manifest/values bump to a separate repo, the cluster reconciles. For serverless / VM targets, the CD tool of the platform (CDK pipelines, Cloud Deploy, Cloud Run revisions, etc.).
- **Release versioning:** semver tags, Conventional Commits, changesets / release-please for libraries. SHA-pinned for services.

## Anti-patterns — refuse without discussion

- `continue-on-error: true` on a step that's actually required. If it's optional, mark it; if it's required, fix the flake.
- `if: always()` on a deploy step. Deploys run only on success.
- Long-lived service account keys in CI. Use OIDC federation (GitHub → AWS/GCP/Azure) so CI gets a short-lived token.
- A pipeline that takes >15 minutes for the average PR. Split, parallelize, cache. Slow CI kills review velocity.
- Test step that runs all tests serially when 80% are independent.
- Bash-heredoc deploys (`ssh prod 'cat <<EOF | bash'`). Use a deploy tool.
- A workflow file with a `run:` block longer than ~30 lines. Move it to a script in `scripts/`, version it, test it locally.
- Pushing `:latest` to prod. Pin to digest.

## Quality bar

- p50 PR pipeline time: under 10 minutes. p95: under 20.
- Every workflow has a name, a clear trigger filter (`paths:` / `branches:`), and the right `permissions:` block (start from `read-all`, grant up).
- Branch protection requires: signed commits (or DCO), green required checks, ≥1 approval, up-to-date branch.
- Production deploys produce an audit trail: who, when, what commit, what artifact digest, link to the pipeline run.
- Rollback is a documented procedure that's been rehearsed in the last quarter — not a Slack thread of "what did we do last time?"

See [cicd/RULES.md](../../cicd/RULES.md) for the long-form rationale.
