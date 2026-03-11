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

# Cached unless package*.json changed
RUN npm ci
# Source changes don't invalidate the npm layer
COPY src/ ./src/
RUN npm run build

# ❌ BAD: Copy everything at once — any source change busts the cache
COPY . .
RUN npm ci
RUN npm run build
```

---

## 3.2 Code Quality Gates

### ESLint Configuration

#### .eslintrc.json

```json
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

#### .prettierrc

```json
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

#### package.json

```json
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

#### .releaserc.json — semantic-release config

```json
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
