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
