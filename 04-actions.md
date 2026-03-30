# GitHub Actions

> YAML-driven CI/CD and automation platform. Workflows live in `.github/workflows/` and run on events, schedules, or manual triggers.

---

## Core Concepts

```
Workflow
└── Triggered by Event(s)
    └── Contains one or more Jobs
        └── Run on a Runner
            └── Contains sequential Steps
                └── Each step: shell command OR Action
```

| Concept | Description |
|---|---|
| **Workflow** | `.yml` file in `.github/workflows/`; defines full automation |
| **Event** | What triggers the workflow (push, PR, schedule, manual…) |
| **Job** | Unit of work; runs on one runner; jobs can run in parallel or sequentially |
| **Step** | Single command or action within a job |
| **Action** | Reusable packaged step (from Marketplace or local) |
| **Runner** | Compute that executes jobs (GitHub-hosted or self-hosted) |
| **Context** | Runtime data available via `${{ }}` expressions |
| **Secret** | Encrypted variable; available to steps as environment variables |
| **Environment** | Named deployment target with its own secrets + protection rules |
| **Artifact** | Files uploaded/downloaded between jobs or stored post-run |
| **Cache** | Store dependencies to speed up subsequent runs |

---

## Workflow File Structure

```yaml
name: CI Pipeline                    # Display name in UI

on:                                  # Trigger(s)
  push:
    branches: [main, develop]
    paths-ignore: ['**.md']
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'             # Every Monday 2am UTC
  workflow_dispatch:                 # Manual trigger
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

env:                                 # Workflow-level environment variables
  NODE_VERSION: '20'
  REGISTRY: ghcr.io

permissions:                         # GITHUB_TOKEN permissions (least privilege)
  contents: read
  pull-requests: write

jobs:
  build:
    name: Build & Test
    runs-on: ubuntu-latest           # Runner
    timeout-minutes: 15              # Prevent runaway jobs

    strategy:                        # Matrix builds
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, windows-latest]
      fail-fast: false               # Don't cancel other matrix jobs on failure

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0             # Full history (0 = all)

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
        env:
          CI: true

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        if: always()                 # Run even if tests fail
        with:
          name: coverage-${{ matrix.node }}
          path: coverage/
          retention-days: 14

  deploy:
    name: Deploy
    needs: build                     # Depends on build job
    runs-on: ubuntu-latest
    environment:                     # Deployment environment (with protection rules)
      name: production
      url: https://myapp.example.com
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: ./deploy.sh
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
```

---

## Trigger Events Reference

### Push / Pull Request
```yaml
on:
  push:
    branches: [main, 'release/**']
    tags: ['v*.*.*']
    paths: ['src/**', 'package.json']
    paths-ignore: ['**.md', 'docs/**']
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches: [main]
  pull_request_target:               # Runs in base repo context (safe for forks)
    types: [opened]
```

### Scheduled
```yaml
on:
  schedule:
    - cron: '30 5 * * 1,3'         # Mon/Wed at 05:30 UTC
```

### Manual & API
```yaml
on:
  workflow_dispatch:                 # Button in GitHub UI
    inputs:
      logLevel:
        type: choice
        options: [info, debug, warning]
  repository_dispatch:               # POST /repos/{owner}/{repo}/dispatches
    types: [deploy-event, custom-event]
```

### Workflow Composition
```yaml
on:
  workflow_call:                     # Called from another workflow
    inputs:
      environment: { type: string, required: true }
    secrets:
      deploy-key: { required: true }
  workflow_run:                      # After another workflow completes
    workflows: ['CI']
    types: [completed]
    branches: [main]
```

### Issue / PR Events
```yaml
on:
  issues:
    types: [opened, labeled, closed]
  issue_comment:
    types: [created]
  pull_request_review:
    types: [submitted]
  release:
    types: [published]
```

---

## Runners

### GitHub-Hosted Runners

| Label | OS | Specs |
|---|---|---|
| `ubuntu-latest` | Ubuntu 24.04 | 4 vCPU, 16 GB RAM, 14 GB SSD |
| `ubuntu-22.04` | Ubuntu 22.04 | 4 vCPU, 16 GB RAM |
| `ubuntu-20.04` | Ubuntu 20.04 | 4 vCPU, 16 GB RAM |
| `windows-latest` | Windows Server 2022 | 4 vCPU, 16 GB RAM |
| `windows-2022` | Windows Server 2022 | 4 vCPU, 16 GB RAM |
| `macos-latest` | macOS 14 (ARM) | 3 vCPU, 7 GB RAM |
| `macos-13` | macOS 13 (Intel) | 4 vCPU, 14 GB RAM |

**Larger runners** (paid plans): up to 64 vCPU, GPU runners, custom images.

### Self-Hosted Runners
```yaml
runs-on: [self-hosted, linux, x64, gpu]   # Labels assigned at registration
```
- Persistent; your infrastructure, your cost
- Access to private networks
- No usage minutes consumed
- Security: do not use on public repos (can execute arbitrary code from PRs)

---

## Secrets & Variables

