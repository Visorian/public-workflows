# public-workflows

Dependency updates for visorian's public repositories: a Renovate preset and the reusable bun security fixes workflow. Public repositories reference only this repository.

## Renovate preset

A public repository's `renovate.json` extends [`default.json`](default.json) and adds only the rules no other repository needs:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>visorian/public-workflows"]
}
```

- Updates arrive weekly, once a release is 7 days old. Security fixes from Dependabot alerts skip both the schedule and the age.
- Bun dependencies raise in-range updates too (`rangeStrategy: bump`), since the bun manager reads no locked versions. Minor and patch updates share one pull request, and each major waits for approval on the Dependency Dashboard. Lockfile maintenance runs weekly.
- Bun follows the release nixpkgs-unstable ships, looked up through Repology. `packageManager`, an exact `engines.bun` and `ARG BUN_VERSION=` in Dockerfiles move together in one pull request, and a Dockerfile takes its `oven/bun` tag from that argument. Repology has no release dates, so nixpkgs packaging is the delay instead of the release age.
- GitHub Actions are pinned by digest, and actions and images share one pull request. That includes the callers' pin of the bun security fixes workflow below.
- Pull request bodies carry no Mend merge confidence badges. Their image URLs would send package names and versions to developer.mend.io, so the preset lists `config:recommended`'s parts without that one.
- `@visorian/*` packages skip the release age and the weekly schedule, so a new code-standards release arrives at the next Renovate run. Each consuming repository also lists them in `minimumReleaseAgeExcludes` in `bunfig.toml`, or bun refuses to install the fresh version.

Each bun repository also needs a `bunfig.toml` with the same 7 days, because Renovate's release age only covers the versions it proposes, not what bun resolves around them or what someone installs by hand:

```toml
[install]
minimumReleaseAge = 604800
minimumReleaseAgeExcludes = ["@visorian/code-standards"]
```

A fresh security fix that Renovate proposes then fails to install. Add its exact `name@version` to `minimumReleaseAgeExcludes` on that pull request's branch. Every workflow reads bun's version from `packageManager` with `bun-version-file: package.json`.

## Bun security fixes

[`bun-security-fixes.yml`](.github/workflows/bun-security-fixes.yml) runs `bun audit fix` against the default branch and keeps one pull request, `security-fixes/bun`, with the result. Renovate never sees these advisories. GitHub's dependency graph doesn't read `bun.lock`, so an advisory in a transitive package raises no Dependabot alert.

- Fixed versions skip the 7-day `minimumReleaseAge` in the caller's `bunfig.toml`. A fresh package that a fix newly needs doesn't, so the run fails and its log names that package. Add its exact `name@version` to `minimumReleaseAgeExcludes`.
- The workflow applies only fixes inside the declared ranges, plus bumps of exact pins in `package.json`. An advisory "blocked by a dependent's range" needs an update of its parent package, which Renovate proposes.
- Each run rebuilds the pull request on the latest default branch and closes it once nothing is left to fix. While the pull request is open, the workflow leaves the branch alone after anyone else pushes to it. A branch left over from a merged or closed pull request gets rebuilt.
- The run fails while an advisory at or above `audit-level` has no fix.
- The audit job runs bun without any secret. Only the pull request job holds the App token, and it executes no repository code. It commits through the API, so GitHub signs the commit.
- Both jobs take about 10 seconds and run on the 1-core `ubuntu-slim` runner.

Add this as `.github/workflows/security-fixes.yml`, with the current commit of this repository's `main` in place of `<commit>`:

```yaml
name: Security fixes

on:
  schedule:
    - cron: "17 */12 * * *"
  push:
    branches:
      - main
    paths:
      - bun.lock
  workflow_dispatch:

permissions: {}

jobs:
  bun:
    name: Bun
    permissions:
      contents: read
      pull-requests: read
    uses: visorian/public-workflows/.github/workflows/bun-security-fixes.yml@<commit> # main
    with:
      app-client-id: ${{ vars.DEPENDENCY_UPDATES_APP_CLIENT_ID }}
    secrets:
      app-private-key: ${{ secrets.DEPENDENCY_UPDATES_APP_PRIVATE_KEY }}
```

Callers pin a commit, as the repositories' workflow policies require. Renovate moves the pin to the newest commit on `main` in each repository's weekly actions pull request.

| Input | Default | |
| --- | --- | --- |
| `app-client-id` | | Client ID of the App that opens the pull request; required unless `dry-run` |
| `audit-level` | `high` | Lowest severity of a remaining advisory that fails the run |
| `dry-run` | `false` | Report the fixes without opening a pull request |
| `runs-on` | `ubuntu-slim` | Runner label for both jobs |
| `working-directory` | `.` | Directory that holds `package.json` and `bun.lock` |

The secret `app-private-key` is the private key of an App with read and write access to contents and pull requests. Pull requests opened with `GITHUB_TOKEN` would need someone to approve their checks first.

Bun comes from `packageManager` in `package.json`, and `bun audit fix` needs 1.4.0 or later.

[`test.yml`](.github/workflows/test.yml) runs the workflow in dry-run mode against [`test/fixture`](test/fixture), which locks a lodash version with a known advisory. Running it manually with `dry-run` off opens a real pull request against this repository.
