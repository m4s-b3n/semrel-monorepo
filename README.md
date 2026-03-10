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
