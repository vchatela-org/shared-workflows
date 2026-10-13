# Shared workflows

| Workflow | What it does |
|---|---|
| [`docker-build-push-harbor.yml`](#reusable-workflow-docker-build-push--scan-harbor) | Builds an image, pushes it to Harbor with OCI labels, gates on Harbor's vulnerability scan |
| [`release-please.yml`](#reusable-workflow-release-pr-release-please) | Keeps a release PR open; merging it tags, publishes the GitHub Release and starts the image builds |
| [`pr-title.yml`](#pr-title-check-pr-titleyml) | Fails a PR whose title is not a conventional commit (on a squash merge it is the release-note line) |

Together they put a version's release notes in front of whoever bumps it: Renovate and
Dependabot read the image's `org.opencontainers.image.source` label to find the repository,
then show the GitHub Release notes of every version the bump skips. For a private
repository, the bot's token must be able to read it (Renovate here runs with a user token
that can).

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
  workflow_dispatch:   # lets release-please.yml start it on a release tag

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
or `dev`), `created`, `title`, `url`, and `description` and `licenses` when the repository has them
(an empty one is dropped, so it never blanks the Dockerfile's or the base image's). Nothing to
configure. The source label is what lets Renovate and Dependabot link a Harbor image to its
repository's release notes.

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

## Composite Action vs Reusable Workflow

This project uses a **reusable workflow** (`workflow_call`) rather than a composite action because:

- The pipeline has **multiple jobs** (build → scan) with outputs flowing between them
- Composite actions are limited to **steps within a single job**
- Reusable workflows natively support **secrets** (composite actions require passing them as inputs)
- Reusable workflows preserve **job-level parallelism** and **conditional execution**

Use composite actions when you need a reusable *building block* within a job (e.g., "setup Harbor CLI"). Use reusable workflows when you need a reusable *entire pipeline*.

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
- The caller grants the four permissions below even with no build workflow: GitHub checks a
  nested job's permissions when the run starts.
- **Merge PRs by squash**, with the PR title as the commit subject and an empty body (merge
  commits stay allowed for the odd case, but a release-please repository doesn't use them):

  ```bash
  gh api -X PATCH repos/<owner>/<repo> \
    -F allow_squash_merge=true -F allow_rebase_merge=false \
    -f squash_merge_commit_title=PR_TITLE -f squash_merge_commit_message=BLANK
  ```

  Rebase-merging is turned off: it would land every branch commit, as a merge commit does.

  The PR title, plus GitHub's `(#N)`, is then the PR's one line in the release notes and
  decides the bump; [`pr-title.yml`](#pr-title-check-pr-titleyml) checks it. Why the body stays
  empty: release-please parses the whole commit message, and in a body copied from the PR a
  paragraph starting with `fix:` becomes an entry of its own (marked breaking when a
  `BREAKING CHANGE:` line follows), and a `Release-As:` line forces the version.
- Add three files, and the title check.

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

`.github/workflows/pr-title.yml`:

```yaml
name: PR title

on:
  pull_request:
    types: [ opened, edited, synchronize, reopened ]

permissions: {}

jobs:
  title:
    uses: vchatela-org/shared-workflows/.github/workflows/pr-title.yml@v1
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
  other file carrying it. Mark the line in that file with a block, so the line itself stays
  as it was (a check that parses `__version__ = "X"` keeps working):

  ```python
  # x-release-please-start-version
  __version__ = "0.9.0"
  # x-release-please-end
  ```

  Drop either key if the repository has no such file. `release-type` can also be `python` or
  `node` for a package manifest.
- **The first release PR starts at the manifest's version**: release-please looks for that
  version's GitHub Release, then for its tag (`v0.9.0`), and only collects the commits after it.
  No GitHub Release needs to exist beforehand.
- **Versions while 0.x**: a `feat` bumps the minor, any other listed type the patch, and a
  breaking change (`feat!:` or a `BREAKING CHANGE:` footer) the minor (`bump-minor-pre-major`).
  To force a version, set `"release-as": "X.Y.Z"` in the package's entry of
  `release-please-config.json`, and remove it once that release is out. (A `Release-As:` footer
  in a commit works too, but a malformed one stays in history and breaks every run until the
  next release; in the config it is one PR to fix.) Hidden types never make a release on their
  own. A revert counts only as `revert: <subject>`: `git revert`'s own `Revert "..."` is not a
  conventional commit, so reword it.
- **Dependencies**: Dependabot's commit prefix decides what reaches the notes. `prefix-development`
  only exists for some ecosystems (npm, pip, bundler, composer, maven, mix), so set it per block:

  ```yaml
    - package-ecosystem: "pip"          # also npm
      commit-message:
        prefix: "deps"                  # runtime: listed, makes a patch release
        prefix-development: "chore"     # dev: hidden
    - package-ecosystem: "docker"
      commit-message:
        prefix: "deps"                  # every FROM, build stages included
    - package-ecosystem: "github-actions"
      commit-message:
        prefix: "ci"                    # hidden
  ```

Once this is in, don't bump the version or tag by hand: merge the release PR. If a build fails
to start, re-run the failed `dispatch-builds` job; it skips a workflow already started on that tag.

## Limits

- **Why squash, not merge commits**: with a merge commit, release-please reads every commit the
  branch brings (each `fix: review findings` becomes a line of the notes), and since it walks
  history by commit date and stops at the last release, a branch commit dated before that
  release but merged after it is never seen. A squash commit is one entry, dated at the merge.
  If a merge commit is used anyway, first update the branch with a rebase
  (`gh pr update-branch --rebase <PR>`), which re-dates its commits.
- **The release PR runs no CI**: it is pushed with `GITHUB_TOKEN`, whose events start no
  workflow. It only touches the version files and `CHANGELOG.md`; the push checks run on the
  default branch once it is merged. A required status check would block it for good: that
  would need a GitHub App token for release-please instead.

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
| `release_created` | `true` when the run created a release, empty otherwise |
| `tag_name` | The new tag (e.g. `v1.4.0`) |
| `version` | The new version (e.g. `1.4.0`) |

# PR title check (`pr-title.yml`)

Fails the PR unless its title reads `type(scope)!: subject` (scope and `!` optional) with a
type from the `types` input, by default the release-please config's sections: `feat fix perf
revert deps docs refactor test build ci chore style`. A bash test, no third-party action;
`edited` in the caller's triggers runs it again when the title is fixed (a re-run of the old
run would check the old title). Dependabot titles pass (`deps: Bump …`, `chore(deps): Bump …`).

It is advisory: a red title check means "don't merge yet", and nothing enforces it. It cannot
be a required status check, because the release PR runs no workflow and would never get it.

| Input | Required | Default | Description |
|---|---|---|---|
| `types` | No | the twelve above | Allowed types, whitespace-separated |
| `runs_on` | No | `arc-runner-set` | Runner label |
