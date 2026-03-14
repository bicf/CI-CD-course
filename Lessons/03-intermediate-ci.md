# Level 3 — Intermediate CI

## 3.1 Docker in CI

### Why Docker in a CI Pipeline

Without Docker, a CI pipeline runs directly on the host runner — a shared machine with a pre-installed OS, language runtimes, and system libraries. This creates a class of problems that become more painful as a project grows:

```text
Without Docker:

  Runner OS: Ubuntu 22.04
  Node version: 18.x  (whatever the runner has installed)
  Python version: 3.10 (same)
  System libs: libpq 14, openssl 1.1 ...

  Your app expects: Node 20, Python 3.12, libpq 16
  Result: "works on my machine" — fails in CI, or worse,
          passes in CI and fails in production
```

Docker solves this by packaging the application **together with its exact environment** into a self-contained image. The image runs identically on a developer's laptop, a CI runner, a staging server, and a production Kubernetes cluster — because they are all running the exact same filesystem snapshot.

**The three reasons to use Docker in CI:**

1. **Environment parity.** The image that CI builds and tests is the exact same artifact that gets deployed. Not "the same code built in a different environment" — the same binary image, byte for byte. If it passes tests in CI, it will behave the same way in production.

2. **Reproducibility.** The Dockerfile is a precise, version-controlled recipe. Anyone can rebuild the image from scratch and get an identical result. No hidden state, no "it worked last week" surprises.

3. **Isolation.** Multiple projects with conflicting dependencies (different Node versions, different system libraries) can run on the same CI runner without interfering with each other, because each runs inside its own container.

### The Goal: From Source Code to a Pullable Image

The end goal of the Docker section of a CI pipeline is to produce a **tagged, versioned image stored in a container registry** (in this guide, GHCR — GitHub Container Registry). Once the image is there, any downstream system — a deployment pipeline, a Kubernetes cluster, a developer's local machine — can pull and run it without needing the source code, build tools, or any knowledge of how it was built.

```text
Source code                     Container Registry (GHCR)
─────────────                   ──────────────────────────
src/
package.json    ──► CI ──►      ghcr.io/org/repo:main
Dockerfile                      ghcr.io/org/repo:1.2.0
                                ghcr.io/org/repo:sha-a3f2c1d
                                        │
                                        ▼
                                Kubernetes / Docker / any server
                                docker pull ghcr.io/org/repo:sha-a3f2c1d
                                docker run  ghcr.io/org/repo:sha-a3f2c1d
```

**Why push to a registry rather than build on every server?**

Building an image takes time and requires build tools, source code, and network access to download dependencies. A registry decouples the build from the deployment: you build once in CI, store the result, and deploy the stored artifact everywhere. Servers in production do not need compilers, package managers, or source code — they only need `docker pull`.

**Why tag by commit SHA?**

The `sha-a3f2c1d` tag is immutable — it always refers to the exact image built from that specific commit. Tags like `:latest` or `:main` are mutable and move with every push, making it impossible to know which version is actually running. In production deployments, always pin to the SHA tag.

```text
Mutable tag (risky in production):
  image: ghcr.io/org/repo:latest    ← changes with every push, no audit trail

Immutable tag (safe):
  image: ghcr.io/org/repo:sha-a3f2c1d  ← always this exact build
```

### The Three-File Setup

A complete Docker-in-CI setup involves three files working together:

```text
Repository:
  ├── Dockerfile                       (1) defines the image build
  ├── .dockerignore                    (2) controls what enters the build context
  └── .github/workflows/docker.yml    (3) automates build + push on every commit
```

| File | Responsibility | Runs on |
| --- | --- | --- |
| `Dockerfile` | How to build the image, layer by layer | Docker daemon on the CI runner |
| `.dockerignore` | Which files to exclude from the build context | Docker daemon, before any layer |
| `docker.yml` | When to trigger the build, how to authenticate, where to push | GitHub Actions runner |

The sections below cover each in detail.

---

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

The workflow file and the Dockerfile are two separate files with distinct responsibilities that work together as a pipeline:

```text
Repository on disk:
  ├── Dockerfile                        ← defines HOW to build the image
  └── .github/
      └── workflows/
          └── docker.yml                ← defines WHEN to build and WHERE to push it
```

The Dockerfile is not referenced by name anywhere in the workflow YAML. Instead, the `docker/build-push-action` step uses the `context: .` parameter — meaning "the root of the checked-out repository" — and Docker automatically looks for a file named `Dockerfile` in that directory. The multi-stage build defined in the Dockerfile is what runs when the action executes `docker build`.

