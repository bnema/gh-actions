# gh-actions

Reusable GitHub Actions workflows for Go, frontend (Svelte/TypeScript), and deployment.

## Quick Start

### Go Project (CI + Release)

`.github/workflows/ci.yml`:
```yaml
name: CI
on:
  push:
    branches: [main]
    paths: ['**.go', 'go.mod', 'go.sum', '.golangci.yml', '.github/workflows/**']
  pull_request:
    paths: ['**.go', 'go.mod', 'go.sum', '.golangci.yml', '.github/workflows/**']

jobs:
  ci:
    uses: bnema/gh-actions/.github/workflows/go-ci.yml@main
    with:
      go-version: stable
```

`.github/workflows/release.yml`:
```yaml
name: Release
on:
  push:
    tags: ['v*']  # v1.0.0, v1.0.0-rc1, v1.0.0-alpha.1

jobs:
  release:
    uses: bnema/gh-actions/.github/workflows/go-release.yml@main
    with:
      go-version: stable
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### SvelteKit Project

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    uses: bnema/gh-actions/.github/workflows/frontend-ci.yml@main
    with:
      package-manager: bun        # or npm
      run-svelte-check: true
      run-eslint: true
```

### TypeScript Project (no Svelte)

```yaml
jobs:
  check:
    uses: bnema/gh-actions/.github/workflows/frontend-ci.yml@main
    with:
      run-svelte-check: false
      run-tsc: true
      run-eslint: true
```

### Gordon Deploy

```yaml
name: Deploy
on:
  push:
    tags: ['v*']

jobs:
  deploy:
    uses: bnema/gh-actions/.github/workflows/gordon-deploy.yml@main
    with:
      image: my-app
    secrets:
      GORDON_REGISTRY: ${{ secrets.GORDON_REGISTRY }}
      GORDON_USERNAME: ${{ secrets.GORDON_USERNAME }}
      GORDON_TOKEN: ${{ secrets.GORDON_TOKEN }}
```

### Dependabot Auto-Merge

```yaml
name: Dependabot Auto-Merge
on: pull_request

jobs:
  auto-merge:
    uses: bnema/gh-actions/.github/workflows/dependabot-auto-merge.yml@main
```

---

## Workflows Reference

### `go-ci.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `go-version` | string | `stable` | Go version |
| `lint-timeout` | string | `5m` | golangci-lint timeout |
| `lint-version` | string | `v2.8.0` | golangci-lint version |
| `test-flags` | string | `-race -v ./...` | go test flags |
| `pre-command` | string | `""` | Run before lint/test |
| `container` | string | `""` | Container image |
| `apt-packages` | string | `""` | apt packages to install |
| `pacman-packages` | string | `""` | pacman packages to install |

### `go-release.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `go-version` | string | `stable` | Go version |
| `goreleaser-version` | string | `~> v2` | GoReleaser version |
| `goreleaser-args` | string | `release --clean` | GoReleaser arguments |
| `node-version` | string | `""` | Node.js version (optional) |
| `apt-packages` | string | `""` | apt packages to install |
| `pre-command` | string | `""` | Run before goreleaser |

**Secrets:** `GITHUB_TOKEN` (required)

### `frontend-ci.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `node-version` | string | `24` | Node.js version |
| `package-manager` | string | `npm` | `npm` or `bun` |
| `working-directory` | string | `.` | Directory with package.json |
| `run-svelte-check` | boolean | `true` | Run svelte-check |
| `run-tsc` | boolean | `false` | Run tsc --noEmit |
| `run-eslint` | boolean | `true` | Run eslint |
| `eslint-args` | string | `src` | Arguments for eslint |

ESLint reads the project's own `eslint.config.js`. If the config includes `eslint-plugin-svelte`, Svelte files are linted automatically.

### `dependabot-auto-merge.yml`

| Input | Type | Default |
|-------|------|---------|
| `merge-method` | string | `squash` |

Auto-merges patch and minor dependency updates from Dependabot.

