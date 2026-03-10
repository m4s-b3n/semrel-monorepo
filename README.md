# semrel-monorepo

Monorepo releasing multiple .NET/NuGet packages independently with [semantic-release](https://github.com/semantic-release/semantic-release).

## Packages

| Package | Path |
|---------|------|
| Package.Alpha | `src/Package.Alpha` |
| Package.Beta | `src/Package.Beta` |
| Package.Gamma | `src/Package.Gamma` |

## How it works

- Each package has its own `.releaserc.json` that extends [`semantic-release-monorepo`](https://github.com/pmowrer/semantic-release-monorepo) and a shared root config.
- `semantic-release-monorepo` filters commits by changed files, so only commits touching a package's directory trigger a release for that package.
- Release type follows [conventional commits](https://www.conventionalcommits.org/) (`fix` → patch, `feat` → minor, breaking change → major). No package-specific scopes needed.

## CI/CD

On **push to `main`**: build → semantic-release → pack & publish (sequentially per package).
On **pull request**: build → semantic-release dry run → pack with PR suffix (no real release).

### Workflows

| Workflow | Purpose |
|----------|---------|
| `release.yml` | Entry point — matrix over all packages |
| `release-package.yml` | Orchestrates build → release → publish for one package |
| `build.yml` | `dotnet build` |
| `semantic-release.yml` | Runs `cycjimmy/semantic-release-action` with local plugin install |
| `publish.yml` | `dotnet pack` & `dotnet nuget push` to GitHub Packages |

### Required secrets

- `GITHUB_TOKEN` — provided automatically by GitHub Actions

### Dry-run version prediction on PRs

On pull requests, semantic-release runs in dry-run mode to predict the next version (used as a NuGet package suffix like `1.1.0-pr-42`). This requires several workarounds because semantic-release and `env-ci` are designed to abort early on PRs:

1. **`ref: github.head_ref`** — On PRs, `github.ref` is `refs/pull/N/merge` (the synthetic merge commit). `env-ci`'s `parseBranch` only strips the `refs/heads/` prefix, so it resolves the branch name as `refs/pull/N/merge` — which isn't a real remote branch. Checking out `github.head_ref` (e.g. `test/my-feature`) ensures the working copy is on the actual branch.

2. **`ci: false`** — semantic-release's core `run()` function has an early return: `if (isCi && isPr && !options.noCi) return false`. Setting `ci: false` passes `noCi: true`, bypassing this PR guard so the analysis can proceed.

3. **`unset_gha_env: true`** — Even with `ci: false`, `env-ci` still detects GitHub Actions via `GITHUB_ACTIONS` env var and reads `GITHUB_REF` to determine the branch. Since `GITHUB_REF` can't be overridden at the step level (GitHub Actions protects default env vars), the cycjimmy action's `unset_gha_env` option deletes `GITHUB_ACTIONS` from `process.env`. This makes `env-ci` fall back to its `git` service, which runs `git rev-parse --abbrev-ref HEAD` to read the branch name from the actual checkout.

4. **`branches: ["base", "head"]`** — semantic-release validates that the current branch exists in its `branches` config. The default config only includes `main` (or whatever `.releaserc.json` specifies). By providing both the base branch (for tag/version history) and the head branch (where we're running), the branch validation passes. The base branch must come first — semantic-release treats the first entry as the primary release branch, and subsequent branches get constrained version ranges based on it.
