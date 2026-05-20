---
name: terraform-devops
description: Apply when writing or reviewing Terraform / OpenTofu code, cloud infrastructure modules, IAM policies, Kubernetes manifests (including Helm/Kustomize), or any infra-as-code change. Triggers on .tf/.tfvars files, on terraform/opentofu CLI usage, on modules/ and envs/ directories, and on terms like "state", "plan", "apply", "module", "provider", "IAM", "VPC".
---

# Terraform / DevOps

Apply these rules to any infrastructure-as-code change.

## Non-negotiables

- **Remote state, always.** S3 + DynamoDB lock, or Terraform Cloud, or equivalent. Never a local `terraform.tfstate` committed or shared by hand.
- **State is private.** State files contain secrets. Bucket is private, encrypted, versioned, access-logged.
- **Plan before apply, every time.** No `terraform apply -auto-approve` outside of CI that just ran a reviewed plan.
- **Modules are versioned, not floating.** Source has a `?ref=v1.2.3` tag or a registry version, never `main`.
- **One environment per workspace/directory.** No `if var.env == "prod"` branching scattered through resources.
- **No clickops to fix what code created.** If you fix something in the console, the next plan will undo it. Fix the code or import the change.

## Defaults

- **Language:** Terraform 1.7+ or OpenTofu 1.6+. Pin the version in `.terraform-version` (tfenv) or `required_version`.
- **Providers:** pin major+minor (`~> 5.40`). Lockfile (`.terraform.lock.hcl`) committed.
- **State backend:** S3 + DynamoDB on AWS; GCS on GCP; azurerm on Azure. Terraform Cloud / Spacelift for teams that want managed runs.
- **Layout:**
  ```
  envs/<env>/         # dev, staging, prod — one backend each
    main.tf
    variables.tf
    terraform.tfvars
  modules/<name>/     # reusable, versioned
  ```
- **Secrets:** never in `.tf` or `.tfvars`. Use `aws_secretsmanager_secret`, SSM Parameter Store, GCP Secret Manager, etc., and reference by ARN.
- **Linting:** `terraform fmt`, `tflint`, `tfsec` / `trivy config` in pre-commit and CI.
- **Docs:** `terraform-docs` to generate module READMEs.

## Anti-patterns — refuse without discussion

- `count = var.create ? 1 : 0` on a stateful resource (RDS, S3 with data, EBS). Switching it off destroys data.
- `lifecycle { ignore_changes = all }` to silence drift. Fix the drift or import the change.
- `null_resource` with `local-exec` running `aws` CLI commands. If Terraform doesn't have a resource for it, write a provider or use a real tool.
- IAM policy with `"Action": "*"` and `"Resource": "*"`. Even for "admin" roles, scope by service.
- Hardcoded account IDs, region strings, AMIs in resource blocks. Variables or data sources.
- A module that takes 40 inputs. Either it's doing too much or it's not a useful abstraction.
- `terraform destroy` against a shared environment without a written approval.

## Quality bar

- Every module has: `README.md` (generated), `versions.tf` with required providers, `variables.tf` with descriptions, `outputs.tf`.
- Every resource that holds data has `prevent_destroy = true` *and* a backup strategy in code (snapshot policy, replication, versioning).
- Every IAM role/policy has a comment explaining the principal and the use case.
- Plan output in PRs (Atlantis, `tf plan` posted by CI). No merge without a visible plan.
- A "blast radius" check before destructive changes: how many resources will be replaced/destroyed? Print it and require explicit ack.

See [terraform-devops/RULES.md](../../terraform-devops/RULES.md) for the long-form rationale.