**End-to-end flow from a push to a pullable image:**

```text
1. Developer pushes to main (or creates a tag like v1.2.0)
        │
        ▼
2. GitHub Actions triggers docker.yml
        │
        ▼
3. Runner checks out the repository
   (Dockerfile is now present on the runner at ./Dockerfile)
        │
        ▼
4. docker/build-push-action runs docker build
   ┌─────────────────────────────────────────────────┐
   │  Executes the Dockerfile stages in order:       │
   │  Stage 1 (deps)    — installs node_modules      │
   │  Stage 2 (builder) — runs npm run build         │
   │  Stage 3 (runner)  — assembles production image │
   └─────────────────────────────────────────────────┘
        │
        ▼
5. The final image (Stage 3 only) is tagged and pushed to GHCR
   ghcr.io/org/repo:main
   ghcr.io/org/repo:sha-a3f2c1d
   ghcr.io/org/repo:1.2.0          ← if triggered by a tag
        │
        ▼
6. Image is now pullable by anyone with access to the repo:
   docker pull ghcr.io/org/repo:main
```

> **Important:** only the final stage of the multi-stage Dockerfile becomes the pushed image. The intermediate stages (`deps`, `builder`) are used during the build and then discarded. They never appear in GHCR.

```yaml
# .github/workflows/docker.yml

name: Build & Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*']             # Matches v1.0.0, v2.3.1, etc.

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write        # Required to push to GHCR — without this
                             # the GITHUB_TOKEN cannot write to the registry

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        # After this step, the runner has the full repo on disk,
        # including the Dockerfile at ./Dockerfile

      # Enable Docker layer caching via GitHub cache
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        # Buildx is an extended Docker build client that supports
        # multi-platform builds and the GitHub Actions cache backend
        # (cache-from: type=gha). The standard `docker build` command
        # does not support the GHA cache backend.

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}    # The user who triggered the workflow
          password: ${{ secrets.GITHUB_TOKEN }}
        # GITHUB_TOKEN is automatically available in every workflow —
        # no manual secret setup required. It is scoped to this repo
        # and expires when the workflow run ends.
        # The `packages: write` permission above is what allows it to push.
      
      # Generate image tags based on git context
      - name: Extract metadata (tags and labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          # github.repository = "org/repo-name"
          # Full image name = "ghcr.io/org/repo-name"
          tags: |
            type=ref,event=branch
            # → ghcr.io/org/repo:main  (on push to main)

            type=semver,pattern={{version}}
            # → ghcr.io/org/repo:1.2.0  (on tag v1.2.0)

            type=semver,pattern={{major}}.{{minor}}
            # → ghcr.io/org/repo:1.2    (floating minor tag)

            type=sha,prefix=sha-
            # → ghcr.io/org/repo:sha-a3f2c1d  (always unique, per commit)
        # The sha tag is the most useful for deployment pipelines —
        # it is immutable and always points to the exact commit that was built.

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          # "." means: use the root of the checked-out repo as the build context.
          # Docker will look for ./Dockerfile automatically.
          # To use a different path: context: ./services/api
          # To use a different filename: file: ./docker/Dockerfile.prod

          push: true
          # push: false would build the image without pushing — useful for
          # PR validation where you want to confirm the image builds
          # but don't want to publish it yet.

          tags: ${{ steps.meta.outputs.tags }}
          # The list of ghcr.io/... tags computed in the metadata step above.
          # One image, multiple tags pointing to it.

          labels: ${{ steps.meta.outputs.labels }}
          # OCI standard labels: org.opencontainers.image.source,
          # .created, .revision, etc. Makes the image traceable back
          # to the exact commit and workflow run that built it.

          cache-from: type=gha
          cache-to: type=gha,mode=max
          # Stores Docker layer cache in GitHub Actions Cache (not in GHCR).
          # On subsequent runs, unchanged layers are restored from cache
          # instead of being rebuilt — the same benefit as the Dockerfile
          # layer ordering described above, but persisted across runner instances.
          # mode=max caches all intermediate layers, not just the final stage.
```

**Where the image lands in GHCR and how to pull it:**

Once the workflow completes, the image is visible at:

```text
https://github.com/ORG/REPO/pkgs/container/REPO
```

By default, a newly pushed image inherits the visibility of the repository (public repo → public image, private repo → private image). To pull it:

