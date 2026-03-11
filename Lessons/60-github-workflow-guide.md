# Annex A — GitHub Actions Workflow Guide

> A complete reference for GitHub Actions workflow structure, syntax, and control flow patterns.
> Designed as a companion to the CI Learning Path.

---

## Table of Contents

1. [What Is a Workflow?](#1-what-is-a-workflow)
2. [Anatomy of a Workflow File](#2-anatomy-of-a-workflow-file)
3. [Triggers (`on`)](#3-triggers-on)
4. [Runners (`runs-on`)](#4-runners-runs-on)
5. [Steps — The Unit of Work](#5-steps--the-unit-of-work)
6. [Sequential vs Parallel Jobs](#6-sequential-vs-parallel-jobs)
7. [Dependent Jobs (`needs`)](#7-dependent-jobs-needs)
8. [Conditional Flow (`if`)](#8-conditional-flow-if)
9. [Loop / Matrix Flow (`strategy.matrix`)](#9-loop--matrix-flow-strategymatrix)
10. [Expressions, Contexts, and Variables](#10-expressions-contexts-and-variables)
11. [Secrets and Environment Variables](#11-secrets-and-environment-variables)
12. [Artifacts and Data Passing Between Jobs](#12-artifacts-and-data-passing-between-jobs)
13. [Reusable Workflows and Composite Actions](#13-reusable-workflows-and-composite-actions)
14. [Permissions](#14-permissions)
15. [Concurrency Control](#15-concurrency-control)
16. [Timeouts and Failure Strategies](#16-timeouts-and-failure-strategies)
17. [Caching Dependencies](#17-caching-dependencies)
18. [Complete Reference Pipeline](#18-complete-reference-pipeline)

---

## 1. What Is a Workflow?

A **workflow** is an automated process defined in a YAML file stored under `.github/workflows/` in your repository. GitHub Actions detects and runs every `.yml` file in that directory according to its trigger conditions.

```text
Repository
└── .github/
    └── workflows/
        ├── ci.yml          ← runs on every push / PR
        ├── docker.yml      ← builds and pushes images
        ├── release.yml     ← creates releases on tag push
        └── nightly.yml     ← scheduled security scans
```

Each workflow is independent. They can run in parallel with each other. One workflow cannot directly call another — unless you use the `workflow_call` trigger (see [Reusable Workflows](#13-reusable-workflows-and-composite-actions)).

---

## 2. Anatomy of a Workflow File

A complete workflow is composed of four top-level keys:

```yaml
name: CI Pipeline          # (1) Display name shown in the GitHub UI

on: [push, pull_request]   # (2) Trigger conditions — when does this workflow run?

env:                        # (3) Workflow-level environment variables (optional)
  NODE_ENV: test

jobs:                       # (4) The work to be done — one or more named jobs
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Run tests
        run: npm test
```

**How the keys relate to each other:**

```text
Workflow
│
├── on          → defines WHEN the workflow runs
├── env         → variables available to ALL jobs
└── jobs        → defines WHAT runs
    ├── job-a   → a named job
    │   ├── runs-on  → which machine type
    │   ├── env      → variables for this job only
    │   └── steps    → ordered list of commands and actions
    └── job-b   → another named job (runs in parallel by default)
```

---

## 3. Triggers (`on`)

The `on` key controls when a workflow is activated. It is the most nuanced part of the file.

### 3.1 Simple Event Triggers

```yaml
on: push                    # Any push to any branch

on: [push, pull_request]    # Either event activates the workflow
```

### 3.2 Filtered Push Triggers — Branch Scoping

This is the most common production pattern: run the workflow only when certain branches or tags are pushed.

```yaml
on:
  push:
    branches:
      - main                # Exact name
      - 'release/**'        # Glob: any branch starting with release/
      - 'feature/**'
    tags:
      - 'v*'                # Any tag starting with v (e.g. v1.0.0, v2.3.1)
    paths:
      - 'src/**'            # Only run if files under src/ changed
      - '!docs/**'          # Exclude docs/ changes (! = negation)
```

**Branch filter logic:**

```text
Push to main              → workflow runs      (matches 'main')
Push to release/1.0       → workflow runs      (matches 'release/**')
Push to develop           → workflow skipped   (no matching rule)
Push tag v1.2.0           → workflow runs      (matches 'v*')
Push tag beta-1           → workflow skipped   (no match)
Push to main (docs only)  → workflow skipped   (paths exclude docs/**)
```

### 3.3 Pull Request Triggers

```yaml
on:
  pull_request:
    branches:
      - main                    # Only PRs targeting main
      - 'release/**'
    types:
      - opened                  # PR was opened
      - synchronize             # New commits pushed to the PR branch
      - reopened                # PR was re-opened after being closed
      # Other types: edited, labeled, unlabeled, closed, ready_for_review
```

> **Important:** `pull_request` workflows from forked repositories run with read-only permissions and no access to secrets, for security reasons. Use `pull_request_target` (with caution) if you need secret access in fork PRs.

### 3.4 Scheduled Triggers (Cron)

```yaml
on:
  schedule:
    - cron: '0 6 * * 1'         # Every Monday at 06:00 UTC
    - cron: '0 2 * * *'         # Every day at 02:00 UTC
```

Cron format: `minute hour day-of-month month day-of-week`

```text
'0 6 * * 1'
 │ │ │ │ └── day of week: 1 = Monday (0=Sun, 7=Sun)
 │ │ │ └──── month: * = every month
 │ │ └────── day of month: * = every day
 │ └──────── hour: 6 = 06:00 UTC
 └────────── minute: 0 = top of the hour
```

### 3.5 Manual Trigger

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]
      debug:
        description: 'Enable debug logging'
        type: boolean
        default: false
```

`workflow_dispatch` adds a "Run workflow" button in the GitHub UI. Inputs are available in steps as `${{ inputs.environment }}`.

### 3.6 Other Useful Triggers

```yaml
on:
  release:
    types: [published]          # When a GitHub Release is published

  workflow_run:
    workflows: ["CI"]           # When another named workflow completes
    types: [completed]

  workflow_call:                # This workflow can be called by other workflows
    inputs:
      version:
        type: string
        required: true
```

---

## 4. Runners (`runs-on`)

The runner is the virtual machine that executes the job. GitHub provides hosted runners; you can also register self-hosted runners.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest      # Recommended for most CI tasks

  mac-build:
    runs-on: macos-latest       # Required for iOS/macOS builds

  windows-build:
    runs-on: windows-latest     # Required for Windows-specific testing
```

**Available GitHub-hosted runners:**

| Label | OS | Notes |
|---|---|---|
| `ubuntu-latest` | Ubuntu 22.04 | Fastest, cheapest, most tooling pre-installed |
| `ubuntu-22.04` | Ubuntu 22.04 | Pin to a specific version |
| `ubuntu-20.04` | Ubuntu 20.04 | Legacy support |
| `macos-latest` | macOS 14 | Required for Xcode/Swift |
| `windows-latest` | Windows Server 2022 | Required for .NET/MSBuild |

**Self-hosted runners:**

```yaml
runs-on: [self-hosted, linux, x64, gpu]
# Labels allow targeting specific machine capabilities
```

---

## 5. Steps — The Unit of Work

A job is a sequence of steps executed in order on the same runner. Each step is either a **shell command** (`run`) or a **pre-built action** (`uses`).

### 5.1 Shell Commands (`run`)

```yaml
steps:
  - name: Install dependencies
    run: npm ci

  - name: Multi-line command
    run: |
      echo "Running tests..."
      npm test
      echo "Done"

  - name: Use a different shell
    shell: python
    run: |
      import sys
      print(f"Python {sys.version}")
```

### 5.2 Pre-built Actions (`uses`)

```yaml
steps:
  - name: Checkout repository
    uses: actions/checkout@v4        # Official GitHub action, version 4
    with:
      fetch-depth: 0                 # Clone full history (default: 1 shallow)
      token: ${{ secrets.PAT }}     # Use a PAT instead of GITHUB_TOKEN

  - name: Setup Node.js
    uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'                   # Built-in cache for node_modules

  - name: Custom action from same repo
    uses: ./.github/actions/my-action
```

### 5.3 Step Outputs

Steps can emit outputs that later steps within the same job can read:

```yaml
steps:
  - name: Get version
    id: version                      # ← assign an id to reference this step
    run: |
      VERSION=$(cat package.json | jq -r '.version')
      echo "tag=v${VERSION}" >> $GITHUB_OUTPUT   # write output

  - name: Print version
    run: echo "Version is ${{ steps.version.outputs.tag }}"
    #                          ↑ reference by step id
```

### 5.4 Step-Level Environment Variables

```yaml
steps:
  - name: Deploy
    env:
      API_URL: https://api.example.com
      DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
    run: ./scripts/deploy.sh
```

---

## 6. Sequential vs Parallel Jobs

### 6.1 Parallel (Default)

All jobs at the top level of `jobs:` run **in parallel** by default. GitHub spins up a separate runner VM for each job simultaneously.

```yaml
jobs:
  lint:          # ┐
    runs-on: ubuntu-latest  # │ these three start at the same time
    steps: ...              # │
                            # │
  test:          # │
    runs-on: ubuntu-latest  # │
    steps: ...              # │
                            # │
  security:      # ┘
    runs-on: ubuntu-latest
    steps: ...
```

```text
Time →

lint     ████████████
test     ████████████████████
security ████████████
```

Use parallel jobs to cut wall-clock time. Each job has an isolated, fresh environment.

### 6.2 Sequential (Using `needs`)

To force sequential execution, use `needs:` to declare dependencies. See [Section 7](#7-dependent-jobs-needs) for the full reference.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: ...

  test:
    needs: build             # test waits for build to succeed
    runs-on: ubuntu-latest
    steps: ...

  deploy:
    needs: test              # deploy waits for test to succeed
    runs-on: ubuntu-latest
    steps: ...
```

```text
Time →

build    ████████
                 test    ████████████
                                     deploy  ████
```

### 6.3 Fan-out / Fan-in Pattern

A common real-world pattern: one job triggers parallel work, then a final job waits for all of them.

```yaml
jobs:
  build:           # runs first
    ...

  test-unit:       # run in parallel after build
    needs: build
    ...

  test-integration:
    needs: build
    ...

  test-e2e:
    needs: build
    ...

  deploy:          # runs after ALL three test jobs pass
    needs: [test-unit, test-integration, test-e2e]
    ...
```

```text
Time →

build              ████████
                            test-unit        ████████
                            test-integration ████████████
                            test-e2e         ██████
                                                         deploy ████
```

---

## 7. Dependent Jobs (`needs`)

`needs` accepts either a single job name or a list of job names. The current job starts only after all listed dependencies complete successfully.

```yaml
jobs:
  a:
    runs-on: ubuntu-latest
    steps:
      - run: echo "job a"

  b:
    needs: a                       # depends on a
    runs-on: ubuntu-latest
    steps:
      - run: echo "job b"

  c:
    needs: a                       # also depends on a (parallel with b)
    runs-on: ubuntu-latest
    steps:
      - run: echo "job c"

  d:
    needs: [b, c]                  # depends on BOTH b AND c
    runs-on: ubuntu-latest
    steps:
      - run: echo "job d"
```

### Accessing Outputs Across Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}    # expose step output as job output
    steps:
      - id: meta
        run: echo "version=1.2.3" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.image-tag }}"
      #                          ↑ access via needs.<job>.outputs.<key>
```

### Running a Dependent Job Even on Failure

By default, if a dependency fails, the dependent job is skipped. Use `always()` or `failure()` conditions to override:

```yaml
  notify:
    needs: [build, test, deploy]
    if: always()                    # run regardless of upstream results
    runs-on: ubuntu-latest
    steps:
      - run: echo "Pipeline done, result = ${{ job.status }}"
```

---

## 8. Conditional Flow (`if`)

The `if` key can be applied to an entire **job** or to an individual **step**. The job/step is skipped when the expression evaluates to false.

### 8.1 Job-Level Conditions

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'       # only on the develop branch

  deploy-production:
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')    # only on version tags

  notify-failure:
    needs: [build, test]
    if: failure()                                 # only if any dependency failed
```

### 8.2 Step-Level Conditions

```yaml
steps:
  - name: Upload coverage
    if: success()                              # default — only if previous steps succeeded

  - name: Notify on failure
    if: failure()                              # only if a prior step failed

  - name: Always clean up
    if: always()                               # runs regardless

  - name: Only on main
    if: github.ref == 'refs/heads/main'

  - name: Only on PR
    if: github.event_name == 'pull_request'

  - name: Only on manual trigger with debug enabled
    if: github.event_name == 'workflow_dispatch' && inputs.debug == 'true'
```

### 8.3 Status Functions

| Function | Evaluates to true when... |
|---|---|
| `success()` | All previous steps succeeded (default) |
| `failure()` | At least one previous step failed |
| `cancelled()` | The workflow was cancelled |
| `always()` | Unconditionally true |

### 8.4 Expression Operators

```yaml
if: github.actor == 'dependabot[bot]'                        # equality
if: github.ref != 'refs/heads/main'                          # inequality
if: contains(github.event.head_commit.message, '[skip ci]')  # contains()
if: startsWith(github.ref, 'refs/tags/')                     # startsWith()
if: github.event_name == 'push' && github.ref == 'refs/heads/main'  # AND
if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'  # OR
if: !contains(github.event.pull_request.labels.*.name, 'skip-ci')  # NOT
```

---

## 9. Loop / Matrix Flow (`strategy.matrix`)

A matrix builds a **job for each combination** of values you define. It is the workflow equivalent of a for-loop. Each cell in the matrix becomes an independent parallel job with its own runner.

### 9.1 Single-Dimension Matrix

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]      # run once for each value

    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}   # access current value
      - run: npm test
```

This creates 3 parallel jobs:

- `test (18)`
- `test (20)`
- `test (22)`

### 9.2 Multi-Dimension Matrix

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    node: [18, 20]
```

This creates **3 × 2 = 6** parallel jobs, one for every combination:

```text
ubuntu-latest + node 18
ubuntu-latest + node 20
macos-latest  + node 18
macos-latest  + node 20
windows-latest + node 18
windows-latest + node 20
```

### 9.3 Matrix with `include` (Add Specific Combinations)

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    python: ['3.10', '3.12']
    include:
      - os: ubuntu-latest
        python: '3.9'          # add this specific combo
        experimental: true     # add extra variable for this combo only
```

### 9.4 Matrix with `exclude` (Remove Specific Combinations)

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    node: [18, 20]
    exclude:
      - os: windows-latest
        node: 18               # skip Node 18 on Windows
```

### 9.5 Failure Handling in Matrices

```yaml
strategy:
  fail-fast: false             # default is true — false lets all cells complete
                               # even if one fails (useful for experimental matrices)
  max-parallel: 3              # limit concurrent jobs to avoid rate limits
  matrix:
    node: [18, 20, 22]
```

With `fail-fast: true` (default), if `node 18` fails, GitHub immediately cancels all other in-progress matrix jobs. With `fail-fast: false`, all 3 complete regardless.

---

## 10. Expressions, Contexts, and Variables

### 10.1 Expression Syntax

Expressions are wrapped in `${{ }}` and can appear in most YAML values:

```yaml
run: echo "Branch is ${{ github.ref_name }}"
if: github.actor != 'dependabot[bot]'
with:
  node-version: ${{ matrix.node }}
```

### 10.2 Key Contexts

**`github` context** — information about the event that triggered the workflow:

```yaml
${{ github.repository }}       # "org/repo-name"
${{ github.ref }}              # "refs/heads/main" or "refs/tags/v1.0.0"
${{ github.ref_name }}         # "main" or "v1.0.0" (short name)
${{ github.sha }}              # Full commit SHA
${{ github.actor }}            # Username who triggered the event
${{ github.event_name }}       # "push", "pull_request", "workflow_dispatch"
${{ github.run_id }}           # Unique ID for this workflow run
${{ github.run_number }}       # Auto-incrementing run count for this workflow
${{ github.server_url }}       # "https://github.com"
${{ github.workspace }}        # Path to checked-out repo on runner
```

**`job` context** — the current job's status:

```yaml
${{ job.status }}              # "success", "failure", "cancelled"
```

**`steps` context** — outputs from earlier steps in the same job:

```yaml
${{ steps.<step-id>.outputs.<output-name> }}
${{ steps.<step-id>.outcome }}     # "success", "failure", "skipped", "cancelled"
${{ steps.<step-id>.conclusion }}  # final status after continue-on-error
```

**`needs` context** — outputs from upstream jobs:

```yaml
${{ needs.<job-id>.outputs.<output-name> }}
${{ needs.<job-id>.result }}       # "success", "failure", "skipped", "cancelled"
```

**`matrix` context** — current matrix cell values:

```yaml
${{ matrix.node-version }}
${{ matrix.os }}
```

**`secrets` context** — encrypted secrets:

```yaml
${{ secrets.MY_TOKEN }}
${{ secrets.GITHUB_TOKEN }}        # auto-provided, no setup needed
```

**`inputs` context** — `workflow_dispatch` or `workflow_call` inputs:

```yaml
${{ inputs.environment }}
${{ inputs.debug }}
```

### 10.3 Setting Dynamic Variables Mid-Job

```yaml
- name: Set computed variable
  run: |
    echo "DEPLOY_ENV=production" >> $GITHUB_ENV
    echo "BUILD_TIME=$(date -u +%Y%m%d%H%M%S)" >> $GITHUB_ENV

- name: Use the variable
  run: echo "Deploying to $DEPLOY_ENV at $BUILD_TIME"
  # Variables set via GITHUB_ENV are available to all subsequent steps in the job
```

---

## 11. Secrets and Environment Variables

### 11.1 Secrets

Secrets are encrypted values stored in repository or organization settings. They are never printed in logs.

```yaml
steps:
  - name: Deploy
    env:
      API_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    run: ./deploy.sh
```

Secrets can be scoped at three levels:

```text
Organization secrets  →  available to all repos in the org
Repository secrets    →  available to this repo only
Environment secrets   →  available only when deploying to a specific environment
```

**`GITHUB_TOKEN`** is a special secret automatically generated for every workflow run. It authenticates as the repository's GitHub App and expires when the workflow ends. No setup required.

```yaml
- name: Comment on PR
  uses: actions/github-script@v7
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    script: |
      github.rest.issues.createComment({ ... })
```

### 11.2 Variable Scoping

```yaml
env:
  WORKFLOW_VAR: "visible to all jobs"     # workflow-level

jobs:
  build:
    env:
      JOB_VAR: "visible to all steps in this job"   # job-level

    steps:
      - name: Deploy
        env:
          STEP_VAR: "visible only to this step"      # step-level
        run: echo "$WORKFLOW_VAR $JOB_VAR $STEP_VAR"
```

### 11.3 Repository Variables (non-secret)

For non-sensitive configuration values, use GitHub repository variables (Settings → Secrets and variables → Variables):

```yaml
run: echo "Deploying to ${{ vars.DEPLOY_HOST }}"
```

---

## 12. Artifacts and Data Passing Between Jobs

Each job runs on a separate, isolated VM. To share files between jobs, upload them as artifacts from one job and download them in another.

### 12.1 Upload an Artifact

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build        # produces ./dist/

      - name: Upload build output
        uses: actions/upload-artifact@v4
        with:
          name: build-output      # artifact name (used to download later)
          path: dist/             # what to upload
          retention-days: 1       # how long to keep it (default: 90)
```

### 12.2 Download an Artifact

```yaml
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build output
        uses: actions/download-artifact@v4
        with:
          name: build-output      # must match the upload name
          path: dist/             # where to put it on this runner

      - run: ls dist/             # files from the build job are now here
```

### 12.3 Upload Test Reports

```yaml
      - name: Upload test results
        if: always()              # upload even if tests fail
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: |
            coverage/
            junit.xml
```

---

## 13. Reusable Workflows and Composite Actions

### 13.1 Reusable Workflows

A workflow triggered by `workflow_call` can be invoked from another workflow. This is the primary way to avoid duplicating entire pipeline definitions.

**Defining the reusable workflow** (`.github/workflows/deploy-shared.yml`):

```yaml
on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true
    secrets:
      deploy-token:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to ${{ inputs.environment }}"
        env:
          TOKEN: ${{ secrets.deploy-token }}
```

**Calling the reusable workflow:**

```yaml
jobs:
  deploy-staging:
    uses: ./.github/workflows/deploy-shared.yml
    with:
      environment: staging
    secrets:
      deploy-token: ${{ secrets.STAGING_TOKEN }}

  deploy-production:
    needs: deploy-staging
    uses: ./.github/workflows/deploy-shared.yml
    with:
      environment: production
    secrets:
      deploy-token: ${{ secrets.PROD_TOKEN }}
```

### 13.2 Composite Actions

A composite action bundles multiple steps into a single `uses:` reference. Unlike reusable workflows, composite actions run within the calling job (no separate VM).

**Define it** (`.github/actions/setup-project/action.yml`):

```yaml
name: Setup Project
description: Install Node.js and cache dependencies

inputs:
  node-version:
    description: Node.js version
    default: '20'

runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm
    - run: npm ci
      shell: bash
```

**Use it in a workflow:**

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-project
    with:
      node-version: '20'
  - run: npm test
```

---

## 14. Permissions

By default, each workflow run is granted `GITHUB_TOKEN` with permissions scoped to the repository. You should always restrict permissions to the minimum required.

```yaml
permissions:
  contents: read          # read files from the repo
  packages: write         # push to GHCR
  pull-requests: write    # comment on PRs
  issues: write           # create/modify issues
  security-events: write  # upload SARIF results to Security tab
  id-token: write         # request OIDC token (for keyless cloud auth)
  actions: read           # read workflow run info
```

Permissions can be set at workflow level (applies to all jobs) or per-job:

```yaml
jobs:
  scan:
    permissions:
      security-events: write    # only this job gets this permission
      contents: read
    steps: ...
```

**To disable all permissions** (run with zero access):

```yaml
permissions: {}
```

---

## 15. Concurrency Control

By default, multiple runs of the same workflow can execute simultaneously. Use `concurrency` to limit this.

```yaml
# Cancel any in-progress run for the same branch when a new push arrives
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

```yaml
# Queue deployments — don't cancel, let each finish
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false
```

```yaml
# Per-job concurrency (protects a shared resource)
jobs:
  deploy:
    concurrency:
      group: production-deploy
      cancel-in-progress: false
```

---

## 16. Timeouts and Failure Strategies

### 16.1 Timeout

```yaml
jobs:
  build:
    timeout-minutes: 30          # job-level timeout (default: 360 min / 6 hours)
    steps:
      - name: Long operation
        timeout-minutes: 10      # step-level timeout
        run: ./long-script.sh
```

### 16.2 Continue on Error

```yaml
steps:
  - name: Optional lint check
    continue-on-error: true      # step failure won't fail the job
    run: npm run lint

  - name: This always runs
    run: echo "Even if lint failed"
```

At job level, use `if: always()` rather than `continue-on-error` to run cleanup steps unconditionally.

### 16.3 Retry Logic

GitHub Actions does not have built-in retry, but a common workaround:

```yaml
- name: Flaky API call with retry
  uses: nick-fields/retry@v3
  with:
    timeout_minutes: 5
    max_attempts: 3
    command: ./scripts/call-api.sh
```

---

## 17. Caching Dependencies

Caching re-uses files from previous runs, dramatically cutting install times.

### 17.1 Built-in Cache via `setup-*` Actions

The easiest approach — `actions/setup-node`, `setup-python`, `setup-java` etc. have a built-in `cache` option:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'         # caches ~/.npm based on package-lock.json hash
```

### 17.2 Manual Cache with `actions/cache`

```yaml
- name: Cache node_modules
  uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    #     └── vary by OS   └── vary by lock file contents
    restore-keys: |
      ${{ runner.os }}-node-    # fallback: any cache for this OS
```

**Cache key strategy:**

```text
Exact hit:   ubuntu-node-abc123  →  restore fully, skip install
Partial hit: ubuntu-node-        →  restore older cache, npm ci updates delta
Miss:        (nothing)           →  full npm ci, save new cache at end
```

### 17.3 Docker Layer Cache

```yaml
- uses: docker/setup-buildx-action@v3

- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    cache-from: type=gha          # read from GitHub Actions cache
    cache-to: type=gha,mode=max   # write layers to GitHub Actions cache
```

---

## 18. Complete Reference Pipeline

This pipeline demonstrates all major concepts together: triggers, parallel jobs, dependencies, matrix, conditionals, artifacts, and notifications.

```yaml
# .github/workflows/ci-complete.yml
name: Complete CI Pipeline

on:
  push:
    branches: [main, develop]
    tags: ['v*']
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      skip-tests:
        type: boolean
        default: false

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_ENV: test
  REGISTRY: ghcr.io

jobs:
  # ── 1. LINT ─────────────────────────────────────────────────────────────
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm
      - run: npm ci
      - run: npm run lint

  # ── 2. TEST (matrix over Node versions) ─────────────────────────────────
  test:
    runs-on: ubuntu-latest
    if: ${{ !inputs.skip-tests }}
    strategy:
      fail-fast: false
      matrix:
        node: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm
      - run: npm ci
      - run: npm test -- --coverage
      - name: Upload coverage
        if: matrix.node == 20        # only upload once
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  # ── 3. BUILD IMAGE ───────────────────────────────────────────────────────
  docker:
    needs: [lint, test]              # wait for lint AND all test matrix cells
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix=sha-
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── 4. DEPLOY STAGING ────────────────────────────────────────────────────
  deploy-staging:
    needs: docker
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
      - run: echo "Deploying ${{ needs.docker.outputs.image-tag }} to staging"

  # ── 5. DEPLOY PRODUCTION ─────────────────────────────────────────────────
  deploy-production:
    needs: docker
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    environment: production
    steps:
      - run: echo "Deploying ${{ needs.docker.outputs.image-tag }} to production"

  # ── 6. NOTIFY ─────────────────────────────────────────────────────────────
  notify:
    needs: [lint, test, docker]
    if: always() && github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "❌ CI failed on `${{ github.ref_name }}` by ${{ github.actor }}\n${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

---

## Quick Reference Card

### Trigger → Branch Mapping

| Goal | Trigger config |
|---|---|
| Run on every push | `on: push` |
| Run only on main | `on: push: branches: [main]` |
| Run on feature branches | `on: push: branches: ['feature/**']` |
| Run on version tags | `on: push: tags: ['v*']` |
| Run on PRs to main | `on: pull_request: branches: [main]` |
| Run on schedule | `on: schedule: - cron: '0 6 * * 1'` |
| Run manually | `on: workflow_dispatch` |

### Job Execution Patterns

| Pattern | Syntax |
|---|---|
| Parallel (default) | Define multiple jobs with no `needs` |
| Sequential A → B | `needs: a` on job B |
| Fan-out A → B, C, D | `needs: a` on B, C, and D each |
| Fan-in B, C, D → E | `needs: [b, c, d]` on job E |
| Loop over values | `strategy: matrix: values: [...]` |
| Skip on failure | `if: success()` (default) |
| Run only on failure | `if: failure()` |
| Always run | `if: always()` |

### Useful Expressions Cheatsheet

```yaml
github.ref == 'refs/heads/main'
startsWith(github.ref, 'refs/tags/v')
contains(github.event.head_commit.message, '[skip ci]')
github.event_name == 'pull_request'
github.actor == 'dependabot[bot]'
matrix.os == 'ubuntu-latest'
needs.build.result == 'success'
inputs.environment == 'production'
```

---

*This guide is part of the CI Learning Path. See the [main index](index.md) for the full table of contents.*
