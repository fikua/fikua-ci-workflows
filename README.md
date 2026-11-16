# fikua-ci-workflows

Shared GitHub Actions workflows for the repositories of the `fikua` organization. Every repository calls them instead of keeping its own copy, so the repository baseline (ADR 0011) changes in one place. Decision: ADR 0018; both live in `fikua-docs` (private).

| Workflow | What it does | Secrets | Caller grants |
| --- | --- | --- | --- |
| `go-ci.yml` | gofmt, `go vet`, `go build`, tests (optional throwaway PostgreSQL, extra checks) | — | `contents: read` |
| `java-ci.yml` | Gradle build and tests; optional Sonar through the `org.sonarqube` Gradle plugin | `SONAR_TOKEN` (with `sonar`) | `contents: read`, `pull-requests: read` |
| `node-ci.yml` | `npm ci`, build, tests, extra checks; optional Sonar scan | `SONAR_TOKEN` (with `sonar`) | `contents: read`, `pull-requests: read` |
| `sonar.yml` | Go tests with coverage + SonarQube Cloud scan | `SONAR_TOKEN` | `contents: read`, `pull-requests: read` |
| `image-release.yml` | multi-arch image to `ghcr.io/fikua/<image>` (+ Docker Hub `fikua/<image>` while ADR 0015 lasts) | `DOCKER_USERNAME`, `DOCKER_TOKEN` (with `dockerhub`) | `contents: read`, `packages: write` |
| `codeql.yml` | CodeQL, `security-and-quality` queries (public repositories only) | — | `contents: read`, `security-events: write` |
| `gitleaks.yml` | secret scan of the whole git history | — | `contents: read` |

Inputs and their defaults are documented at the top of each workflow.

## Rules for callers

- **Pin by commit SHA**, with the release in a comment: `uses: fikua/fikua-ci-workflows/.github/workflows/go-ci.yml@<sha>  # v1.1.0`. Dependabot (`github-actions` ecosystem) opens the pull request when a new release appears, so a change reaches every repository as one reviewed PR per repository, never silently.
- **Pass secrets explicitly**, never `secrets: inherit`: each workflow gets only the secrets it uses.
- The release job `needs` the test job.

## Examples

Replace `<sha>` with the commit of the latest release (`git ls-remote https://github.com/fikua/fikua-ci-workflows refs/tags/v1.1.0^{}`).

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
    uses: fikua/fikua-ci-workflows/.github/workflows/go-ci.yml@<sha>  # v1.1.0
    with:
      postgres: true

  gitleaks:
    if: github.event_name == 'push' || github.event_name == 'pull_request'
    uses: fikua/fikua-ci-workflows/.github/workflows/gitleaks.yml@<sha>  # v1.1.0

  sonar:
    if: github.event_name == 'push' || github.event_name == 'pull_request'
    permissions:
      contents: read
      pull-requests: read
    uses: fikua/fikua-ci-workflows/.github/workflows/sonar.yml@<sha>  # v1.1.0
    with:
      postgres: true
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  release:
    needs: test
    if: github.event_name != 'pull_request'
    permissions:
      contents: read
      packages: write
    uses: fikua/fikua-ci-workflows/.github/workflows/image-release.yml@<sha>  # v1.1.0
    with:
      image: my-service
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_TOKEN: ${{ secrets.DOCKER_TOKEN }}
```

A Java service: the `test` job becomes

```yaml
  test:
    permissions:
      contents: read
      pull-requests: read
    uses: fikua/fikua-ci-workflows/.github/workflows/java-ci.yml@<sha>  # v1.1.0
    with:
      sonar: ${{ github.event_name == 'push' || github.event_name == 'pull_request' }}
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

and its Dockerfile compiles in a stage pinned to the build platform (`FROM --platform=$BUILDPLATFORM gradle:… AS build`): the JAR is the same for every architecture, so Gradle never runs under QEMU.

A frontend: `node-ci.yml` with `test-command` writing the coverage report Sonar reads (e.g. `npx vitest run --coverage`) and `extra-checks: npx tsc --noEmit`.

CodeQL in its own `.github/workflows/codeql.yml`, for the weekly schedule:

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
    uses: fikua/fikua-ci-workflows/.github/workflows/codeql.yml@<sha>  # v1.1.0
    with:
      languages: '["go", "actions"]'   # or java-kotlin, javascript-typescript
```

And Dependabot, so the pins move with the releases — `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

### Tests against PostgreSQL

With `postgres: true` the Go jobs start `postgres:17-alpine` and export `TEST_DATABASE_URL` (user, password and database `test`). Tests that need a database read that variable and skip without it.

## Versions

- Every release is a tag `v1.Y.Z`. A change that breaks a caller (an input renamed or removed, a new required secret, a different default) is a new major: `v2.0.0`.
- Third-party actions and images inside the workflows are pinned to a commit SHA or digest; Dependabot updates them here.

Releasing:

```sh
git tag -a v1.Y.Z -m "v1.Y.Z: <what changed>"
git push origin v1.Y.Z
```

## Development

```sh
pre-commit install   # gitleaks before every commit
actionlint           # the same check CI runs
```

A change can be tried from a consumer before releasing it by pointing its `uses:` line at a branch in a pull request there.

## License

MIT, see [LICENSE](LICENSE). Contact: <hola@fikua.com>.