### Scopes
| Scope | Set in | Available to |
|---|---|---|
| **Repository secret** | Repo Settings → Secrets | That repo only |
| **Repository variable** | Repo Settings → Variables | That repo only |
| **Environment secret** | Repo Settings → Environments | Jobs targeting that environment |
| **Organisation secret** | Org Settings → Secrets | All/selected repos in org |
| **Organisation variable** | Org Settings → Variables | All/selected repos in org |

### Using Secrets
```yaml
steps:
  - run: echo "Deploying..."
    env:
      API_KEY: ${{ secrets.API_KEY }}          # secret
      APP_ENV: ${{ vars.APP_ENVIRONMENT }}     # variable (not secret)
```

### GITHUB_TOKEN
- Auto-provided in every workflow run — no setup needed
- Scoped to the repository where the workflow runs
- Expires when job completes
- Default permissions configurable at repo/org level
- **Set least-privilege at workflow level**:
```yaml
permissions:
  contents: read       # Read repo contents
  issues: write        # Comment on issues
  pull-requests: write # Comment on PRs
  packages: write      # Push to GHCR
  id-token: write      # OIDC for cloud auth
```

**GITHUB_TOKEN CANNOT**:
- Push to protected branches (unless branch protection includes bypass for Actions)
- Create other workflow runs by default (use `workflow_dispatch` via PAT/App token)
- Access other repositories

---

## Environments

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://myapp.com    # URL shown in deployment UI
```

**Environment features**:
- **Protection rules**: require reviewers before job starts, wait timer
- **Deployment branches**: only allow deployment from specific branches/tags
- **Secrets**: environment-scoped secrets (override org/repo secrets of same name)
- **Variables**: environment-scoped variables

---

## Contexts

| Context | Contains |
|---|---|
| `github` | Event data, repo name, ref, SHA, actor, workflow |
| `env` | Env vars set in workflow/job/step |
| `vars` | Repository/org variables |
| `secrets` | Secret values |
| `steps` | Outputs from previous steps |
| `jobs` | Status/outputs of other jobs |
| `runner` | Runner OS, temp dir, tool cache |
| `needs` | Outputs from dependency jobs |
| `inputs` | workflow_dispatch / workflow_call inputs |
| `matrix` | Current matrix combination values |

```yaml
# Common context examples
${{ github.repository }}          # "owner/repo"
${{ github.ref }}                 # "refs/heads/main"
${{ github.sha }}                 # Full commit SHA
${{ github.event.pull_request.number }}
${{ github.actor }}               # Username who triggered
${{ runner.os }}                  # "Linux" | "Windows" | "macOS"
${{ steps.my-step.outputs.result }}
${{ needs.build.outputs.version }}
```

---

## Caching

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# Built-in caching in setup actions:
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'          # Equivalent to manual cache action
```

Cache limit: 10 GB per repo. Evicted after 7 days of no access or when repo exceeds limit.

---

## Artifacts

```yaml
# Upload
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
    retention-days: 30       # 90 max; 90 default

# Download in another job
- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: ./downloaded-dist

# Download across workflow runs (v4 only)
- uses: actions/download-artifact@v4
  with:
    run-id: ${{ github.event.workflow_run.id }}
    name: build-output
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

---

## Job Outputs

```yaml
jobs:
  build:
    outputs:
      version: ${{ steps.version.outputs.tag }}
    steps:
      - id: version
        run: echo "tag=1.2.3" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.version }}"
```

---

## Reusable Workflows

**Caller**:
```yaml
jobs:
  call-deploy:
    uses: ./.github/workflows/deploy.yml
    with:
      environment: production
    secrets:
      deploy-key: ${{ secrets.PROD_DEPLOY_KEY }}
```

**Callee** (`deploy.yml`):
```yaml
on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true
    secrets:
      deploy-key:
        required: true
```

---

## Composite Actions

Create reusable local actions in `.github/actions/my-action/action.yml`:
```yaml
name: My Composite Action
description: Does something useful
inputs:
  token:
    required: true
runs:
  using: composite
  steps:
    - run: echo "Hello from composite"
      shell: bash
    - uses: actions/checkout@v4
```

Use in workflow:
```yaml
- uses: ./.github/actions/my-action
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
```

---

## Concurrency Control

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true    # Cancel older runs on same branch
```

---

## Common Patterns

### Release Please (automated releases)
```yaml
- uses: googleapis/release-please-action@v4
  with:
    release-type: node
```

### Docker build & push to GHCR
```yaml
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- uses: docker/build-push-action@v5
  with:
    push: true
    tags: ghcr.io/${{ github.repository }}:latest
```

### Terraform Plan + Apply
```yaml
- uses: hashicorp/setup-terraform@v3
- run: terraform init
- run: terraform plan -out=tfplan
- run: terraform apply tfplan
  if: github.ref == 'refs/heads/main'
```

---

## Limits

| Resource | Limit |
|---|---|
| Workflow run duration | 35 days |
| Job duration | 6 hours (35 days self-hosted) |
| Workflow queue time | 24 hours |
| API calls per workflow | 1,000/hour |
| Concurrent jobs (free) | 20 |
| Workflow file size | 512 KB |
| Matrix combinations | 256 per job |
| Environment variables | 10 KB total per job |
| Step summary | 1 MB |
| Artifact upload | 10 GB per run (2 GB per file) |
| Cache per repo | 10 GB |
