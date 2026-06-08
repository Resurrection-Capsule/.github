# Releasing & CI standard (Resurrection-Capsule)

This repo (`Resurrection-Capsule/.github`) holds the **reusable GitHub Actions workflows** every
.NET repo in the org calls, plus the MSBuild/NuGet templates. One place to fix; everyone inherits.

## Reusable workflows

Reference them from a repo's own `.github/workflows/*.yml`, pinned to a tag (`@v1`):

| Workflow | Purpose | Trigger in the caller |
|---|---|---|
| `dotnet-ci.yml` | restore → build → test | `push` / `pull_request` |
| `dotnet-nuget.yml` | build → test → pack → push to GitHub Packages | `push` tag `v*` |
| `dotnet-app-release.yml` | self-contained publish per RID → GitHub Release | `push` tag `v*` |

Callers must pass `secrets: inherit` so `GITHUB_TOKEN` reaches the pack/push job.

### Library repo (e.g. RakNexus) — `.github/workflows/`

```yaml
# ci.yml
on: { push: { branches: [main] }, pull_request: { branches: [main] } }
jobs:
  ci:
    uses: Resurrection-Capsule/.github/.github/workflows/dotnet-ci.yml@v1
    with: { solution: RakNexus.sln }
    secrets: inherit
```

```yaml
# release.yml
on: { push: { tags: ['v*'] } }
jobs:
  publish:
    uses: Resurrection-Capsule/.github/.github/workflows/dotnet-nuget.yml@v1
    with: { project: src/RakNexus.csproj, solution: RakNexus.sln }
    secrets: inherit
```

## Versioning — MinVer (automatic)

No hand-edited version numbers. The package version is derived from the nearest `vX.Y.Z` git tag
(`MinVerTagPrefix=v`). Untagged commits get a height-based pre-release (e.g. `1.2.1-alpha.3`).

**Cut a release:**
```bash
git tag v1.2.0
git push origin v1.2.0
```
That fires `release.yml` → pack + push to GitHub Packages, and (for apps) a GitHub Release with
auto-generated notes. Pre-release tags (any `-`, e.g. `v1.2.0-rc.1`) are marked pre-release.

> MinVer needs full history — the reusable workflows already `checkout` with `fetch-depth: 0`.

## Per-repo setup

1. Copy `templates/Directory.Build.props` to the repo root; set `RepositoryUrl`.
2. Each **library** `.csproj`: `<IsPackable>true</IsPackable>` + `PackageId` + `Description` + `PackageTags`.
   Test/app projects: `<IsPackable>false</IsPackable>`.
3. Add a `README.md` (packed into the `.nupkg`) and an MIT `LICENSE`.
4. Add the two caller workflows above.

## Consuming org packages

Copy `templates/nuget.config` to the consumer repo root (server, Hub). It declares the org
GitHub Packages feed. **Credentials are never committed** — CI injects them via the
`NuGetPackageSourceCredentials_github` env var (`Username=actions;Password=$GITHUB_TOKEN`).
Locally, set that env var or register a PAT (scope `read:packages`):

```bash
dotnet nuget add source https://nuget.pkg.github.com/Resurrection-Capsule/index.json \
  -n github -u <github-username> -p <PAT> --store-password-in-clear-text
```

## Bootstrapping this repo

Create `Resurrection-Capsule/.github` on GitHub, push this content, then tag it so callers can pin:
```bash
git tag v1 && git push origin v1
```
Move the `v1` tag forward as the reusable workflows evolve (callers pinned to `@v1` pick it up).
