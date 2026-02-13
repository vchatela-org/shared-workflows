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
    uses: vchatela-org/shared-workflows/.github/workflows/docker-build-push-harbor.yml@main
    with:
      harbor_registry: ${{ vars.HARBOR_REGISTRY }}
      harbor_project: ${{ vars.HARBOR_PROJECT }}
      harbor_image: ${{ vars.HARBOR_IMAGE }}
    secrets:
      harbor_username: ${{ secrets.HARBOR_USERNAME }}
      harbor_password: ${{ secrets.HARBOR_PASSWORD }}
```

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
    uses: vchatela-org/shared-workflows/.github/workflows/docker-build-push-harbor.yml@main
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
    uses: vchatela-org/shared-workflows/.github/workflows/docker-build-push-harbor.yml@main
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
    uses: vchatela-org/shared-workflows/.github/workflows/docker-build-push-harbor.yml@main
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
    uses: vchatela-org/shared-workflows/.github/workflows/docker-build-push-harbor.yml@main
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
