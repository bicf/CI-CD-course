# Continuous Integration — Complete Practical Guide

> A hands-on, sample-driven companion to the CI Learning Path.  
> Every concept is paired with working code, real configurations, and actionable explanations.

---

## Table of Contents

1. [Level 1 — Foundations](#level-1--foundations)
2. [Level 2 — Core CI Skills](#level-2--core-ci-skills)
3. [Level 3 — Intermediate CI](#level-3--intermediate-ci)
4. [Level 4 — Advanced CI](#level-4--advanced-ci)
5. [Level 5 — Mastery](#level-5--mastery)
6. [Reference: Full Pipeline Examples](#reference-full-pipeline-examples)

---

# Level 1 — Foundations

## 1.1 The SDLC and Where CI Fits

### The Traditional Problem

Before CI, teams worked in isolation for days or weeks, then attempted to merge everything at once — a painful process known as **"integration hell"**.

```
Developer A (2 weeks of work)  ──┐
Developer B (2 weeks of work)  ──┼──► MERGE ──► 💥 Conflicts everywhere
Developer C (2 weeks of work)  ──┘
```

### The CI Solution

Integrate continuously — every change triggers an automated pipeline:

```
Developer A (1 commit)  ──► Pipeline ──► ✅ Merged
Developer B (1 commit)  ──► Pipeline ──► ✅ Merged
Developer C (1 commit)  ──► Pipeline ──► ❌ Tests fail → Fix immediately
```

### The SDLC with CI

```
┌─────────┐   ┌──────┐   ┌───────┐   ┌──────┐   ┌─────────┐   ┌────────┐
│  PLAN   │──►│ CODE │──►│ BUILD │──►│ TEST │──►│ RELEASE │──►│ DEPLOY │
└─────────┘   └──────┘   └───────┘   └──────┘   └─────────┘   └────────┘
                 │              ▲
                 └───── CI ─────┘  (automated feedback on every commit)
```

| Stage | Without CI | With CI |
|-------|-----------|---------|
| Code | Manual reviews | Automated linting, type checking |
| Build | "Works on my machine" | Reproducible builds in clean environments |
| Test | Run manually, often skipped | Mandatory, automated on every commit |
| Release | Manual, error-prone | Automated versioning and changelogs |

---

## 1.2 Git Mastery for CI

### Essential Git Configuration

```bash
# Identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Default branch
git config --global init.defaultBranch main

# Useful aliases
git config --global alias.st "status --short"
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.recent "branch --sort=-committerdate"
```

### Branching Strategies Compared

#### Git Flow — scheduled releases

```
main ──────────────────────────────────────────────► (production tags)
  └── develop ─────────────────────────────────────► (integration)
        ├── feature/login ──────────► merge to develop
        ├── feature/checkout ───────► merge to develop
        └── release/1.2.0 ──────────► merge to main + develop
              └── hotfix/critical ──► merge to main + develop
```

#### GitHub Flow — continuous web delivery

```
main ─────────────────────────────────────────────► (always deployable)
  ├── feature/new-api ──► PR ──► merge ──► deploy
  └── fix/auth-bug ─────► PR ──► merge ──► deploy
```

#### Trunk-Based Development — recommended for CI

The model used by Google, Meta, and most high-performing engineering teams. Everyone integrates into `main` continuously, keeping the trunk always in a deployable state.

```
          Day 1                  Day 2                  Day 3
            │                     │                     │
main ───────┼─────────────────────┼─────────────────────┼──────► (always green)
            │  ╭─ feat/login ─╮   │  ╭─ fix/auth ─╮     │
            │  │  (< 4 hours) │   │  │  (2 hours) │     │
            │  ╰──────────────╯   │  ╰────────────╯     │
            │        │ PR+merge   │        │ PR+merge    │
            ▼        ▼            ▼        ▼             ▼
          commit   merge        commit   merge         commit
            │                             │
            └── CI runs ✅                └── CI runs ✅
```

**The core rules:**

1. **Branches live less than a day.** If a branch lasts more than 24 hours, it's a signal that the task needs to be split into smaller pieces — not that the branch should keep growing.

2. **Main is always deployable.** Every commit merged to `main` must pass CI. The pipeline is the gatekeeper — there is no "integration branch" to hide broken code.

3. **Unfinished features ship behind feature flags.** Code can be merged before the feature is complete, as long as the flag keeps it invisible to users. This separates *deployment* (code goes to production) from *release* (users can see it).

**Why it works better for CI than Git Flow or GitHub Flow:**

| | Git Flow | GitHub Flow | Trunk-Based Dev |
|---|----------|-------------|-----------------|
| Branch lifetime | Days–weeks | Hours–days | Hours (< 1 day) |
| Merge conflicts | Large, painful | Moderate | Minimal |
| CI feedback speed | Delayed | Fast | Immediate |
| Integration risk | High at release | Low | Very low |
| Feature flag needed | No | Rarely | Yes |

**What a typical TBD day looks like:**

```
09:00  Developer picks up a ticket
09:15  git checkout -b feat/add-email-validation
       (writes code + tests)
11:30  git push → opens PR
11:45  CI passes ✅, teammate reviews
12:00  Merged to main → CI runs again ✅ → auto-deployed to staging
       (feature is behind a flag: email_validation_v2 = false)

14:00  Developer picks up next ticket
       ...same cycle repeats
```

**Feature flags in practice:**

The flag lets you merge incomplete or experimental code safely. The feature is invisible in production until the team is confident enough to flip the switch — independently of any deployment.

```typescript
// The feature ships in the codebase, but is off by default
const useNewEmailValidation = featureFlags.get('email_validation_v2', false);

if (useNewEmailValidation) {
  return validateWithZod(email);      // New path — dark until flag is on
} else {
  return legacyValidate(email);       // Old path — still running
}
```

Flag lifecycle:
```
Merge (flag off) ──► QA testing (flag on for testers) ──► Gradual rollout ──► 100% ──► Remove flag
      day 1                  day 2–3                          day 4–5           day 6      day 7+
```

**When NOT to use TBD:**

- Open source projects with external contributors who submit infrequent large PRs
- Teams without a fast CI pipeline (if CI takes 30+ minutes, short-lived branches become impractical)
- Projects with formal release gating (regulated industries, firmware) — Git Flow is more appropriate

### Conventional Commits

A machine-readable commit format that enables automated changelogs and semantic versioning.

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

| Type | Meaning | Version Bump |
|------|---------|-------------|
| `feat` | New feature | Minor (1.x.0) |
| `fix` | Bug fix | Patch (1.0.x) |
| `feat!` | Breaking change | Major (x.0.0) |
| `docs` | Documentation only | None |
| `chore` | Maintenance tasks | None |
| `test` | Adding or fixing tests | None |
| `ci` | CI config changes | None |
| `perf` | Performance improvement | Patch |
| `refactor` | Code restructure | None |

**Examples:**

```bash
git commit -m "feat(auth): add OAuth2 login with Google"
git commit -m "fix(api): handle null response from upstream service"
git commit -m "feat!: remove deprecated /v1 endpoints

BREAKING CHANGE: /v1/users and /v1/orders removed. Use /v2 equivalents."
git commit -m "chore(deps): bump axios from 1.3.0 to 1.6.0"
```

### .gitignore for CI Workflows

```gitignore
# Dependencies
node_modules/
vendor/
__pycache__/
*.pyc
.venv/

# Build outputs
dist/
build/
*.egg-info/
target/

# Secrets — NEVER commit these
.env
.env.local
.env.*.local
*.pem
*.key
secrets.yaml

# CI artifacts
coverage/
.nyc_output/
junit.xml
test-results/
*.log

# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store
Thumbs.db
```

### Branch Protection Rules

Branch protection rules are the enforcement layer that makes CI meaningful. Without them, a developer can merge a PR even if every check is failing — the pipeline becomes advisory, not mandatory. Rules are configured per branch, typically targeting `main` (and sometimes `develop` or `release/*`).

**What happens without protection vs with it:**

```
Without rules:                        With rules:
┌─────────────────────────────┐        ┌─────────────────────────────┐
│  CI: ❌ Tests failing       │        │  CI: ❌ Tests failing       │
│  CI: ❌ Lint failing        │        │  CI: ❌ Lint failing        │
│                             │        │                             │
│  [ Merge ] ← still clickable│        │  [ Merge ] ← BLOCKED 🔒    │
└─────────────────────────────┘        └─────────────────────────────┘
       Developer can ignore CI                CI is a hard gate
```

#### Setting Up on GitHub (UI path)

```
Repository → Settings → Branches → Add branch protection rule
  Pattern: main   (supports wildcards: release/*, v*.*)
```

The key options and what each one actually does:

```
☑  Require a pull request before merging
   └─ Prevents direct pushes to main — all changes must come through a PR.
      Nobody, including repo admins, can bypass this without explicitly
      overriding the rule.

   ☑  Require approvals: 1
   │   └─ At least one reviewer must explicitly approve before merging.
   │      Prevents rubber-stamp self-merges on solo projects or
   │      "I'll review it after" patterns.
   │
   └─ ☑  Dismiss stale pull request approvals when new commits are pushed
         └─ If someone approves a PR and the author then pushes new
            commits, the approval is revoked. Forces re-review of changes
            made after approval — common attack vector if left unchecked.

☑  Require status checks to pass before merging
   │   └─ The actual CI gate. Lists which jobs MUST be green.
   │      The job names here must exactly match the `name:` field in your
   │      workflow (GitHub Actions) or the job key (GitLab CI).
   │
   ├─ "CI / build-and-test"    ← matches `name: build-and-test` in workflow
   ├─ "CI / lint"
   └─ "CI / security-scan"

   ☑  Require branches to be up to date before merging
         └─ The PR's branch must include all commits currently on main.
            Without this, two PRs could both pass CI individually but
            conflict when merged together ("works in isolation" problem).
            Forces the author to rebase/merge main before merging their PR.

☑  Require conversation resolution before merging
   └─ All review comments must be marked as resolved. Prevents merging
      with open questions or unaddressed feedback.

☑  Do not allow bypassing the above settings
   └─ Applies rules even to repository administrators and owners.
      Without this, admins can force-push or merge bypassing all checks.
      Essential for compliance and auditability.
```

#### How Status Check Names Are Determined

The name you register in the branch protection rule must match **exactly** what the CI system reports. Here is how to find the correct name for each platform:

**GitHub Actions** — the check name is `{workflow name} / {job id}`:
```yaml
# .github/workflows/ci.yml
name: CI                    # ← workflow name

jobs:
  build-and-test:           # ← job id
    ...
  lint:
    ...

# Results in checks named:
#   CI / build-and-test
#   CI / lint
```

**GitLab CI** — the check name is the job key:
```yaml
# .gitlab-ci.yml
unit-tests:         # ← this is the status check name reported to GitHub
  script: npm test

lint:               # ← and this one
  script: npm run lint
```

**What happens when a check is not yet run** (e.g., no CI file exists yet):

```
Status: Pending — "Expected — Waiting for status to be reported"
Result: PR is blocked until the check runs and passes.

This is the correct and safe behavior. A branch protection rule that
references a check that never runs will block merges indefinitely —
which is better than silently allowing unverified merges.
```

#### Full Recommended Configuration

```yaml
# GitHub: Settings → Branches → Add rule
# Pattern: main

# --- Pull Request Requirements ---
require_pull_request: true
required_approving_review_count: 1     # Increase to 2 for critical repos
dismiss_stale_reviews: true            # Re-review required after new commits
require_review_from_codeowners: true   # CODEOWNERS file defines who reviews what

# --- CI Status Checks (must match exact job names) ---
require_status_checks: true
require_branches_up_to_date: true      # Branch must be current with main
required_checks:
  - "CI / lint"
  - "CI / typecheck"
  - "CI / unit-tests"
  - "CI / integration-tests"
  - "CI / security-scan"

# --- Commit Requirements ---
require_signed_commits: true           # GPG-signed commits (audit trail)
require_linear_history: true           # Disables merge commits, enforces rebase
                                       # Keeps git log clean and bisectable

# --- Push Restrictions ---
restrict_pushes: true                  # Only allow pushes via PRs
allow_force_pushes: false              # Prevents history rewriting on main
allow_deletions: false                 # Prevents accidental branch deletion

# --- Admin Override ---
bypass_pull_request_allowances: []     # Nobody bypasses — not even admins
```

#### CODEOWNERS — Automatic Review Assignment

Pair branch protection with a `CODEOWNERS` file to automatically assign reviewers based on which files changed:

```bash
# .github/CODEOWNERS  (or CODEOWNERS at repo root)

# Default owner for everything
*                           @org/backend-team

# Frontend — assigned to frontend team automatically
/frontend/                  @org/frontend-team
*.tsx                       @org/frontend-team
*.css                       @org/frontend-team

# Infrastructure changes require DevOps review
/infrastructure/            @org/devops-team
Dockerfile                  @org/devops-team
docker-compose*.yml         @org/devops-team

# CI config requires a senior engineer
.github/workflows/          @org/senior-engineers
.gitlab-ci.yml              @org/senior-engineers

# Secrets and sensitive config require security team
*secret*                    @org/security-team
*credential*                @org/security-team
```

When a PR touches `/infrastructure/Dockerfile`, GitHub will automatically request a review from `@org/devops-team` and block merging until they approve.

#### GitLab Equivalent — Protected Branches

```yaml
# GitLab: Settings → Repository → Protected branches

Branch: main
Allowed to merge: Developers + Maintainers
Allowed to push: No one          # Forces all changes through MRs
Allowed to force push: false

# Settings → Merge requests
Pipelines must succeed: true      # CI gate
All discussions must be resolved: true
```

---

## 1.3 Core CI Concepts

### CI vs Continuous Delivery vs Continuous Deployment

```
──────────────────────────────────────────────────────────────
  CONTINUOUS INTEGRATION
  [Code] ──► [Build] ──► [Test]

──────────────────────────────────────────────────────────────
  + CONTINUOUS DELIVERY (human approves deploy)
  [Code] ──► [Build] ──► [Test] ──► [Staging] ──► [👤 Approve] ──► [Prod]

──────────────────────────────────────────────────────────────
  + CONTINUOUS DEPLOYMENT (fully automated)
  [Code] ──► [Build] ──► [Test] ──► [Staging] ──► [Auto ✅] ──► [Prod]
──────────────────────────────────────────────────────────────
```

### The 8 Principles of CI

1. **Single source repository** — one canonical repo
2. **Automate the build** — `make build` or equivalent, no manual steps
3. **Self-testing build** — failing tests = failing build
4. **Commit to mainline daily** — small, frequent integration reduces risk
5. **Every commit triggers CI** — the server watches every push
6. **Fix broken builds immediately** — broken CI blocks the whole team
7. **Keep builds fast** — target under 10 minutes
8. **Test in a production-like environment** — use containers for parity

---

# Level 2 — Core CI Skills

## 2.1 CI Platform Comparison

| Platform | Config File | Language | Free Tier | Best For |
|----------|-------------|----------|-----------|----------|
| **GitHub Actions** | `.github/workflows/*.yml` | YAML | 2,000 min/mo | GitHub projects |
| **GitLab CI/CD** | `.gitlab-ci.yml` | YAML | 400 min/mo | Full DevOps suite |
| **Jenkins** | `Jenkinsfile` | Groovy/YAML | Free (self-hosted) | Enterprise customization |
| **CircleCI** | `.circleci/config.yml` | YAML | 6,000 min/mo | Docker-native speed |
| **Bitbucket Pipelines** | `bitbucket-pipelines.yml` | YAML | 50 min/mo | Atlassian ecosystem |
| **Azure DevOps** | `azure-pipelines.yml` | YAML | 1,800 min/mo | Microsoft/.NET shops |
| **Drone CI** | `.drone.yml` | YAML | Free (self-hosted) | Lightweight, container-native |

> **Recommendation:** Start with **GitHub Actions** or **GitLab CI** — best documentation, native integration, and large community.

---

## 2.2 Anatomy of a Pipeline

### Pipeline Lifecycle

```
┌──────────┐
│  TRIGGER │  push / pull_request / schedule / manual / webhook
└────┬─────┘
     │
┌────▼─────┐
│ CHECKOUT │  clone repository at the specific commit SHA
└────┬─────┘
     │
┌────▼─────┐
│ RESTORE  │  restore dependency cache (npm, pip, cargo, maven…)
│  CACHE   │
└────┬─────┘
     │
┌────▼─────────────────────────────────┐
│              PARALLEL JOBS           │
│  ┌─────────┐  ┌──────┐  ┌────────┐  │
│  │  LINT   │  │BUILD │  │  TEST  │  │
│  └─────────┘  └──────┘  └────────┘  │
└────┬─────────────────────────────────┘
     │
┌────▼─────┐
│  REPORT  │  upload coverage, test results, build logs
└────┬─────┘
     │
┌────▼─────┐
│  NOTIFY  │  Slack, email, PR comment, commit status
└──────────┘
```

### Key Concepts Glossary

| Term | Definition |
|------|-----------|
| **Trigger** | Event that starts the pipeline (push, PR, cron, webhook) |
| **Runner/Agent** | The machine where jobs execute |
| **Job** | An isolated unit of work with its own runner |
| **Step** | A single command or action within a job |
| **Stage** | Logical group of jobs (test, build, deploy) |
| **Artifact** | File(s) produced by a job, passed to downstream jobs |
| **Cache** | Persisted files reused across pipeline runs |
| **Environment** | Named deployment target (dev, staging, prod) |
| **Secret** | Encrypted value injected as an environment variable |

---

## 2.3 Writing Pipelines

### GitHub Actions — Complete Annotated Example

```yaml
# .github/workflows/ci.yml

name: CI                          # Displayed in the GitHub Actions UI

on:                               # Triggers
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:              # Allow manual trigger from UI

env:                              # Global environment variables
  NODE_VERSION: '20'
  REGISTRY: ghcr.io

jobs:
  # ──────────────────────────────────────────
  # JOB 1: Lint
  # ──────────────────────────────────────────
  lint:
    name: Lint & Format Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'            # Automatically caches node_modules

      - name: Install dependencies
        run: npm ci               # Faster than npm install, uses lock file

      - name: Run ESLint
        run: npm run lint

      - name: Check formatting
        run: npm run format:check

  # ──────────────────────────────────────────
  # JOB 2: Test (depends on nothing, runs in parallel with lint)
  # ──────────────────────────────────────────
  test:
    name: Unit & Integration Tests
    runs-on: ubuntu-latest

    services:                     # Spin up a real Postgres for integration tests
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci

      - name: Run tests with coverage
        run: npm test -- --coverage
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          NODE_ENV: test

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 7

      - name: Comment coverage on PR
        if: github.event_name == 'pull_request'
        uses: davelosert/vitest-coverage-report-action@v2

  # ──────────────────────────────────────────
  # JOB 3: Build (runs only after lint + test pass)
  # ──────────────────────────────────────────
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]           # Waits for both jobs to succeed

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci

      - name: Build application
        run: npm run build
        env:
          NODE_ENV: production

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 14

  # ──────────────────────────────────────────
  # JOB 4: Notify (always runs, reports final status)
  # ──────────────────────────────────────────
  notify:
    name: Notify
    runs-on: ubuntu-latest
    needs: [build]
    if: always()                  # Run even if previous jobs failed

    steps:
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "CI ${{ needs.build.result == 'success' && '✅ Passed' || '❌ Failed' }}: ${{ github.repository }}@${{ github.ref_name }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

### GitLab CI — Complete Annotated Example

```yaml
# .gitlab-ci.yml

# ─── Global Configuration ───────────────────────────────────────────
default:
  image: node:20-alpine
  before_script:
    - npm ci --cache .npm --prefer-offline
  cache:
    key:
      files:
        - package-lock.json       # Cache invalidates when lock file changes
    paths:
      - .npm/
      - node_modules/

variables:
  NODE_ENV: test
  POSTGRES_DB: testdb
  POSTGRES_USER: testuser
  POSTGRES_PASSWORD: testpass

stages:
  - validate
  - test
  - build
  - publish

# ─── Stage: validate ────────────────────────────────────────────────
lint:
  stage: validate
  script:
    - npm run lint
    - npm run format:check
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

typecheck:
  stage: validate
  script:
    - npm run typecheck

# ─── Stage: test ────────────────────────────────────────────────────
unit-tests:
  stage: test
  script:
    - npm run test:unit -- --coverage
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'   # Extract coverage % for GitLab badge
  artifacts:
    when: always
    reports:
      junit: junit.xml             # GitLab parses this for test result UI
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths:
      - coverage/
    expire_in: 1 week

integration-tests:
  stage: test
  services:
    - postgres:16
  script:
    - npm run test:integration
  variables:
    DATABASE_URL: "postgresql://testuser:testpass@postgres:5432/testdb"

# ─── Stage: build ───────────────────────────────────────────────────
build:
  stage: build
  script:
    - NODE_ENV=production npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 2 weeks
  only:
    - main
    - tags

# ─── Stage: publish ─────────────────────────────────────────────────
publish-image:
  stage: publish
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main
```

---

### Jenkins — Declarative Pipeline Example

```groovy
// Jenkinsfile

pipeline {
    agent any

    environment {
        NODE_VERSION = '20'
        REGISTRY     = 'registry.example.com'
    }

    tools {
        nodejs "${NODE_VERSION}"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()          // Prevent parallel runs on same branch
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Validate') {
            parallel {
                stage('Lint') {
                    steps { sh 'npm run lint' }
                }
                stage('Type Check') {
                    steps { sh 'npm run typecheck' }
                }
            }
        }

        stage('Test') {
            steps {
                sh 'npm test -- --coverage'
            }
            post {
                always {
                    junit 'junit.xml'
                    publishHTML([
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('Build') {
            when {
                anyOf {
                    branch 'main'
                    tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
                }
            }
            steps {
                sh 'NODE_ENV=production npm run build'
            }
        }
    }

    post {
        success {
            slackSend(color: 'good', message: "✅ Build passed: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
        failure {
            slackSend(color: 'danger', message: "❌ Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n${env.BUILD_URL}")
        }
    }
}
```

---

## 2.4 Automated Testing

### The Test Pyramid

```
            ┌───────────┐
           /│   E2E     │\       Few, slow, expensive
          / └───────────┘ \      Run: nightly or on release
         /─────────────────\
        / │  Integration  │ \    Moderate number
       /  └───────────────┘  \   Run: every PR
      /───────────────────────\
     /    │   Unit Tests    │   \  Many, fast, cheap
    /     └─────────────────┘    \ Run: every commit
   /─────────────────────────────\
```

**Guideline:** 70% unit / 20% integration / 10% E2E

### Unit Tests — Node.js with Vitest

```typescript
// src/utils/validate.ts
export function validateEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

export function validateAge(age: number): boolean {
  return Number.isInteger(age) && age >= 0 && age <= 150;
}
```

```typescript
// src/utils/validate.test.ts
import { describe, it, expect } from 'vitest';
import { validateEmail, validateAge } from './validate';

describe('validateEmail', () => {
  it('accepts a valid email', () => {
    expect(validateEmail('user@example.com')).toBe(true);
  });

  it('rejects email without @', () => {
    expect(validateEmail('userexample.com')).toBe(false);
  });

  it('rejects empty string', () => {
    expect(validateEmail('')).toBe(false);
  });
});

describe('validateAge', () => {
  it('accepts age 0', () => expect(validateAge(0)).toBe(true));
  it('accepts age 25', () => expect(validateAge(25)).toBe(true));
  it('rejects negative age', () => expect(validateAge(-1)).toBe(false));
  it('rejects float', () => expect(validateAge(25.5)).toBe(false));
});
```

### Unit Tests — Python with pytest

```python
# src/utils/validate.py
import re

def validate_email(email: str) -> bool:
    pattern = r'^[^\s@]+@[^\s@]+\.[^\s@]+$'
    return bool(re.match(pattern, email))

def validate_age(age: int) -> bool:
    return isinstance(age, int) and 0 <= age <= 150
```

```python
# tests/test_validate.py
import pytest
from src.utils.validate import validate_email, validate_age

class TestValidateEmail:
    def test_valid_email(self):
        assert validate_email("user@example.com") is True

    def test_missing_at(self):
        assert validate_email("userexample.com") is False

    def test_empty_string(self):
        assert validate_email("") is False

    @pytest.mark.parametrize("email", [
        "a@b.co",
        "user+tag@domain.org",
        "first.last@sub.domain.com",
    ])
    def test_valid_emails(self, email):
        assert validate_email(email) is True


class TestValidateAge:
    def test_zero(self):
        assert validate_age(0) is True

    def test_normal_age(self):
        assert validate_age(25) is True

    def test_negative(self):
        assert validate_age(-1) is False

    def test_float_rejected(self):
        assert validate_age(25.5) is False  # type: ignore
```

### Integration Test — API with Supertest

```typescript
// tests/api/users.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import { app } from '../../src/app';
import { db } from '../../src/db';

beforeAll(async () => {
  await db.migrate.latest();
  await db.seed.run();
});

afterAll(async () => {
  await db.destroy();
});

describe('POST /api/users', () => {
  it('creates a user with valid data', async () => {
    const res = await request(app)
      .post('/api/users')
      .send({ name: 'Alice', email: 'alice@example.com' })
      .expect(201);

    expect(res.body).toMatchObject({
      id: expect.any(Number),
      name: 'Alice',
      email: 'alice@example.com',
    });
  });

  it('rejects invalid email', async () => {
    await request(app)
      .post('/api/users')
      .send({ name: 'Bob', email: 'not-an-email' })
      .expect(422);
  });
});
```

### E2E Test — Playwright

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Login flow', () => {
  test('user can log in with valid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.fill('[data-testid="email"]', 'user@example.com');
    await page.fill('[data-testid="password"]', 'password123');
    await page.click('[data-testid="submit"]');

    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('h1')).toContainText('Welcome');
  });

  test('shows error on invalid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.fill('[data-testid="email"]', 'bad@example.com');
    await page.fill('[data-testid="password"]', 'wrong');
    await page.click('[data-testid="submit"]');

    await expect(page.locator('[data-testid="error"]')).toBeVisible();
    await expect(page.locator('[data-testid="error"]')).toContainText('Invalid credentials');
  });
});
```

```yaml
# playwright.config.ts — CI-optimized settings
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,    // Retry flaky tests in CI only
  workers: process.env.CI ? 4 : undefined,
  reporter: process.env.CI
    ? [['junit', { outputFile: 'e2e-results.xml' }], ['html']]
    : 'list',
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
});
```

### Coverage Enforcement in CI

```yaml
# GitHub Actions — fail if coverage drops below 80%
- name: Run tests with coverage
  run: |
    npm test -- --coverage --coverageThreshold='{"global":{"lines":80,"functions":80,"branches":75}}'
```

```ini
# pytest.ini — enforce coverage threshold
[tool:pytest]
addopts = --cov=src --cov-report=xml --cov-report=term-missing --cov-fail-under=80
```

---

## 2.5 Build Automation

### Makefile — Universal Build Interface

```makefile
# Makefile
.PHONY: install lint test build clean docker-build

install:
	npm ci

lint:
	npm run lint
	npm run format:check

test:
	npm test -- --coverage

build:
	NODE_ENV=production npm run build

clean:
	rm -rf dist/ coverage/ node_modules/

docker-build:
	docker build -t myapp:$(shell git rev-parse --short HEAD) .

# CI-specific target — run everything in order
ci: install lint test build
```

### package.json Scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint src --ext .ts,.tsx --max-warnings 0",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:ui": "vitest --ui",
    "test:e2e": "playwright test",
    "ci": "npm run lint && npm run typecheck && npm run test && npm run build"
  }
}
```

### Dependency Locking Best Practices

```bash
# Always commit lock files — they ensure reproducible builds
git add package-lock.json          # Node.js
git add poetry.lock                # Python
git add Cargo.lock                 # Rust
git add go.sum                     # Go

# Use exact install commands in CI (not `npm install`)
npm ci                             # Node — uses lock file, fails if mismatched
pip install -r requirements.txt    # Python
cargo build                        # Rust — uses Cargo.lock automatically
```

---

# Level 3 — Intermediate CI

## 3.1 Docker in CI

### Multi-Stage Dockerfile

A well-structured Dockerfile that keeps the production image small and the build environment separate:

```dockerfile
# ─── Stage 1: Dependencies ───────────────────────────────────────────
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && cp -r node_modules /tmp/prod-deps
RUN npm ci                                      # Install all deps for build

# ─── Stage 2: Build ──────────────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# ─── Stage 3: Production Image ───────────────────────────────────────
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

# Run as non-root user (security best practice)
RUN addgroup --system --gid 1001 nodejs
RUN adduser  --system --uid 1001 appuser

COPY --from=deps     /tmp/prod-deps ./node_modules
COPY --from=builder  /app/dist      ./dist
COPY --from=builder  /app/package.json ./

USER appuser

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Building and Pushing in GitHub Actions

```yaml
# .github/workflows/docker.yml

name: Build & Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write               # Required to push to GHCR

    steps:
      - uses: actions/checkout@v4

      # Enable Docker layer caching via GitHub cache
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Generate image tags based on git context
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha           # Use GitHub Actions cache for layers
          cache-to: type=gha,mode=max
```

### Docker Layer Caching Strategy

```dockerfile
# ✅ GOOD: Copy package files BEFORE source code
# This way, npm install is only re-run when package.json changes
COPY package*.json ./
RUN npm ci                     # Cached unless package*.json changed
COPY src/ ./src/               # Source changes don't invalidate the npm layer
RUN npm run build

# ❌ BAD: Copy everything at once — any source change busts the cache
COPY . .
RUN npm ci
RUN npm run build
```

---

## 3.2 Code Quality Gates

### ESLint Configuration

```json
// .eslintrc.json
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:import/recommended"
  ],
  "rules": {
    "no-console": "warn",
    "no-unused-vars": "off",
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/no-explicit-any": "warn",
    "import/order": ["error", { "alphabetize": { "order": "asc" } }]
  }
}
```

### Prettier Configuration

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "arrowParens": "avoid"
}
```

### Pre-commit Hooks with Husky

Run quality checks locally before committing — mirrors what CI will do:

```bash
npm install --save-dev husky lint-staged
npx husky init
```

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml}": ["prettier --write"]
  }
}
```

```bash
# .husky/pre-commit
#!/bin/sh
npx lint-staged
```

### SonarQube Integration in CI

```yaml
# GitHub Actions SonarQube scan
- name: SonarQube Scan
  uses: SonarSource/sonarqube-scan-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

```properties
# sonar-project.properties
sonar.projectKey=my-project
sonar.sources=src
sonar.tests=tests
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.testExecutionReportPaths=junit.xml

# Quality gate thresholds
sonar.qualitygate.wait=true       # Fail CI if quality gate fails
```

---

## 3.3 Security Scanning

### Secret Detection — Preventing Leaks Before They Happen

```yaml
# GitHub Actions — scan for secrets in every PR
- name: Scan for secrets (Gitleaks)
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

```toml
# .gitleaks.toml — custom rules
[allowlist]
  description = "Allowlist for known safe patterns"
  regexes = [
    '''EXAMPLE_KEY''',
    '''test_.*_key'''
  ]
  paths = [
    '''tests/fixtures/.*'''
  ]
```

### Dependency Vulnerability Scanning

```yaml
# Scan Node.js dependencies
- name: Audit dependencies
  run: npm audit --audit-level=high
  # Fails if HIGH or CRITICAL vulnerabilities found

# Or use Snyk for richer reports
- name: Snyk vulnerability scan
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

```yaml
# Scan Python dependencies
- name: Safety check
  run: |
    pip install safety
    safety check -r requirements.txt --json > safety-report.json
```

### Container Image Scanning with Trivy

```yaml
- name: Scan Docker image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'ghcr.io/${{ github.repository }}:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'              # Fail the pipeline if vulnerabilities found

- name: Upload Trivy results to GitHub Security tab
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

### CodeQL Static Analysis

```yaml
# .github/workflows/codeql.yml
name: CodeQL Analysis

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'        # Weekly scan every Monday at 6am

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      actions: read
      contents: read

    strategy:
      matrix:
        language: [javascript, python]

    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
```

---

## 3.4 Artifact Management

### Semantic Versioning in CI

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0              # Full history needed for changelog

      - name: Semantic Release
        uses: cycjimmy/semantic-release-action@v4
        with:
          semantic_version: 23
          extra_plugins: |
            @semantic-release/changelog
            @semantic-release/git
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

```json
// .releaserc.json — semantic-release config
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", {
      "changelogFile": "CHANGELOG.md"
    }],
    ["@semantic-release/npm", {
      "npmPublish": true
    }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }],
    "@semantic-release/github"
  ]
}
```

### Publishing to PyPI from CI

```yaml
- name: Build Python package
  run: |
    pip install build
    python -m build

- name: Publish to PyPI
  uses: pypa/gh-action-pypi-publish@release/v1
  with:
    password: ${{ secrets.PYPI_API_TOKEN }}
    # Use TestPyPI first: repository-url: https://test.pypi.org/legacy/
```

---

## 3.5 Notifications & Observability

### Slack Notifications

```yaml
- name: Notify Slack on failure
  if: failure()
  uses: slackapi/slack-github-action@v1.26.0
  with:
    payload: |
      {
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "❌ *CI Failed*: <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>\n*Repo:* ${{ github.repository }}\n*Branch:* `${{ github.ref_name }}`\n*Commit:* `${{ github.sha }}`\n*Author:* ${{ github.actor }}"
            }
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

### Build Status Badge

```markdown
<!-- In README.md -->
![CI](https://github.com/owner/repo/actions/workflows/ci.yml/badge.svg)
![Coverage](https://codecov.io/gh/owner/repo/branch/main/graph/badge.svg)
![Security](https://snyk.io/test/github/owner/repo/badge.svg)
```

---

# Level 4 — Advanced CI

## 4.1 Pipeline Optimization

### Parallelization with Matrix Builds

```yaml
# Test across multiple Node.js versions and operating systems
jobs:
  test:
    strategy:
      fail-fast: false             # Don't cancel other matrix jobs on failure
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]
        exclude:
          - os: windows-latest
            node: 18              # Skip this combination

    runs-on: ${{ matrix.os }}
    name: Test (Node ${{ matrix.node }} / ${{ matrix.os }})

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test
```

### Conditional Execution — Path-Based Triggers

```yaml
# Only run frontend tests when frontend code changes
jobs:
  frontend-tests:
    if: |
      contains(github.event.head_commit.modified, 'frontend/') ||
      contains(github.event.head_commit.added, 'frontend/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd frontend && npm ci && npm test
```

```yaml
# GitLab: rules with changes filter
frontend-test:
  script: cd frontend && npm test
  rules:
    - changes:
        - frontend/**/*
        - package.json

backend-test:
  script: cd backend && go test ./...
  rules:
    - changes:
        - backend/**/*
        - go.mod
        - go.sum
```

### Dependency Graph — DAG Pipelines in GitLab

```yaml
# Jobs run in parallel when they don't share a stage dependency
stages: [build, test, deploy]

build-frontend:
  stage: build
  script: npm run build:frontend
  artifacts:
    paths: [dist/frontend]

build-backend:
  stage: build              # Runs in PARALLEL with build-frontend
  script: go build ./...
  artifacts:
    paths: [bin/]

test-frontend:
  stage: test
  needs: [build-frontend]   # Only waits for build-frontend, not build-backend
  script: npm test

test-backend:
  stage: test
  needs: [build-backend]    # Only waits for build-backend
  script: go test ./...

deploy:
  stage: deploy
  needs: [test-frontend, test-backend]   # Waits for BOTH tests
  script: ./deploy.sh
```

### Advanced Caching Strategies

```yaml
# GitHub Actions — cache npm with composite key
- name: Cache node modules
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-     # Partial match fallback

# Cache pip packages
- name: Cache pip
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
    restore-keys: ${{ runner.os }}-pip-

# Cache Rust/Cargo (more complex — cache both registry and target)
- name: Cache Cargo
  uses: Swatinem/rust-cache@v2
  with:
    workspaces: ". -> target"
    cache-on-failure: true
```

---

## 4.2 Self-Hosted Runners

### GitHub Actions Self-Hosted Runner Setup

```bash
# On your server/VM — register a runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.317.0/actions-runner-linux-x64-2.317.0.tar.gz
tar xzf ./actions-runner-linux-x64.tar.gz

# Register with your repo
./config.sh --url https://github.com/OWNER/REPO --token YOUR_TOKEN

# Install and start as a service
sudo ./svc.sh install
sudo ./svc.sh start
```

```yaml
# Use your self-hosted runner in a workflow
jobs:
  gpu-job:
    runs-on: [self-hosted, linux, gpu]     # Target by labels
    steps:
      - run: nvidia-smi                    # Only works on your GPU runner
```

### Ephemeral Runners with Docker

```dockerfile
# Dockerfile.runner
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    curl git jq libicu70 \
    && rm -rf /var/lib/apt/lists/*

# Install GitHub Actions runner
ARG RUNNER_VERSION=2.317.0
RUN curl -o /tmp/runner.tar.gz -L \
    https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
    && tar xzf /tmp/runner.tar.gz -C /opt/actions-runner

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

```bash
#!/bin/bash
# entrypoint.sh — register, run once, then deregister (ephemeral)
/opt/actions-runner/config.sh \
  --url "${GITHUB_URL}" \
  --token "${RUNNER_TOKEN}" \
  --name "ephemeral-$(hostname)" \
  --ephemeral \
  --unattended

/opt/actions-runner/run.sh
```

---

## 4.3 Advanced Testing Strategies

### Test Sharding — Split a Large Suite Across Runners

```yaml
# GitHub Actions — shard Playwright E2E tests across 4 runners
jobs:
  e2e:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]

    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx playwright test --shard=${{ matrix.shard }}/4
      - uses: actions/upload-artifact@v4
        with:
          name: blob-report-${{ matrix.shard }}
          path: blob-report/

  # Merge shard reports into one HTML report
  merge-reports:
    needs: [e2e]
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: blob-report-*
          merge-multiple: true
          path: all-blob-reports
      - run: npx playwright merge-reports --reporter html ./all-blob-reports
```

### Contract Testing with Pact

```typescript
// consumer.pact.test.ts — define the contract from the consumer's perspective
import { PactV3, MatchersV3 } from '@pact-foundation/pact';

const provider = new PactV3({
  consumer: 'OrderService',
  provider: 'UserService',
});

describe('UserService contract', () => {
  it('returns a user by ID', () => {
    return provider
      .given('User 123 exists')
      .uponReceiving('a request for user 123')
      .withRequest({ method: 'GET', path: '/users/123' })
      .willRespondWith({
        status: 200,
        body: {
          id: MatchersV3.integer(123),
          name: MatchersV3.string('Alice'),
          email: MatchersV3.email('alice@example.com'),
        },
      })
      .executeTest(async (mockServer) => {
        const client = new UserClient(mockServer.url);
        const user = await client.getUser(123);
        expect(user.name).toBe('Alice');
      });
  });
});
```

### Performance Testing with k6

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '30s', target: 20 },   // Ramp up to 20 users
    { duration: '1m',  target: 20 },   // Stay at 20
    { duration: '20s', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests under 500ms
    errors: ['rate<0.01'],             // Error rate under 1%
  },
};

export default function () {
  const res = http.get(`${__ENV.BASE_URL}/api/users`);

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  errorRate.add(res.status !== 200);
  sleep(1);
}
```

```yaml
# Run k6 in CI
- name: Run k6 load test
  uses: grafana/k6-action@v0.3.1
  with:
    filename: load-test.js
  env:
    BASE_URL: https://staging.example.com
    K6_CLOUD_TOKEN: ${{ secrets.K6_CLOUD_TOKEN }}
```

---

## 4.4 GitOps & CI/CD Integration

### CI → CD Trigger Pattern

```yaml
# CI pipeline (build repo) — triggers CD on success
- name: Trigger deployment pipeline
  if: github.ref == 'refs/heads/main' && success()
  uses: peter-evans/repository-dispatch@v3
  with:
    token: ${{ secrets.CD_DISPATCH_TOKEN }}
    repository: org/cd-config-repo
    event-type: deploy-staging
    client-payload: |
      {
        "image": "ghcr.io/${{ github.repository }}",
        "tag": "${{ github.sha }}",
        "environment": "staging"
      }
```

### Kubernetes Manifest Update (GitOps)

```yaml
# In the CD repo — update image tag and commit (ArgoCD/Flux picks it up)
- name: Update image tag in Kubernetes manifest
  run: |
    IMAGE_TAG=${{ github.event.client_payload.tag }}
    sed -i "s|image: .*|image: ghcr.io/org/app:${IMAGE_TAG}|" k8s/staging/deployment.yaml
    git config user.name "ci-bot"
    git config user.email "ci-bot@example.com"
    git add k8s/staging/deployment.yaml
    git commit -m "chore: deploy ${IMAGE_TAG} to staging [skip ci]"
    git push
```

### Environment Promotion Pipeline

```yaml
# GitLab CI — promote through environments
stages: [build, deploy-dev, deploy-staging, deploy-prod]

.deploy-template: &deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/app app=$IMAGE:$CI_COMMIT_SHA -n $NAMESPACE

deploy-dev:
  <<: *deploy
  stage: deploy-dev
  variables:
    NAMESPACE: dev
  environment:
    name: development
    url: https://dev.example.com
  only: [main]

deploy-staging:
  <<: *deploy
  stage: deploy-staging
  variables:
    NAMESPACE: staging
  environment:
    name: staging
    url: https://staging.example.com
  when: manual                       # Require human approval
  only: [main]

deploy-prod:
  <<: *deploy
  stage: deploy-prod
  variables:
    NAMESPACE: production
  environment:
    name: production
    url: https://example.com
  when: manual
  only: [main]
  allow_failure: false
```

---

## 4.5 Feature Flags & Progressive Delivery

### Feature Flag Pattern in Code

```typescript
// Using OpenFeature SDK
import { OpenFeature } from '@openfeature/server-sdk';

const client = OpenFeature.getClient('my-app');

export async function handleCheckout(userId: string) {
  const useNewCheckout = await client.getBooleanValue(
    'new-checkout-flow',
    false,                           // Default: disabled
    { targetingKey: userId }
  );

  if (useNewCheckout) {
    return newCheckoutFlow(userId);
  } else {
    return legacyCheckoutFlow(userId);
  }
}
```

### Canary Release with GitHub Actions + Kubernetes

```yaml
- name: Deploy canary (10% traffic)
  run: |
    # Deploy new version as canary
    kubectl apply -f k8s/canary/deployment.yaml

    # Set traffic split: 90% stable, 10% canary
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: app-vs
    spec:
      http:
        - route:
            - destination:
                host: app-stable
              weight: 90
            - destination:
                host: app-canary
              weight: 10
    EOF

- name: Monitor canary for 10 minutes
  run: |
    sleep 600
    ERROR_RATE=$(curl -s prometheus/api/v1/query?query=canary_error_rate | jq '.data.result[0].value[1]')
    if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
      echo "Error rate too high, rolling back"
      kubectl rollout undo deployment/app-canary
      exit 1
    fi
```

---

## 4.6 Infrastructure as Code in CI

### Terraform Validation Pipeline

```yaml
# .github/workflows/terraform.yml
name: Terraform CI

on:
  pull_request:
    paths:
      - 'infrastructure/**'

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infrastructure/

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: terraform init -backend=false   # Skip remote backend in PR

      - name: Terraform Validate
        run: terraform validate

      - name: Run tflint (Terraform linter)
        uses: terraform-linters/setup-tflint@v4
      - run: tflint --recursive

      - name: Run Checkov (security scan)
        uses: bridgecrewio/checkov-action@master
        with:
          directory: infrastructure/
          framework: terraform
          output_format: sarif
          output_file_path: checkov.sarif

      - name: Terraform Plan
        run: terraform plan -out=plan.tfplan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Comment plan on PR
        uses: actions/github-script@v7
        with:
          script: |
            const plan = require('fs').readFileSync('plan.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Terraform Plan\n\`\`\`hcl\n${plan.slice(0, 60000)}\n\`\`\``
            });
```

---

# Level 5 — Mastery

## 5.1 CI Architecture at Scale

### Monorepo CI with Nx (Affected-Only Builds)

```json
// nx.json
{
  "affected": {
    "defaultBase": "main"
  },
  "tasksRunnerOptions": {
    "default": {
      "runner": "@nrwl/nx-cloud",
      "options": {
        "cacheableOperations": ["build", "test", "lint"],
        "accessToken": "your-nx-cloud-token"   // Distributed cache
      }
    }
  }
}
```

```yaml
# Only test/build affected projects — not the entire monorepo
- name: Get affected projects
  run: |
    AFFECTED=$(npx nx show projects --affected --type=app | tr '\n' ',')
    echo "AFFECTED=${AFFECTED}" >> $GITHUB_ENV

- name: Run affected tests
  run: npx nx affected --target=test --parallel=4

- name: Build affected apps
  run: npx nx affected --target=build --parallel=2
```

### Distributed Caching with Bazel

```python
# .bazelrc — remote cache configuration
build --remote_cache=grpcs://cache.example.com:443
build --remote_header=Authorization=Bearer $BAZEL_CACHE_TOKEN
build --disk_cache=~/.cache/bazel

# Enable remote execution for CI
build:ci --remote_executor=grpcs://rbe.example.com:443
build:ci --remote_instance_name=default
```

---

## 5.2 Platform Engineering

### Reusable Workflow Library (GitHub)

```yaml
# .github/workflows/_reusable-node-ci.yml — shared template

name: Reusable Node.js CI

on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '20'
      working-directory:
        type: string
        default: '.'
      test-command:
        type: string
        default: 'npm test'
    secrets:
      SLACK_WEBHOOK_URL:
        required: false

jobs:
  ci:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
          cache-dependency-path: ${{ inputs.working-directory }}/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: ${{ inputs.test-command }}
      - run: npm run build
```

```yaml
# Consuming the reusable workflow in any project
name: CI
on: [push, pull_request]

jobs:
  ci:
    uses: org/.github/.github/workflows/_reusable-node-ci.yml@main
    with:
      node-version: '22'
      test-command: 'npm run test:coverage'
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### GitLab CI/CD Components (Catalog)

```yaml
# components/node-ci/templates/node.yml
spec:
  inputs:
    node_version:
      default: '20'
    coverage_threshold:
      default: '80'

---
node-lint:
  image: node:$[[ inputs.node_version ]]-alpine
  script:
    - npm ci
    - npm run lint

node-test:
  image: node:$[[ inputs.node_version ]]-alpine
  script:
    - npm ci
    - npm test -- --coverage --coverageThreshold='{"global":{"lines":$[[ inputs.coverage_threshold ]]}}'
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'
```

```yaml
# Consuming the component
include:
  - component: $CI_SERVER_FQDN/org/ci-catalog/node-ci@v1.0
    inputs:
      node_version: '22'
      coverage_threshold: '85'
```

---

## 5.3 DORA Metrics

### Tracking the Four Key Metrics

| Metric | How to Measure | Target (Elite) |
|--------|---------------|---------------|
| **Deployment Frequency** | Count deploys to prod per day via CI logs | Multiple/day |
| **Lead Time for Changes** | Time from first commit to prod deploy | < 1 hour |
| **Change Failure Rate** | % of deploys that trigger a rollback or incident | < 5% |
| **MTTR** | Time between incident alert and resolved deploy | < 1 hour |

### Collecting Metrics from GitHub Actions

```python
# scripts/collect_dora.py — parse workflow runs and emit metrics

import requests
import datetime

GITHUB_TOKEN = os.environ['GITHUB_TOKEN']
REPO = 'org/my-app'

headers = {'Authorization': f'Bearer {GITHUB_TOKEN}'}

def get_deploy_frequency(days=30):
    since = (datetime.datetime.now() - datetime.timedelta(days=days)).isoformat()
    runs = requests.get(
        f'https://api.github.com/repos/{REPO}/actions/workflows/deploy.yml/runs',
        params={'status': 'success', 'branch': 'main', 'created': f'>{since}'},
        headers=headers
    ).json()

    total_deploys = runs['total_count']
    per_day = total_deploys / days
    return {'total': total_deploys, 'per_day': round(per_day, 2)}

def get_lead_time():
    # Compare first commit timestamp vs successful deploy timestamp
    # Emit to Grafana/Datadog/custom dashboard
    pass
```

---

## 5.4 CI Culture

### The CI Social Contract

These are team agreements — equally as important as the technical setup:

```markdown
## Our CI Contract

1. **Green main is sacred** — never merge a PR that breaks main.
2. **Fix the build before anything else** — a broken pipeline blocks everyone.
3. **You broke it, you fix it** — the author of the breaking commit owns the fix.
4. **Don't disable tests to make CI pass** — fix the root cause.
5. **PRs stay small** — large PRs are review liabilities and merge risks.
6. **Short-lived branches** — branches older than 2 days need a plan.
7. **No "works on my machine"** — if CI fails, it's a real problem.
8. **Review pipeline metrics weekly** — slow pipelines are tech debt.
```

### Trunk-Based Development Checklist

```markdown
## TBD Readiness Checklist

Infrastructure:
- [ ] CI pipeline runs in < 10 minutes
- [ ] Branch protection enforces CI before merge
- [ ] Automated deployment to at least one environment on merge to main

Practices:
- [ ] Feature flags are available for hiding incomplete features
- [ ] All developers commit to main (or short-lived branches < 1 day)
- [ ] PR review SLA: < 2 hours during business hours
- [ ] On-call rotation to handle build failures quickly

Code Health:
- [ ] Test suite is fast enough to run locally in < 2 minutes
- [ ] Flaky tests are quarantined and tracked
- [ ] Build produces the same output regardless of environment
```

---

# Reference: Full Pipeline Examples

## Complete Node.js TypeScript CI/CD Pipeline

```yaml
# .github/workflows/full-ci-cd.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

concurrency:                                  # Cancel in-progress runs for same PR
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ─── Quality Checks (parallel) ──────────────────────────────────────
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run lint && npm run typecheck

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm audit --audit-level=high
      - uses: gitleaks/gitleaks-action@v2
        env: { GITHUB_TOKEN: '${{ secrets.GITHUB_TOKEN }}' }

  # ─── Tests (parallel) ───────────────────────────────────────────────
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: --health-cmd pg_isready --health-interval 10s --health-retries 5
        ports: ['5432:5432']
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run test:integration
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

  # ─── Build ──────────────────────────────────────────────────────────
  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: [lint, security, unit-tests, integration-tests]
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}

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
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=sha,prefix=sha-
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
      - uses: docker/build-push-action@v5
        with:
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ─── Deploy to Staging (on merge to main) ───────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - run: |
          echo "Deploying ${{ needs.build.outputs.image-tag }} to staging"
          # kubectl / helm / ArgoCD / etc.

  # ─── Deploy to Production (on tag) ──────────────────────────────────
  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build, deploy-staging]
    if: startsWith(github.ref, 'refs/tags/v')
    environment:
      name: production
      url: https://example.com

    steps:
      - run: |
          echo "Deploying ${{ needs.build.outputs.image-tag }} to production"
```

---

## Complete Python FastAPI CI Pipeline

```yaml
# .github/workflows/python-ci.yml
name: Python CI

on: [push, pull_request]

jobs:
  ci:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.11', '3.12']

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Lint with Ruff
        run: ruff check . --output-format=github

      - name: Format check with Black
        run: black --check .

      - name: Type check with mypy
        run: mypy src/

      - name: Run tests
        run: pytest -v --cov=src --cov-report=xml --cov-fail-under=80

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: coverage.xml
```

---

*This guide is a living document. CI tooling evolves rapidly — always verify against official documentation.*  
*Last updated: March 2026*
