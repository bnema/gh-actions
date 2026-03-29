# gh-actions

Shared GitHub Actions workflows. Add CI, releases, and deploys to a repo with a few lines of YAML.

## Workflows

| Workflow | What it does |
|----------|-------------|
| `go-ci.yml` | Lint with golangci-lint v2 + run tests |
| `go-release.yml` | GoReleaser build on tag push (supports pre-releases) |
| `frontend-ci.yml` | svelte-check, tsc, and/or ESLint (npm or bun) |
| `dependabot-auto-merge.yml` | Auto-merge patch/minor Dependabot PRs |
| `discord-notify.yml` | Send Discord embed notifications via webhook |

## Usage

All workflows use `workflow_call`. Point your repo's workflow at one of these with `uses:`.

### Go CI

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

If you need to run something before lint/test (code generation, frontend build), use `pre-command`:

```yaml
    with:
      pre-command: make build-frontend
```

For projects that need a specific container (GTK apps, for example):

```yaml
    with:
      container: archlinux:latest
      pacman-packages: git go webkitgtk-6.0 gtk4 base-devel
```

### Go Release

```yaml
name: Release
on:
  push:
    tags: ['v*']

jobs:
  release:
    uses: bnema/gh-actions/.github/workflows/go-release.yml@main
    with:
      go-version: stable
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The `v*` pattern matches stable tags (`v1.0.0`) and pre-release tags (`v1.0.0-rc.1`, `v1.0.0-alpha.1`). GoReleaser handles the distinction if your `.goreleaser.yaml` includes:

```yaml
release:
  prerelease: auto
git:
  tag_sort: smartsemver
```

`prerelease: auto` reads the semver suffix and marks the GitHub release accordingly. `smartsemver` (GoReleaser v2.12+) generates changelogs against the previous stable release, not the last pre-release.

### Frontend CI

Works for SvelteKit, plain TypeScript, or both. Each check is toggled independently.

SvelteKit with ESLint:
```yaml
jobs:
  check:
    uses: bnema/gh-actions/.github/workflows/frontend-ci.yml@main
    with:
      package-manager: bun
      run-svelte-check: true
      run-eslint: true
```

TypeScript only:
```yaml
    with:
      run-svelte-check: false
      run-tsc: true
      run-eslint: true
```

ESLint uses the project's own `eslint.config.js`. If that config includes `eslint-plugin-svelte`, Svelte files get linted. No workflow-side config needed.

### Gordon Deploy

Gordon ships its own deploy action at [`bnema/gordon/.github/actions/deploy`](https://github.com/bnema/gordon). Use it directly:

```yaml
name: Gordon Deploy
on:
  push:
    tags: ['v*']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: bnema/gordon/.github/actions/deploy@main
        with:
          registry: ${{ secrets.GORDON_REGISTRY }}
          username: ${{ secrets.GORDON_USERNAME }}
          password: ${{ secrets.GORDON_TOKEN }}
          image: my-app
```

Secrets setup:
- `GORDON_REGISTRY` — registry hostname (e.g. `registry.mydomain.com`)
- `GORDON_USERNAME` — token subject
- `GORDON_TOKEN` — JWT from `gordon auth token generate --subject <name> --scopes push --expiry 0`

### Dependabot Auto-Merge

```yaml
name: Dependabot Auto-Merge
on: pull_request

jobs:
  auto-merge:
    uses: bnema/gh-actions/.github/workflows/dependabot-auto-merge.yml@main
```

Merges patch and minor updates automatically. Major versions require manual review.

## Inputs Reference

### go-ci.yml

| Input | Default | Description |
|-------|---------|-------------|
| `go-version` | `stable` | Go version |
| `lint-timeout` | `5m` | golangci-lint timeout |
| `lint-version` | `v2.8.0` | golangci-lint version |
| `test-flags` | `-race -v ./...` | Flags for `go test` |
| `pre-command` | | Run before lint and test |
| `container` | | Container image to run in |
| `apt-packages` | | Space-separated apt packages |
| `pacman-packages` | | Space-separated pacman packages |

### go-release.yml

| Input | Default | Description |
|-------|---------|-------------|
| `go-version` | `stable` | Go version |
| `goreleaser-version` | `~> v2` | GoReleaser version constraint |
| `goreleaser-args` | `release --clean` | GoReleaser CLI arguments |
| `node-version` | | Node.js version (skip if empty) |
| `apt-packages` | | Space-separated apt packages |
| `pre-command` | | Run before GoReleaser |

Requires `GITHUB_TOKEN` secret.

### frontend-ci.yml

| Input | Default | Description |
|-------|---------|-------------|
| `node-version` | `24` | Node.js version |
| `package-manager` | `npm` | `npm` or `bun` |
| `working-directory` | `.` | Directory with package.json |
| `run-svelte-check` | `true` | Run svelte-check |
| `run-tsc` | `false` | Run tsc --noEmit |
| `run-eslint` | `true` | Run ESLint |
| `eslint-args` | `src` | Arguments passed to eslint |

### dependabot-auto-merge.yml

| Input | Default | Description |
|-------|---------|-------------|
| `merge-method` | `squash` | squash, merge, or rebase |

## golangci-lint v2 Starter Config

A starting point for new Go projects. Copy and adapt per project.

```yaml
version: "2"

run:
  timeout: 5m
  tests: true

linters:
  default: none
  enable:
    - govet
    - errcheck
    - staticcheck
    - unused
    - ineffassign
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
    - gosec
    - modernize
    - fatcontext
    - intrange
    - copyloopvar
    - exptostd
    - errorlint
    - bodyclose
    - mnd
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
      enable: [nilness, shadow]
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

Notable v2 changes from v1: formatters live in their own `formatters:` section, `default: none|standard|all|fast` replaces `enable-all`, and `exclusions:` replaces `issues.exclude-rules`.

New linters worth knowing about:

- `modernize` flags patterns that have simpler alternatives in recent Go (range-over-int, slices package, etc.)
- `fatcontext` catches context.WithValue calls inside loops
- `intrange` suggests `for range N` syntax from Go 1.22
- `copyloopvar` flags unnecessary loop variable copies (Go 1.22 changed loop variable semantics)
- `exptostd` finds `golang.org/x/exp` functions that now exist in the standard library

## What stays in each repo

These workflows handle the shared CI/CD plumbing. Project-specific configuration stays local:

- `.golangci.yml` for linter rules and exclusions
- `.goreleaser.yaml` for build targets, archives, and release notes
- `eslint.config.js` for ESLint rules and plugins
- Specialized workflows like AUR publishing or Flatpak builds
