# Level 2 — Core CI Skills

## 2.1 CI Platform Comparison

| Platform | Config File | Free Tier (cloud) | Self-hostable | Best For |
|----------|-------------|-------------------|---------------|----------|
| **GitHub Actions** | `.github/workflows/*.yml` | 2,000 min/mo (public repos: unlimited) | ✅ | GitHub projects |
| **GitLab CI/CD** | `.gitlab-ci.yml` | 400 min/mo | ✅ | Full DevOps suite |
| **Jenkins** | `Jenkinsfile` | ❌ (no cloud offering) | ✅ free | Enterprise customization |
| **CircleCI** | `.circleci/config.yml` | 6,000 min/mo | ✅ | Docker-native speed |
| **Bitbucket Pipelines** | `bitbucket-pipelines.yml` | 50 min/mo | ❌ | Atlassian ecosystem |
| **Azure DevOps** | `azure-pipelines.yml` | 1,800 min/mo (public: unlimited) | ✅ | Microsoft/.NET shops |
| **Drone CI** | `.drone.yml` | ❌ (no cloud offering) | ✅ free | Lightweight, container-native |

> **Recommendation:** Start with **GitHub Actions** or **GitLab CI** — best documentation, native integration, and large community.

### Understanding the Free Tier

"Free tier" in cloud CI platforms means **compute minutes per month** billed against a shared runner fleet. It is not unlimited free usage — it is a monthly quota that resets on a billing cycle. Understanding how minutes are counted and when you will exhaust them avoids surprise bills.

#### How Minutes Are Counted

A "minute" in CI is a **runner-minute**: one minute of wall-clock time on one runner. Parallel jobs each consume their own minutes simultaneously.

```text
Example pipeline with 3 parallel jobs, each taking 5 minutes:

  lint      ████████████  5 min
  test      ████████████  5 min   ← all run at the same time
  build     ████████████  5 min

  Wall-clock time elapsed:  5 minutes
  Minutes billed:          15 minutes  (3 jobs × 5 min)
```

Runner OS also affects billing on some platforms. GitHub Actions applies a multiplier to non-Linux runners:

| OS | GitHub Actions multiplier |
|----|--------------------------|
| Linux (ubuntu-*) | 1× (base rate) |
| Windows (windows-*) | 2× |
| macOS (macos-*) | 10× |

A 10-minute macOS job costs 100 minutes of your free quota. This is relevant if you are building mobile apps or testing cross-platform CLI tools.

#### Free Tier Realities Per Platform

##### GitHub Actions — 2,000 min/mo on private repos

The most important exception: **public repositories get unlimited free minutes** on GitHub-hosted runners. This makes GitHub Actions the default choice for open source projects.

```text
Private repo:   2,000 min/mo  on Linux  (~33 hours of single-job pipelines)
                  200 min/mo  equivalent on macOS (10× multiplier)
Public repo:    Unlimited     on all OS
```

A team running 20 PRs/day with a 6-minute pipeline uses roughly:

```text
20 PRs × 6 min × 22 working days = 2,640 min/mo
→ Exceeds the free tier on private repos
→ Fine on public repos
```

##### GitLab CI — 400 min/mo (shared runners)

GitLab's free tier is significantly smaller. It suits individuals and very small teams but is quickly exhausted by active development workflows. The practical alternative: **register your own machine as a GitLab Runner** — this is free, has no minute limit, and is how most self-hosted GitLab setups operate.

```text
400 min/mo ÷ 22 working days ≈ 18 min/day of shared runner time

If your pipeline takes 8 min and you push 3 times a day:
  3 × 8 min = 24 min/day → quota runs out mid-month
```

##### CircleCI — 6,000 min/mo

The largest free cloud quota, but limited to 1 concurrent job on free plans. Parallelism (multiple jobs running at the same time) requires a paid plan.

```text
Free plan:  6,000 min/mo, 1 concurrent job
            Jobs queue rather than run in parallel
            → a pipeline with 4 jobs runs sequentially, not in parallel
```

##### Bitbucket Pipelines — 50 min/mo

Effectively a trial tier. 50 minutes is consumed by a single working day of normal development. Only viable for very low-frequency pipelines or as a trigger for external systems.

##### Jenkins and Drone CI — no cloud offering

These platforms have no managed cloud runner. "Free" means the software licence is free, but **you provide and pay for the compute yourself** (a VM, a bare-metal server, or a container on your own infrastructure). There is no minute quota, but there are real costs: hardware, electricity, maintenance, and runner uptime.

#### When to Move Off the Free Tier

Signs you have outgrown a cloud free tier:

```text
1. Pipelines are queuing — jobs wait minutes before a runner picks them up
2. You are rationing pushes to save minutes
3. You are skipping CI runs to preserve quota
4. Your monthly bill spikes unexpectedly at day 18

Solutions, in order of preference:
  a) Self-host one or more runners → unlimited minutes on your own hardware
  b) Optimize pipelines to reduce per-run minute consumption
  c) Upgrade to a paid plan for the parallelism and quota you actually need
```

#### Self-Hosted Runners: Zero Minute Cost

Every platform that supports self-hosted runners exempts them from minute quotas entirely. Jobs on your own machine do not consume any cloud minutes regardless of how long they run.

```text
GitHub Actions self-hosted runner:
  runs-on: self-hosted          ← this job costs $0 in GitHub minutes
  runs-on: [self-hosted, linux, gpu]   ← same, with label targeting
```

This is the standard solution for teams with GPU workloads, private network access, or simply high pipeline volume. See [Section 4.2 — Self-Hosted Runners](04-advanced-ci.md#42-self-hosted-runners) for setup details.

---

## 2.2 Anatomy of a Pipeline

### Pipeline Lifecycle

```text
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

| Term             | Definition                                               |
|------------------|----------------------------------------------------------|
| **Trigger**      | Event that starts the pipeline (push, PR, cron, webhook) |
| **Runner/Agent** | The machine where jobs execute                           |
| **Job**          | An isolated unit of work with its own runner             |
| **Step**         | A single command or action within a job                  |
| **Stage**        | Logical group of jobs (test, build, deploy)              |
| **Artifact**     | File(s) produced by a job, passed to downstream jobs     |
| **Cache**        | Persisted files reused across pipeline runs              |
| **Environment**  | Named deployment target (dev, staging, prod)             |
| **Secret**       | Encrypted value injected as an environment variable      |

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

```text
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

#### playwright.config.ts — CI-optimized settings

```typescript
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
