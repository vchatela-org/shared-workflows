# Shared workflows

| Workflow | What it does |
|---|---|
| [`docker-build-push-harbor.yml`](#reusable-workflow-docker-build-push--scan-harbor) | Builds an image, pushes it to Harbor with OCI labels, gates on Harbor's vulnerability scan |
| [`release-please.yml`](#reusable-workflow-release-pr-release-please) | Keeps a release PR open; merging it tags, publishes the GitHub Release and starts the image builds |

Together they put a version's release notes in front of whoever bumps it: Renovate and
Dependabot read the image's `org.opencontainers.image.source` label to find the repository,
then show the GitHub Release notes of every version the bump skips.

# Reusable Workflow: Docker Build, Push & Scan (Harbor)

## Setup

This reusable workflow should live in a **dedicated shared repository**, e.g.:

```
vchatela-org/shared-workflows
└── .github/
    └── workflows/
        └── docker-build-push-harbor.yml   ← this file
```

1. Create the repo `vchatela-org/shared-workflows`
2. Copy `docker-build-push-harbor.yml` into `.github/workflows/`
3. Make the repo accessible to your other repos:
   - **Private org repos**: Go to the shared repo → Settings → Actions → General → "Access" → Allow access from other repos in the org
   - Or make the shared repo `internal` visibility

## Usage (in any consuming repo)

Create `.github/workflows/docker-build-push.yml`:

```yaml
name: Build and Push Docker Image to Harbor

on:
  push:
    branches: [ 'main' ]
    tags: [ 'v*' ]

jobs:
  docker-build-push-scan:
    uses: vchatela-org/shared-workflows/.github/workflows/docker-build-push-harbor.yml@v1
    with:
      harbor_registry: ${{ vars.HARBOR_REGISTRY }}
      harbor_project: ${{ vars.HARBOR_PROJECT }}
      harbor_image: ${{ vars.HARBOR_IMAGE }}
    secrets:
      harbor_username: ${{ secrets.HARBOR_USERNAME }}
      harbor_password: ${{ secrets.HARBOR_PASSWORD }}
```

## Versioning

Pin to a tag rather than `@main`, so a change here can never break a consuming
repo's build without you choosing to take it.

| Ref | Moves? | Use when |
|---|---|---|
| `@v1` | Yes — follows the latest `v1.x.y` | **Recommended.** Bugfixes and backwards-compatible features arrive automatically. |
| `@v1.0.0` | No | You need a fully reproducible build and will bump manually. |
| `@main` | Yes — every push | Testing an unreleased change only. Not for production. |

Releases follow [semantic versioning](https://semver.org/):

- **Patch** (`v1.0.1`) — bugfixes, no interface change.
- **Minor** (`v1.1.0`) — new optional inputs or outputs; existing callers keep working.
- **Major** (`v2.0.0`) — a breaking change, such as a removed/renamed input, a new
  required input, or a changed default that alters behaviour. A new `v2` tag is
  published and `v1` stops moving, so `@v1` callers are unaffected until they opt in.

### Cutting a release

```bash
git tag -a v1.1.0 -m "Add <thing>"      # immutable release tag
git tag -f -a v1 -m "v1 -> v1.1.0"      # move the floating major tag
git push origin v1.1.0
git push origin v1 --force              # required: v1 is a moving ref
```

## Image labels

Every image carries the OCI labels of [docker/metadata-action](https://github.com/docker/metadata-action):
`org.opencontainers.image.source` (the repository URL), `revision` (the commit), `version` (the tag,
or `dev`), `created`, `title`, `url`, `description` and `licenses`. Nothing to configure. The source
label is what lets Renovate and Dependabot link a Harbor image to its repository's release notes.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `harbor_registry` | Yes | — | Harbor registry host |
| `harbor_project` | Yes | — | Harbor project name |
| `harbor_image` | Yes | — | Image name |
| `runs_on` | No | `arc-runner-set` | Runner label |
| `dockerfile` | No | `.` | Docker build context path |
| `buildx_driver` | No | `kubernetes` | Buildx driver |
| `buildx_driver_opts` | No | *(kubernetes defaults)* | Buildx driver options |
| `scan_enabled` | No | `true` | Enable vulnerability scan |
| `scan_timeout_seconds` | No | `300` | Scan wait timeout |
| `max_critical_vulns` | No | `0` | Fail threshold for critical vulns |
| `max_high_vulns` | No | `5` | Fail threshold for high vulns |

## Secrets

| Secret | Required | Description |
|---|---|---|
| `harbor_username` | Yes | Harbor robot account username |
| `harbor_password` | Yes | Harbor robot account password |

## Outputs

| Output | Description |
|---|---|
| `image_digest` | Image digest from the build |
| `image_tag` | Tag applied to the image |
| `full_image_ref` | Full `registry/project/image:tag` reference |
| `scan_status` | Vulnerability scan status |
| `scan_critical` | Number of critical vulnerabilities |
| `scan_high` | Number of high vulnerabilities |

## Examples

### Minimal (defaults)
```yaml
jobs:
  build:
    uses: vchatela-org/shared-workflows/.github/workflows/docker-build-push-harbor.yml@v1
    with:
      harbor_registry: ${{ vars.HARBOR_REGISTRY }}
      harbor_project: ${{ vars.HARBOR_PROJECT }}
      harbor_image: ${{ vars.HARBOR_IMAGE }}
    secrets:
      harbor_username: ${{ secrets.HARBOR_USERNAME }}
      harbor_password: ${{ secrets.HARBOR_PASSWORD }}
```

### Skip scan, custom runner
```yaml
jobs:
  build:
    uses: vchatela-org/shared-workflows/.github/workflows/docker-build-push-harbor.yml@v1
    with:
      harbor_registry: ${{ vars.HARBOR_REGISTRY }}
      harbor_project: ${{ vars.HARBOR_PROJECT }}
      harbor_image: ${{ vars.HARBOR_IMAGE }}
      scan_enabled: false
      runs_on: ubuntu-latest
    secrets:
      harbor_username: ${{ secrets.HARBOR_USERNAME }}
      harbor_password: ${{ secrets.HARBOR_PASSWORD }}
```

### Strict security policy
```yaml
jobs:
  build:
    uses: vchatela-org/shared-workflows/.github/workflows/docker-build-push-harbor.yml@v1
    with:
      harbor_registry: ${{ vars.HARBOR_REGISTRY }}
      harbor_project: ${{ vars.HARBOR_PROJECT }}
      harbor_image: ${{ vars.HARBOR_IMAGE }}
      max_critical_vulns: 0
      max_high_vulns: 0
    secrets:
      harbor_username: ${{ secrets.HARBOR_USERNAME }}
      harbor_password: ${{ secrets.HARBOR_PASSWORD }}
```

### Using outputs in a downstream job
```yaml
jobs:
  build:
    uses: vchatela-org/shared-workflows/.github/workflows/docker-build-push-harbor.yml@v1
    with:
      harbor_registry: ${{ vars.HARBOR_REGISTRY }}
      harbor_project: ${{ vars.HARBOR_PROJECT }}
      harbor_image: ${{ vars.HARBOR_IMAGE }}
    secrets:
      harbor_username: ${{ secrets.HARBOR_USERNAME }}
      harbor_password: ${{ secrets.HARBOR_PASSWORD }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.full_image_ref }}"
```

# Reusable Workflow: Release PR (release-please)

[release-please](https://github.com/googleapis/release-please) reads the
[conventional commits](https://www.conventionalcommits.org/) on the default branch and keeps
**one release PR** open with the next version and its `CHANGELOG.md` entry. Features and fixes
pile up in it until you merge it; merging it is the release:

1. release-please tags `vX.Y.Z` and publishes the GitHub Release with the same notes;
2. this workflow starts the caller's build workflows on that tag (`workflow_dispatch`, the one
   event the default `GITHUB_TOKEN` may trigger: a tag it creates starts nothing by itself), so
   no personal token is needed and the builds see the tag as their ref, as on a pushed tag.

## Setup (once per repository)

- **Settings → Actions → General → "Allow GitHub Actions to create and approve pull requests"**
  must be on, in the organization and in the repository, or release-please cannot open its PR.
- Each build workflow listed in `build_workflows` must have `workflow_dispatch:` among its triggers.
- Add three files.

`.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    branches: [ 'main' ]
  workflow_dispatch:

permissions:
  contents: write
  issues: write
  pull-requests: write
  actions: write

jobs:
  release:
    uses: vchatela-org/shared-workflows/.github/workflows/release-please.yml@v1
    with:
      build_workflows: docker-build-push.yml ghcr-publish.yml
```

`.release-please-manifest.json`, the current version (the latest tag, without its `v`):

```json
{ ".": "0.9.0" }
```

`release-please-config.json`:

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "release-type": "simple",
  "include-component-in-tag": false,
  "bump-minor-pre-major": true,
  "bump-patch-for-minor-pre-major": false,
  "changelog-sections": [
    { "type": "feat", "section": "Features" },
    { "type": "fix", "section": "Bug Fixes" },
    { "type": "perf", "section": "Performance" },
    { "type": "revert", "section": "Reverts" },
    { "type": "deps", "section": "Dependencies" },
    { "type": "docs", "section": "Documentation", "hidden": true },
    { "type": "refactor", "section": "Refactoring", "hidden": true },
    { "type": "test", "section": "Tests", "hidden": true },
    { "type": "build", "section": "Build", "hidden": true },
    { "type": "ci", "section": "CI", "hidden": true },
    { "type": "chore", "section": "Chores", "hidden": true },
    { "type": "style", "section": "Style", "hidden": true }
  ],
  "packages": {
    ".": {
      "version-file": "VERSION",
      "extra-files": [
        { "type": "generic", "path": "app/__version__.py" }
      ]
    }
  }
}
```

- `version-file` is the file holding only the version (`simple` type); `extra-files` lists any
  other file carrying it, each marked on that line with a `x-release-please-version` comment,
  e.g. `__version__ = "0.9.0"  # x-release-please-version`. Drop either key if the repository
  has no such file. `release-type` can also be `python` or `node` for a package manifest.
- **The first release PR starts at the manifest's version**: release-please looks for that
  version's GitHub Release, then for its tag (`v0.9.0`), and only collects the commits after it.
  No GitHub Release needs to exist beforehand.
- **Versions while 0.x**: a `feat` bumps the minor, any other listed type the patch, and a
  breaking change (`feat!:` or a `BREAKING CHANGE:` footer) the minor (`bump-minor-pre-major`).
  A `Release-As: X.Y.Z` footer forces the version. Hidden types never make a release on their own.
- **Dependencies**: have Dependabot prefix runtime bumps with `deps` and the rest with `chore`/`ci`,
  so only what ships in the image reaches the notes:

  ```yaml
      commit-message:
        prefix: "deps"
        prefix-development: "chore"
  ```

Once this is in, don't bump the version or tag by hand: merge the release PR. If a build fails
to start, re-run the failed `dispatch-builds` job; the release is already there.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `build_workflows` | No | `""` | Caller workflow files to start on the new tag, space-separated |
| `config_file` | No | `release-please-config.json` | release-please config |
| `manifest_file` | No | `.release-please-manifest.json` | release-please manifest |
| `runs_on` | No | `ubuntu-latest` | Runner label (needs the `gh` CLI) |

## Outputs

| Output | Description |
|---|---|
| `release_created` | `true` when the run created a release |
| `tag_name` | The new tag (e.g. `v1.4.0`) |
| `version` | The new version (e.g. `1.4.0`) |

## Composite Action vs Reusable Workflow

This project uses a **reusable workflow** (`workflow_call`) rather than a composite action because:

- The pipeline has **multiple jobs** (build → scan) with outputs flowing between them
- Composite actions are limited to **steps within a single job**
- Reusable workflows natively support **secrets** (composite actions require passing them as inputs)
- Reusable workflows preserve **job-level parallelism** and **conditional execution**

Use composite actions when you need a reusable *building block* within a job (e.g., "setup Harbor CLI"). Use reusable workflows when you need a reusable *entire pipeline*.