```bash
# Public image — no authentication needed
docker pull ghcr.io/org/repo:main

# Private image — authenticate first
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
docker pull ghcr.io/org/repo:main

# Pull a specific immutable build by commit SHA
docker pull ghcr.io/org/repo:sha-a3f2c1d

# In a Kubernetes manifest
# image: ghcr.io/org/repo:sha-a3f2c1d   ← prefer SHA over :latest in production
```

### Docker Layer Caching Strategy

Docker builds images as a stack of layers. Each `COPY` and `RUN` instruction creates a new layer. When a layer changes, Docker invalidates that layer **and every layer above it** — forcing all subsequent instructions to re-run from scratch. Understanding this cascade is the key to fast, cache-efficient builds in CI.

```text
Layer stack (top = last instruction):

  [5] RUN npm run build      ← rebuilt if layer 4 changes
  [4] COPY src/ ./src/       ← rebuilt if layer 3 changes or source files change 
  [3] RUN npm ci             ← rebuilt if layer 2 changes
  [2] COPY package*.json ./  ← rebuilt if base layer changes or package.json changes
  [1] FROM node:20-alpine    ← base image, rarely changes
```

The rule: **put instructions that change rarely near the bottom, instructions that change often near the top.**

---

#### ✅ Correct Pattern — Separate Dependencies from Source

Copy the package manifest files first, install dependencies, then copy source code. This way a source file change only reruns the build step, not the install step.

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app

# Step 1 — copy only the files that define your dependencies
COPY package.json package-lock.json ./

# Step 2 — install; this layer is cached as long as the lock file
#           does not change, regardless of any source code edits
RUN npm ci

# Step 3 — now copy source; changes here only invalidate layers above
COPY src/ ./src/
COPY tsconfig.json ./

# Step 4 — build; re-runs only when source changes, not on every commit
RUN npm run build
```

What gets cached and what gets rebuilt on a typical source-only change:

```text
Commit: changed src/api/users.ts

  [1] FROM node:20-alpine       ✅ cache hit
  [2] COPY package*.json        ✅ cache hit  (manifest unchanged)
  [3] RUN npm ci                ✅ cache hit  (skipped — ~60s saved)
  [4] COPY src/                 ❌ cache miss (source changed)
  [5] RUN npm run build         ❌ re-runs    (~10s)

  Total: ~10s instead of ~70s
```

The same pattern applies to other ecosystems:

```dockerfile
# Python
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY src/ ./src/

# Go
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build ./...

# Java / Maven
COPY pom.xml ./
RUN mvn dependency:go-offline -q
COPY src/ ./src/
RUN mvn package -DskipTests
```

---

#### ❌ Common Mistake — Copying Everything at Once

The most common mistake: a single `COPY . .` before installing dependencies. Any change to any file in the project — including a one-line edit in a README — busts the dependency cache.

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app

# ❌ This copies source, tests, docs, configs — everything
COPY . .

# ❌ npm ci re-runs on EVERY build because the layer above always changes
#    Even if package.json hasn't been touched
RUN npm ci
RUN npm run build
```

What happens on every single commit, including trivial ones:

```text
Commit: fixed a typo in README.md

  [1] FROM node:20-alpine       ✅ cache hit
  [2] COPY . .                  ❌ cache miss (any file change breaks this)
  [3] RUN npm ci                ❌ re-runs    (~60s wasted)
  [4] RUN npm run build         ❌ re-runs    (~10s)

  Total: ~70s on every push, even for a typo fix
```

Other variants of the same mistake:

```dockerfile
# ❌ Variant: copying the whole project before the lock file specifically
COPY . .
COPY package-lock.json ./    # too late — layer 1 already invalidated

# ❌ Variant: using ADD instead of COPY (adds remote fetch overhead too)
ADD . /app

# ❌ Variant: no .dockerignore — node_modules, .git, coverage/ all copied
#    into the image context, slowing down even cache-hit builds
```

#### .dockerignore — Exclude What Docker Doesn't Need

A missing `.dockerignore` silently breaks cache efficiency. Docker sends the entire build context to the daemon before evaluating any cache. If `node_modules/` (potentially hundreds of MB) is in the context, every build pays that transfer cost even on a cache hit.

```dockerignore
# .dockerignore

# Dependencies — rebuilt inside the image, not copied from host
node_modules/
.venv/
vendor/

# Version control
.git/
.gitignore

# CI and editor artifacts
coverage/
.nyc_output/
dist/
build/
*.log
.env
.env.*

# Documentation and tooling not needed at runtime
README.md
CHANGELOG.md
.eslintrc*
.prettierrc*
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

#### .husky/pre-commit

```bash
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