### `gordon-deploy.yml`

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `image` | string | `""` | Image name (defaults to repo name) |
| `tag` | string | `""` | Override tag |
| `dockerfile` | string | `./Dockerfile` | Dockerfile path |
| `context` | string | `.` | Build context |
| `build-args` | string | `""` | Build arguments |
| `platforms` | string | `""` | Target platforms |
| `push-latest` | boolean | `true` | Also push :latest |

**Secrets:** `GORDON_REGISTRY`, `GORDON_USERNAME`, `GORDON_TOKEN`

---

## Pre-release Tags

The `go-release.yml` workflow is triggered by `tags: ['v*']` in the caller, which matches both stable and pre-release tags:
- `v1.0.0` — stable release
- `v1.0.0-rc.1` — release candidate
- `v1.0.0-alpha.1` — alpha
- `v1.0.0-beta.2` — beta

Add this to your `.goreleaser.yaml` for correct pre-release handling:

```yaml
release:
  prerelease: auto       # auto-detect from tag suffix
git:
  tag_sort: smartsemver  # correct changelog for mixed stable/pre-release
```

`prerelease: auto` detects semver pre-release suffixes and marks the GitHub release as a pre-release. `smartsemver` (GoReleaser v2.12+) ensures changelogs compare against the previous stable release, not the last pre-release.

---

## Recommended `.golangci.yml` (v2 starter)

```yaml
version: "2"

run:
  timeout: 5m
  tests: true

linters:
  default: none
  enable:
    # Standard
    - govet
    - errcheck
    - staticcheck
    - unused
    - ineffassign
    # Quality
    - revive
    - gocritic
    - gocyclo
    - funlen
    - dupl
    - goconst
    - unconvert
    - unparam
    - misspell
    - whitespace
    - nolintlint
    # Security
    - gosec
    # Modern Go (v2.0+)
    - modernize
    - fatcontext
    - intrange
    - copyloopvar
    - exptostd
    # Error handling
    - errorlint
    # HTTP
    - bodyclose
    # Magic numbers
    - mnd
    # Line length
    - lll

  settings:
    gocyclo:
      min-complexity: 20
    lll:
      line-length: 140
    mnd:
      checks: [argument, case, condition, return]
      ignored-numbers: ['0', '1', '2', '3', '10', '100']
    govet:
      enable:
        - nilness
        - shadow
    errorlint:
      asserts: false
    nolintlint:
      allow-unused: false
      require-explanation: true
      require-specific: true
    revive:
      rules:
        - name: indent-error-flow
        - name: unused-parameter
        - name: unused-receiver

  exclusions:
    generated: lax
    rules:
      - path: _test\.go
        linters: [dupl, mnd, lll, funlen, goconst, errcheck, gosec]

formatters:
  enable:
    - gofmt

issues:
  max-same-issues: 50

severity:
  default: error
```

### Key v2 changes
- **`formatters:` section** — formatters (gofmt, goimports) are separate from linters
- **`default: none|standard|all|fast`** — replaces `enable-all`
- **`exclusions:`** — replaces `issues.exclude-rules`

### New linters worth enabling
| Linter | Description |
|--------|-------------|
| `modernize` | Suggests modern Go patterns (slices.Sort, range-over-int, etc.) |
| `fatcontext` | Detects nested context.WithValue in loops |
| `intrange` | Suggests `for range N` (Go 1.22+) |
| `copyloopvar` | Flags unnecessary loop variable copies |
| `exptostd` | Flags `x/exp` functions replaceable by stdlib |

---

## Docker Deploy Action

The `actions/docker-deploy` composite action can be used directly for non-Gordon registries:

```yaml
- uses: bnema/gh-actions/actions/docker-deploy@main
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
    image: my-app
    platforms: linux/amd64,linux/arm64
```

---

## Per-repo Customization

Each consumer repo keeps its own:
- **`.golangci.yml`** — project-specific linter rules and exclusions
- **`.goreleaser.yaml`** — build targets, archives, release notes
- **`eslint.config.js`** — ESLint rules and plugins
- **Specialized workflows** — AUR publishing, Flatpak builds, etc.

The reusable workflows handle the common CI/CD plumbing; project-specific config stays local.
