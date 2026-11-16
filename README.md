# fikua-ci-workflows

Shared GitHub Actions workflows for the repositories of the `fikua` organization. Every repository calls them instead of keeping its own copy, so the repository baseline (ADR 0011) changes in one place. Decision: ADR 0018; both live in `fikua-docs` (private).

| Workflow | What it does | Caller grants |
| --- | --- | --- |
| `go-ci.yml` | gofmt, `go vet`, `go build`, tests (optional throwaway PostgreSQL, extra checks) | `contents: read` |
| `image-release.yml` | multi-arch image to `ghcr.io/fikua/<image>` (+ Docker Hub `fikua/<image>` while ADR 0015 lasts) | `contents: read`, `packages: write` |
| `codeql.yml` | CodeQL, `security-and-quality` queries (public repositories only) | `contents: read`, `security-events: write` |
| `sonar.yml` | tests with coverage + SonarQube Cloud scan | `contents: read`, `pull-requests: read` |
| `gitleaks.yml` | secret scan of the whole git history | `contents: read` |

## Using them

A Go service with a database, an image and the analysis — `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
    paths-ignore: ['**.md']
  pull_request:
  release:
    types: [published]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  test:
    uses: fikua/fikua-ci-workflows/.github/workflows/go-ci.yml@v1
    with:
      postgres: true

  gitleaks:
    if: github.event_name == 'push' || github.event_name == 'pull_request'
    uses: fikua/fikua-ci-workflows/.github/workflows/gitleaks.yml@v1

  sonar:
    if: github.event_name == 'push' || github.event_name == 'pull_request'
    permissions:
      contents: read
      pull-requests: read
    uses: fikua/fikua-ci-workflows/.github/workflows/sonar.yml@v1
    with:
      postgres: true
    secrets: inherit

  release:
    needs: test
    if: github.event_name != 'pull_request'
    permissions:
      contents: read
      packages: write
    uses: fikua/fikua-ci-workflows/.github/workflows/image-release.yml@v1
    with:
      image: my-service
    secrets: inherit
```

And `.github/workflows/codeql.yml` (its own file, for the weekly schedule):

```yaml
name: CodeQL
on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '23 4 * * 1'

permissions:
  contents: read

jobs:
  codeql:
    permissions:
      contents: read
      security-events: write
    uses: fikua/fikua-ci-workflows/.github/workflows/codeql.yml@v1
```

Inputs and their defaults are documented at the top of each workflow.

### Tests against PostgreSQL

With `postgres: true` the job starts `postgres:17-alpine` and exports `TEST_DATABASE_URL` (user, password and database `test`). Tests that need a database read that variable and skip without it.

## Versions

- Consumers call `@v1`, a tag that moves to the latest compatible release.
- Every release is also tagged `v1.Y.Z`. A change that breaks a caller (an input renamed or removed, a new required secret, a different default) is a new major: `v2`.
- Third-party actions and images inside the workflows are pinned to a commit SHA or digest; Dependabot updates them here, once for every repository.

Releasing:

```sh
git tag -a v1.Y.Z -m "v1.Y.Z: <what changed>"
git tag -f v1 v1.Y.Z
git push origin v1.Y.Z
git push -f origin v1
```

## Development

```sh
pre-commit install   # gitleaks before every commit
actionlint           # the same check CI runs
```

A change can be tried from a consumer before releasing it by pointing the `uses:` line at a branch (`@my-branch`) in a pull request there.

## License

MIT, see [LICENSE](LICENSE). Contact: hola@fikua.com.
