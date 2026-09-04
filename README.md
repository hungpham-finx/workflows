# GitHub Actions shared library

Reusable workflows for AIFactory service repositories. Nothing account-specific is stored
here — every AWS identifier is a workflow input supplied by the calling repository.

## build-push-ecr.yml

Builds a multi-arch image (push-by-digest per architecture, then one OCI index carrying the
tag) and pushes it to Amazon ECR. Authentication is GitHub OIDC — no static AWS keys.

Tag convention: `<branch>-<6-char-commit>`, with `/` sanitized to `-`.

### Inputs

| Name | Required | Default | Purpose |
|---|---|---|---|
| `aws_region` | yes | | ECR region |
| `aws_account_id` | yes | | Registry account |
| `aws_role_arn` | yes | | Role assumed via OIDC |
| `ecr_repository` | yes | | ECR repository name |
| `dockerfile` | no | `./Dockerfile` | Dockerfile path |
| `build_context` | no | `.` | Build context |
| `build_args` | no | | Newline-separated `KEY=value` build args |
| `build_target` | no | | Multi-stage target stage |
| `cache_scope` | no | `ecr_repository` | GHA cache scope override for monorepos |
| `tag_prefix` | no | | Prefix added before `<branch>-<sha>` |
| `targets` | no | amd64 + arm64 | JSON array of `{arch, platform, runner}` |
| `build_timeout_minutes` | no | `90` | Per-arch build timeout |

### Outputs

`image` (full reference), `tag`, `registry`.

### Caller

```yaml
name: Build & Push to ECR

on:
  push:
    branches: [develop, "feature/**"]
    paths-ignore: ["**/*.md", "docs/**"]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    uses: hungpham-finx/workflows/.github/workflows/build-push-ecr.yml@poc
    permissions:
      id-token: write
      contents: read
    with:
      aws_region: ap-southeast-1
      aws_account_id: "366366122943"
      aws_role_arn: arn:aws:iam::366366122943:role/gfx-dev-aifactory-github-actions
      ecr_repository: aifactory-be
```

`permissions` must be declared by the caller. A reusable workflow cannot widen the token
it receives, so omitting `id-token: write` fails the OIDC assume-role step.

## lint-test.yml

One entry point for every service regardless of language. `language` selects which job
runs (`python`, `node`, `go`, `java`); the others are skipped. `pre-commit` runs alongside
unless disabled.

| Input | Required | Default | Purpose |
|---|---|---|---|
| `language` | yes | | `python` \| `node` \| `go` \| `java` |
| `version` | no | per-language | Toolchain version |
| `package_manager` | no | per-language | `uv`, `npm`, `pnpm`, `yarn`, `maven`, `gradle` |
| `working_directory` | no | `.` | Subdirectory for monorepos |
| `runner` | no | `ubuntu-latest` | Runner label |
| `run_pre_commit` | no | `true` | Also run the pre-commit gate |
| `lint_command` | no | per-language | Override lint step |
| `test_command` | no | per-language | Override test step |

```yaml
jobs:
  ci:
    uses: hungpham-finx/workflows/.github/workflows/lint-test.yml@poc
    with:
      language: python
      package_manager: uv
```

## pre-commit.yml

Standalone pre-commit gate. Inputs: `python_version` (default `3.12`),
`extra_args` (default `--all-files`).

```yaml
jobs:
  lint:
    uses: hungpham-finx/workflows/.github/workflows/pre-commit.yml@poc
```

## Versioning

Pin callers to a tag (`@v1`) or a commit SHA. Do not pin `@main` — every consumer would
pick up unreviewed commits immediately.

## OIDC note

The token minted for a reusable workflow keeps the **caller** repository in its `sub`
claim, so per-repo IAM trust conditions still apply. The `job_workflow_ref` claim
identifies this library instead, which allows one trust condition to serve every consumer
repository — verify the claim's exact format against a real token before relying on it.
