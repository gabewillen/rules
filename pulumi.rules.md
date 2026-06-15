# Pulumi Go Rules

## 1. Project & Stack Architecture
- **Boundaries**: 1 Project = Deployment boundary. 1 Stack = Environment boundary (`dev`, `staging`, `prod`).
- **Structure**: Keep `Pulumi.yaml`, `go.mod`, and `main.go` at the project root.
- **Composition**: Use `main.go` strictly for composition. Move reusable infrastructure logic into internal Go packages.
- **Scaling**: Split monolithic projects by layer or owner; connect them using stack references.
- **Metadata**: Add stack tags for grouping (`environment`, `service`, `owner`).

## 2. Resource Management
- **Naming**: Rely on default auto-naming. If custom names are required, include randomness to prevent collisions during replacements.
- **Refactoring**: **Never rename logical resources blindly**, as it forces a delete-and-replace. Use resource `aliases` to migrate state during renames, moves, or reparenting.
- **Providers**: Use default providers for single-account/region setups. Pass explicit `provider` options for multi-account/region deployments within a single stack.

## 3. Configuration & Secrets
- **CLI Only**: Use `pulumi config set` (with `--secret` for sensitive data). Never hand-edit `Pulumi.<stack>.yaml`.
- **Code Integration**: Always read sensitive values using `RequireSecret` in Go.
- **State Files**: Track shared environment configuration (`Pulumi.<stack>.yaml`) in version control.
- **External Stores**: Use Pulumi ESC, KMS, or Vault for managing team/production secrets.

## 4. Dependencies & Versioning
- **Pinning**: Explicitly pin the Pulumi CLI (`requiredPulumiVersion`) and provider versions (`packages` map) in `Pulumi.yaml`.
- **Local SDKs**: Version-control locally generated SDK artifacts (under `sdks/`) when used by the program.
- **Syncing**: Always run `pulumi install` after package updates or cloning.

## 5. CI/CD & Operations
- **Pipelines**: PRs must run `pulumi preview`. Merges to protected branches run `pulumi up`.
- **Determinism**: Use update plans (`--save-plan` and `--plan`) for strict change control.
- **Auth**: Authenticate clouds via OIDC. Avoid long-lived static credentials.
- **Drift**: Detect drift regularly with `pulumi refresh --preview-only`. Treat drift as a production incident.
- **GitHub Actions**: Prefer `pulumi/actions` over the deprecated `setup-pulumi`.

## 6. Testing & Governance
- **Unit Tests**: Use `go test` with Pulumi mocks to validate transformation and business logic.
- **Integration Tests**: Use `integration.ProgramTest` for full lifecycle validation and post-deploy assertions.
- **Policy**: Enforce policy checks on local and CI previews before merging using pre-built and custom policy packs.
