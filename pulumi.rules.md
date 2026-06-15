# Pulumi Go Rules (2025-2026)

Last verified: 2026-02-19.
Scope: practical rules for organizing and operating Pulumi Go projects in this repository.

## 1. Project and stack organization

- Treat each Pulumi project as a deployment boundary and each stack as an environment boundary.
- Use stack-per-environment naming (`dev`, `staging`, `prod`, feature stacks).
- Keep Pulumi project structure aligned with Git repository boundaries when possible.
- If a project grows into many layer files or mixed ownership, split into multiple Pulumi projects and connect with stack references.
- Add stack tags for grouping and operations (`environment`, `service`, `owner`).

## 2. Repo layout conventions (Go)

- Keep `Pulumi.yaml`, `go.mod`, and `go.sum` at the Pulumi program root.
- Keep `main.go` as the composition entrypoint; move reusable infra logic into internal packages/components.
- Keep generated local SDKs (for parameterized providers) versioned in-repo when used by this project.
- Keep Pulumi stack settings files (`Pulumi.<stack>.yaml`) versioned for shared environments.

## 3. Versioning and setup hygiene

- Pin the Pulumi CLI version range using `requiredPulumiVersion` in `Pulumi.yaml`.
- Pin provider/package versions explicitly in `Pulumi.yaml` (`packages` map).
- Use `pulumi install` after clone and whenever `Pulumi.yaml` package declarations change.
- Prefer `pulumi/actions` over deprecated `pulumi/setup-pulumi` in GitHub Actions.

## 4. Configuration and secrets

- Set config with CLI commands (`pulumi config set`, `pulumi config get`) instead of hand-editing YAML.
- Store sensitive config with `pulumi config set --secret`.
- In Go code, read secrets with `RequireSecret`/secret getters, not plain getters.
- Prefer ESC environments for shared org config/secrets and import via stack `environment`.
- Use cloud KMS/Key Vault/HCP Vault style secrets providers for team/production stacks.

## 5. Naming and refactors

- Keep default auto-naming unless explicit names are required by platform constraints.
- If using custom auto-naming patterns, include randomness to reduce collision and replacement downtime risk.
- Never rename resources blindly: changing logical names can force delete+create.
- For renames/moves/reparenting, use resource `aliases` until all stacks have migrated.

## 6. Providers and multi-env patterns

- Use default providers for single-region/single-account deployments.
- Use explicit providers when deploying multiple regions/accounts/tenants in one stack.
- When using explicit providers, pass `provider` in resource options consistently.

## 7. CI/CD and deployment workflows

- Standard workflow: PR runs `pulumi preview`, protected branch runs `pulumi up`.
- For stricter change control, use update plans:
  - Generate: `pulumi preview --save-plan=plan.json`
  - Apply: `pulumi up --plan=plan.json`
- If using Pulumi Deployments in monorepos, enable path filtering.
- Consider `Skip intermediate deployments` for high-commit branches.
- Prefer OIDC for cloud auth in deployments; avoid long-lived static cloud credentials.
- Configure deployment role assignment when stacks need stack references, ESC environments, or org resources.

## 8. Drift detection and remediation

- Run regular drift checks with `pulumi refresh --preview-only`.
- For Pulumi Deployments, configure drift schedules/remediation in deployment settings.
- Treat drift findings as production incidents until triaged.

## 9. Testing strategy (Go)

- Use `go test` for unit tests with Pulumi mocks.
- Keep unit tests focused on transformation/business logic; mocks do not implement the full Pulumi engine.
- Use integration tests for lifecycle behavior and real provider interactions.
- Use Pulumi integration testing (`integration.ProgramTest`) and `ExtraRuntimeValidation` for post-deploy assertions.

## 10. Governance and policy

- Run policy checks in preview paths (local and CI) before merge.
- Start with pre-built Pulumi policy packs for baseline security/compliance.
- Add custom policy packs for org-specific controls and set explicit enforcement levels (`advisory`/`mandatory`).

## 11. This repository: required defaults

- Keep `Pulumi.yaml` package declarations version-pinned.
- Keep local provider SDK artifacts under `sdks/` tracked in Git when used by the program.
- Keep stack config and secrets metadata files tracked for shared stacks.
- Do not merge infra PRs without a fresh preview against the target stack.

## Sources

- https://www.pulumi.com/docs/iac/guides/basics/organizing-projects-stacks/
- https://www.pulumi.com/docs/iac/concepts/stacks/
- https://www.pulumi.com/docs/iac/concepts/projects/project-file/
- https://www.pulumi.com/docs/iac/concepts/projects/stack-settings-file/
- https://www.pulumi.com/docs/iac/get-started/terraform/terraform-providers/
- https://www.pulumi.com/docs/iac/concepts/resources/names/
- https://www.pulumi.com/docs/iac/concepts/resources/options/aliases/
- https://www.pulumi.com/docs/iac/concepts/providers/
- https://www.pulumi.com/docs/iac/concepts/state-and-backends/
- https://www.pulumi.com/docs/iac/concepts/secrets/
- https://www.pulumi.com/docs/esc/environments/working-with-environments/
- https://www.pulumi.com/docs/iac/guides/testing/unit/
- https://www.pulumi.com/docs/iac/guides/testing/integration/framework/
- https://www.pulumi.com/docs/iac/guides/continuous-delivery/github-actions/
- https://github.com/pulumi/actions
- https://github.com/pulumi/setup-pulumi
- https://www.pulumi.com/docs/iac/guides/basics/update-plans/
- https://www.pulumi.com/docs/deployments/deployments/drift/
- https://www.pulumi.com/docs/deployments/deployments/using/settings/
- https://www.pulumi.com/docs/insights/policy/policy-packs/pre-built-packs/
- https://www.pulumi.com/docs/insights/policy/policy-packs/authoring/
