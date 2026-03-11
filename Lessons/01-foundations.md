# Level 1 — Foundations

## 1.1 The SDLC and Where CI Fits

### The Traditional Problem

Before CI, teams worked in isolation for days or weeks, then attempted to merge everything at once — a painful process known as **"integration hell"**.

```text
Developer A (2 weeks of work)  ──┐
Developer B (2 weeks of work)  ──┼──► MERGE ──► 💥 Conflicts everywhere
Developer C (2 weeks of work)  ──┘
```

### The CI Solution

Integrate continuously — every change triggers an automated pipeline:

```text
Developer A (1 commit)  ──► Pipeline ──► ✅ Merged
Developer B (1 commit)  ──► Pipeline ──► ✅ Merged
Developer C (1 commit)  ──► Pipeline ──► ❌ Tests fail → Fix immediately
```

### The SDLC with CI

```text
┌─────────┐   ┌──────┐   ┌───────┐   ┌──────┐   ┌─────────┐   ┌────────┐
│  PLAN   │──►│ CODE │──►│ BUILD │──►│ TEST │──►│ RELEASE │──►│ DEPLOY │
└─────────┘   └──────┘   └───────┘   └──────┘   └─────────┘   └────────┘
                 │              ▲
                 └───── CI ─────┘  (automated feedback on every commit)
```

| Stage   | Without CI                  | With CI                                   |
|---------|-----------------------------|-------------------------------------------|
| Code    | Manual reviews              | Automated linting, type checking          |
| Build   | "Works on my machine"       | Reproducible builds in clean environments |
| Test    | Run manually, often skipped | Mandatory, automated on every commit      |
| Release | Manual, error-prone         | Automated versioning and changelogs       |

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

```text
main ──────────────────────────────────────────────► (production tags)
  └── develop ─────────────────────────────────────► (integration)
        ├── feature/login ──────────► merge to develop
        ├── feature/checkout ───────► merge to develop
        └── release/1.2.0 ──────────► merge to main + develop
              └── hotfix/critical ──► merge to main + develop
```

#### GitHub Flow — continuous web delivery

```text
main ─────────────────────────────────────────────► (always deployable)
  ├── feature/new-api ──► PR ──► merge ──► deploy
  └── fix/auth-bug ─────► PR ──► merge ──► deploy
```

#### Trunk-Based Development — recommended for CI

The model used by Google, Meta, and most high-performing engineering teams. Everyone integrates into `main` continuously, keeping the trunk always in a deployable state.

```text
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

|                     | Git Flow        | GitHub Flow | Trunk-Based Dev |
|---------------------|-----------------|-------------|-----------------|
| Branch lifetime     | Days–weeks      | Hours–days  | Hours (< 1 day) |
| Merge conflicts     | Large, painful  | Moderate    | Minimal         |
| CI feedback speed   | Delayed         | Fast        | Immediate       |
| Integration risk    | High at release | Low         | Very low        |
| Feature flag needed | No              | Rarely      | Yes             |

**What a typical TBD day looks like:**

```text
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

```text
Merge (flag off) ──► QA testing (flag on for testers) ──► Gradual rollout ──► 100% ──► Remove flag
      day 1                  day 2–3                          day 4–5           day 6      day 7+
```

**When NOT to use TBD:**

- Open source projects with external contributors who submit infrequent large PRs
- Teams without a fast CI pipeline (if CI takes 30+ minutes, short-lived branches become impractical)
- Projects with formal release gating (regulated industries, firmware) — Git Flow is more appropriate

### Conventional Commits

A machine-readable commit format that enables automated changelogs and semantic versioning.

```text
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

| Type       | Meaning                 | Version Bump  |
|------------|-------------------------|---------------|
| `feat`     | New feature             | Minor (1.x.0) |
| `fix`      | Bug fix                 | Patch (1.0.x) |
| `feat!`    | Breaking change         | Major (x.0.0) |
| `docs`     | Documentation only      | None          |
| `chore`    | Maintenance tasks       | None          |
| `test`     | Adding or fixing tests  | None          |
| `ci`       | CI config changes       | None          |
| `perf`     | Performance improvement | Patch         |
| `refactor` | Code restructure        | None          |

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

```text
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

```text
Repository → Settings → Branches → Add branch protection rule
  Pattern: main   (supports wildcards: release/*, v*.*)
```

The key options and what each one actually does:

```text
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

#### ⚠️ Checks Only Appear After the First Pipeline Run

This is one of the most common points of confusion when setting up branch protection for the first time. GitHub does **not** know which checks exist until the pipeline has actually run at least once against your repository. Until then, the search box under "Status checks that are required" returns no results — not because the configuration is wrong, but because GitHub has never seen those check names before.

```text
First time setup order:

  Step 1 — Push your CI workflow file (.github/workflows/ci.yml)
            ↓
  Step 2 — Let it run once (on any branch or a test PR)
            ↓
  Step 3 — Go to Settings → Branches → Add rule → main
            Type the check name in the search box
            ↓ now it appears as an autocomplete suggestion
  Step 4 — Select and save
```

**What the UI looks like before vs after the first run:**

```text
Before first run:
  Status checks that are required
  ┌─────────────────────────────────────────┐
  │ Search for status checks...             │
  └─────────────────────────────────────────┘
  No results found for "CI / lint"          ← frustrating, but expected

After first run:
  Status checks that are required
  ┌─────────────────────────────────────────┐
  │ CI /                                    │
  └─────────────────────────────────────────┘
  ✓ CI / build-and-test                     ← now selectable
  ✓ CI / lint
  ✓ CI / security-scan
```

**Practical bootstrap sequence for a new repository:**

```bash
# 1. Add your workflow file and push to a feature branch
git checkout -b ci/setup
mkdir -p .github/workflows
# ... create ci.yml ...
git add .github/workflows/ci.yml
git commit -m "ci: add initial CI pipeline"
git push origin ci/setup

# 2. Open a PR — this triggers the pipeline for the first time
#    Wait for all jobs to complete (green or red doesn't matter,
#    GitHub just needs to register the check names)

# 3. Now go to Settings → Branches → Add rule
#    The check names will appear in the autocomplete search

# 4. Merge this PR — branch protection is now active for all future PRs
```

> **Tip:** If you rename a job in your workflow YAML later (e.g., `lint` → `lint-and-format`), GitHub will stop receiving reports for the old name. The branch protection rule will reference a check that no longer exists, blocking all PRs. Always update the required checks in Settings immediately after renaming a job, or run the pipeline once so the new name appears in the search before saving the rule.

**What happens when a required check never runs:**

```text
Scenario: You added "CI / lint" as required, but the workflow only
          triggers on pull_request, and someone pushes directly to
          a non-protected branch.

Status shown on PR: "Expected — Waiting for status to be reported"
Result: PR is permanently blocked — the check never fires.

This is intentional and safe. It forces you to ensure your workflow
trigger covers the protected branch context. Common fix:

  on:
    push:
      branches: [main]
    pull_request:         ← this is the trigger that populates the check
      branches: [main]    ← on PRs targeting main
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

```text
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
