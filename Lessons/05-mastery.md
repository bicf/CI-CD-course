# Level 5 — Mastery

## 5.1 CI Architecture at Scale

### Monorepo CI with Nx (Affected-Only Builds)

#### nx.json

```json
{
  "affected": {
    "defaultBase": "main"
  },
  "tasksRunnerOptions": {
    "default": {
      "runner": "@nrwl/nx-cloud",
      "options": {
        "cacheableOperations": ["build", "test", "lint"],
        "accessToken": "your-nx-cloud-token"
      }
    }
  }
}
```

`nrwl/nx-cloud` used as distributed cache

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

| Metric                    | How to Measure                                   | Target (Elite) |
|---------------------------|--------------------------------------------------|----------------|
| **Deployment Frequency**  | Count deploys to prod per day via CI logs        | Multiple/day   |
| **Lead Time for Changes** | Time from first commit to prod deploy            | < 1 hour       |
| **Change Failure Rate**   | % of deploys that trigger a rollback or incident | < 5%           |
| **MTTR**                  | Time between incident alert and resolved deploy  | < 1 hour       |

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
