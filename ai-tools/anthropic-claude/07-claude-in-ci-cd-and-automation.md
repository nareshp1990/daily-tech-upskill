# Tutorial 07: Claude in CI/CD and Automation

## Table of Contents

1. [Overview: Claude in Automation](#1-overview-claude-in-automation)
2. [Claude Code in GitHub Actions](#2-claude-code-in-github-actions)
3. [Automated PR Review Pipeline](#3-automated-pr-review-pipeline)
4. [Automated Documentation Generation](#4-automated-documentation-generation)
5. [Automated Test Generation](#5-automated-test-generation)
6. [Security Scanning with Claude](#6-security-scanning-with-claude)
7. [Terraform and Infrastructure Review](#7-terraform-and-infrastructure-review)
8. [Advanced Automation Patterns](#8-advanced-automation-patterns)
9. [Practical GitHub Actions Workflows](#9-practical-github-actions-workflows)
10. [Try This Exercises](#10-try-this-exercises)
11. [Tips and Gotchas](#11-tips-and-gotchas)

---

## 1. Overview: Claude in Automation

### Where Claude Fits in CI/CD

```
Developer pushes code
        │
        ▼
┌─────────────────────────────────┐
│       GitHub Actions            │
│                                 │
│  ┌─────────────┐               │
│  │ Build & Test │               │
│  └──────┬──────┘               │
│         │                       │
│  ┌──────▼──────┐               │
│  │ Claude Code  │◄── Automated: │
│  │ Analysis     │  - PR Review  │
│  │              │  - Security   │
│  │              │  - Docs Gen   │
│  │              │  - Test Gen   │
│  └──────┬──────┘               │
│         │                       │
│  ┌──────▼──────┐               │
│  │ Deploy      │               │
│  └─────────────┘               │
└─────────────────────────────────┘
```

### Key Principles

1. **Claude Code runs in headless mode** (`-p` flag) for non-interactive use
2. **Restrict permissions** with `--allowedTools` to prevent unintended changes
3. **Store API keys in GitHub Secrets** — never in code
4. **Set budget limits** — automated workflows can run up costs quickly
5. **Use Sonnet or Haiku** for CI/CD tasks — Opus is usually overkill and too expensive for automation

---

## 2. Claude Code in GitHub Actions

### Basic Setup

```yaml
# .github/workflows/claude-basics.yml
name: Claude Code Analysis

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  analyse:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for accurate diffs

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run Analysis
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "Analyse this project and list the top 5 code quality issues" \
            --allowedTools "Read,Glob,Grep" \
            --output-format text
```

### Setting Up the API Key Secret

```bash
# Add to GitHub repository secrets
gh secret set ANTHROPIC_API_KEY --body "sk-ant-api03-your-key"
```

### Model Selection for CI/CD

```yaml
# Use environment variable to control model
- name: Run Analysis
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    CLAUDE_MODEL: claude-sonnet-4-0520  # Cost-effective for automation
  run: |
    claude -p "Review this PR" \
      --model $CLAUDE_MODEL \
      --allowedTools "Read,Glob,Grep"
```

---

## 3. Automated PR Review Pipeline

### Comprehensive PR Review

```yaml
# .github/workflows/claude-pr-review.yml
name: Claude PR Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

concurrency:
  group: claude-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Get PR diff
        id: diff
        run: |
          git diff origin/${{ github.event.pull_request.base.ref }}...HEAD > /tmp/pr-diff.txt
          echo "diff_size=$(wc -c < /tmp/pr-diff.txt)" >> $GITHUB_OUTPUT

      - name: Review PR
        if: steps.diff.outputs.diff_size < 500000  # Skip very large PRs
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          cat /tmp/pr-diff.txt | claude -p "$(cat <<'PROMPT'
          You are reviewing a pull request for a Spring Boot microservice.

          Review this diff and provide feedback in the following format:

          ## Summary
          Brief description of what the PR does.

          ## Issues Found

          ### Critical (must fix before merge)
          - Issue description with file:line reference

          ### Suggestions (nice to have)
          - Improvement suggestion with file:line reference

          ## Checklist
          - [ ] No hardcoded secrets or credentials
          - [ ] Error handling is adequate
          - [ ] Input validation is present
          - [ ] No N+1 query patterns
          - [ ] Thread safety is maintained
          - [ ] Tests cover the changes
          - [ ] No breaking API changes without versioning

          ## Verdict
          APPROVE / REQUEST_CHANGES / COMMENT

          Keep the review concise and actionable. Only flag real issues.
          PROMPT
          )" \
            --allowedTools "Read,Glob,Grep" \
            --output-format text > /tmp/review.md

      - name: Post Review
        if: steps.diff.outputs.diff_size < 500000
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('/tmp/review.md', 'utf8');
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Claude Code Review\n\n${review}\n\n---\n*Automated review by Claude Code*`
            });

      - name: Skip notice for large PRs
        if: steps.diff.outputs.diff_size >= 500000
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '> Claude Code review skipped: PR diff is too large for automated review. Please request a manual review.'
            });
```

### Per-File Review (For Large PRs)

```yaml
      - name: Review changed files individually
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          CHANGED_FILES=$(git diff --name-only origin/${{ github.event.pull_request.base.ref }}...HEAD | grep '\.java$')
          REVIEW=""

          for file in $CHANGED_FILES; do
            if [ -f "$file" ]; then
              FILE_DIFF=$(git diff origin/${{ github.event.pull_request.base.ref }}...HEAD -- "$file")
              FILE_REVIEW=$(echo "$FILE_DIFF" | claude -p \
                "Review this Java file diff. List only real issues (not style nits). Format: '- **file:line**: issue'" \
                --allowedTools "Read,Glob,Grep" \
                --output-format text 2>/dev/null || echo "Review failed for $file")
              REVIEW="$REVIEW\n\n### $file\n$FILE_REVIEW"
            fi
          done

          echo -e "$REVIEW" > /tmp/review.md
```

---

## 4. Automated Documentation Generation

### API Documentation Generator

```yaml
# .github/workflows/claude-docs.yml
name: Generate API Documentation

on:
  push:
    branches: [main]
    paths:
      - 'src/main/java/**/controller/**'
      - 'src/main/java/**/api/**'

jobs:
  generate-docs:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Generate API Documentation
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "$(cat <<'PROMPT'
          Analyse all REST controllers in this Spring Boot project.

          Generate comprehensive API documentation in Markdown format:

          For each endpoint:
          - HTTP method and path
          - Description
          - Request parameters (path, query, header)
          - Request body (with JSON example)
          - Response body (with JSON example)
          - Error responses
          - Authentication requirements

          Group endpoints by controller/resource.
          Output as a single Markdown document.
          PROMPT
          )" \
            --allowedTools "Read,Glob,Grep" \
            --output-format text > docs/api-reference.md

      - name: Create PR with updated docs
        run: |
          git config user.name "Claude Code Bot"
          git config user.email "claude-bot@noreply.github.com"
          BRANCH="docs/api-update-$(date +%Y%m%d%H%M%S)"
          git checkout -b "$BRANCH"
          git add docs/api-reference.md
          git diff --cached --quiet || {
            git commit -m "Update API documentation (auto-generated)"
            git push origin "$BRANCH"
            gh pr create \
              --title "Update API documentation" \
              --body "Auto-generated API documentation from controller analysis." \
              --base main
          }
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Changelog Generation

```yaml
      - name: Generate Changelog
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          # Get commits since last tag
          LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
          if [ -n "$LAST_TAG" ]; then
            COMMITS=$(git log $LAST_TAG..HEAD --oneline)
          else
            COMMITS=$(git log --oneline -20)
          fi

          echo "$COMMITS" | claude -p "$(cat <<'PROMPT'
          Generate a CHANGELOG entry from these git commits.

          Format:
          ## [Unreleased] - YYYY-MM-DD

          ### Added
          - New features

          ### Changed
          - Changes to existing functionality

          ### Fixed
          - Bug fixes

          ### Security
          - Security improvements

          Group commits logically. Skip merge commits and CI changes.
          Write in past tense. Be concise.
          PROMPT
          )" \
            --output-format text > /tmp/changelog-entry.md
```

---

## 5. Automated Test Generation

### Generate Tests for Changed Code

```yaml
# .github/workflows/claude-test-gen.yml
name: Generate Missing Tests

on:
  workflow_dispatch:
    inputs:
      target_path:
        description: 'Path to generate tests for (e.g., src/main/java/com/example/service/)'
        required: true
        default: 'src/main/java/'

jobs:
  generate-tests:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Generate Tests
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "$(cat <<PROMPT
          Analyse the Java classes in ${{ github.event.inputs.target_path }}.

          For each class that does NOT have a corresponding test file, generate:
          1. Unit tests using JUnit 5 + Mockito + AssertJ
          2. Test all public methods
          3. Include edge cases and error scenarios
          4. Follow the existing test patterns in this project
          5. Place tests in the matching test directory structure

          Only generate tests for classes that are missing them.
          Skip interfaces, DTOs/records, and configuration classes.
          PROMPT
          )" \
            --auto-accept-edits

      - name: Run Tests
        run: |
          mvn test -pl . --fail-at-end 2>&1 | tee /tmp/test-results.txt

      - name: Fix Failing Tests
        if: failure()
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          cat /tmp/test-results.txt | claude -p \
            "These generated tests are failing. Fix the test code (not the production code) to make them pass. Common issues: wrong mock setup, incorrect assertions, missing test data." \
            --auto-accept-edits

          mvn test -pl .

      - name: Create PR
        run: |
          git config user.name "Claude Code Bot"
          git config user.email "claude-bot@noreply.github.com"
          BRANCH="test/auto-generate-$(date +%Y%m%d%H%M%S)"
          git checkout -b "$BRANCH"
          git add 'src/test/**'
          git diff --cached --quiet || {
            git commit -m "Add auto-generated unit tests"
            git push origin "$BRANCH"
            gh pr create \
              --title "Add auto-generated unit tests" \
              --body "$(cat <<'EOF'
          ## Auto-Generated Tests

          Tests generated by Claude Code for classes in `${{ github.event.inputs.target_path }}`.

          ### Review Checklist
          - [ ] Tests are meaningful (not just asserting trivially)
          - [ ] Edge cases are covered
          - [ ] Mocking is appropriate (not over-mocking)
          - [ ] Test names are descriptive
          - [ ] No flaky tests (no timing/ordering dependencies)
          EOF
          )" \
              --base main
          }
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Test Coverage Gap Analysis

```yaml
      - name: Analyse Coverage Gaps
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          # Run tests with coverage
          mvn test jacoco:report

          # Get coverage report
          claude -p "$(cat <<'PROMPT'
          Analyse the JaCoCo coverage report in target/site/jacoco/
          and the source code.

          Identify the top 10 most important uncovered code paths:
          1. Business-critical logic without tests
          2. Error handling paths that are untested
          3. Edge cases in complex methods

          For each gap, suggest what test to write.
          Format as a prioritised list with effort estimates.
          PROMPT
          )" \
            --allowedTools "Read,Glob,Grep" \
            --output-format text > /tmp/coverage-gaps.md
```

---

## 6. Security Scanning with Claude

### Security Review Workflow

```yaml
# .github/workflows/claude-security.yml
name: Claude Security Scan

on:
  pull_request:
    paths:
      - '**.java'
      - '**.yml'
      - '**.yaml'
      - '**.properties'
      - 'Dockerfile*'
      - '**.tf'

jobs:
  security-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      security-events: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Security Scan
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          git diff origin/${{ github.event.pull_request.base.ref }}...HEAD | claude -p "$(cat <<'PROMPT'
          You are a security auditor reviewing code changes for a Spring Boot application.

          Scan this diff for:

          ## OWASP Top 10 Checks
          1. **Injection** (SQL, LDAP, OS command, JPQL)
          2. **Broken Authentication** (weak session, missing auth checks)
          3. **Sensitive Data Exposure** (hardcoded secrets, PII in logs)
          4. **XXE** (XML parsing without disabling external entities)
          5. **Broken Access Control** (missing @PreAuthorize, IDOR)
          6. **Security Misconfiguration** (debug enabled, default creds)
          7. **XSS** (unescaped output in responses)
          8. **Insecure Deserialization** (untrusted ObjectInputStream)
          9. **Known Vulnerabilities** (outdated dependencies)
          10. **Insufficient Logging** (missing audit trail for sensitive operations)

          ## Additional Checks
          - Hardcoded API keys, passwords, or tokens
          - Missing CSRF protection
          - Missing rate limiting on authentication endpoints
          - Unsafe random number generation (using Math.random for security)
          - Missing input size limits
          - SQL queries using string concatenation

          Format as:
          ### [SEVERITY] Finding Title
          **File:** path/to/file.java:LINE
          **Issue:** Description
          **Fix:** How to remediate
          **Code:**
          ```suggestion
          // Fixed code
          ```

          If no issues found, state "No security issues found in this change."
          PROMPT
          )" \
            --allowedTools "Read,Glob,Grep" \
            --output-format text > /tmp/security-review.md

      - name: Post Security Review
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('/tmp/security-review.md', 'utf8');
            const hasCritical = review.includes('[CRITICAL]') || review.includes('[HIGH]');

            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Security Scan Results\n\n${review}\n\n---\n*Automated security scan by Claude Code*`
            });

            if (hasCritical) {
              core.setFailed('Critical or high security issues found. See PR comment for details.');
            }
```

### Dependency Audit

```yaml
      - name: Audit Dependencies
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          mvn dependency:tree > /tmp/deps.txt

          cat /tmp/deps.txt | claude -p "$(cat <<'PROMPT'
          Analyse this Maven dependency tree for security concerns:

          1. Known vulnerable libraries (check versions against common CVEs)
          2. Outdated major versions
          3. Libraries that have been abandoned/deprecated
          4. Unnecessary transitive dependencies that increase attack surface
          5. Duplicate libraries at different versions (dependency conflicts)

          For each issue, provide:
          - Library name and version
          - Risk level
          - Recommended action (upgrade to version X, replace with Y, exclude)
          PROMPT
          )" \
            --output-format text > /tmp/dep-audit.md
```

---

## 7. Terraform and Infrastructure Review

### Terraform Plan Review

```yaml
# .github/workflows/claude-terraform.yml
name: Claude Terraform Review

on:
  pull_request:
    paths:
      - 'infrastructure/**'
      - '**.tf'
      - '**.tfvars'

jobs:
  terraform-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.7'

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Terraform Init and Plan
        run: |
          cd infrastructure
          terraform init -backend=false
          terraform plan -no-color > /tmp/tf-plan.txt 2>&1 || true
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Review Terraform Changes
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          cat /tmp/tf-plan.txt | claude -p "$(cat <<'PROMPT'
          Review this Terraform plan for an Azure-based infrastructure.

          Check for:
          1. **Cost Impact** — Any expensive resources being created? Estimate monthly cost impact.
          2. **Security** — Public IPs, open security groups, missing encryption, public blob storage.
          3. **High Availability** — Missing availability zones, no redundancy, single points of failure.
          4. **Compliance** — Missing tags (environment, team, cost-center), naming convention violations.
          5. **Destructive Changes** — Resources being destroyed or replaced that might cause downtime.
          6. **Best Practices** — Resource sizing, network segmentation, logging enabled.

          Format as:

          ## Cost Impact
          Estimated monthly change: +$X / -$X

          ## Risk Assessment
          ### High Risk
          ### Medium Risk
          ### Low Risk

          ## Recommendations
          - Actionable suggestions

          ## Verdict
          SAFE_TO_APPLY / NEEDS_REVIEW / DANGEROUS
          PROMPT
          )" \
            --output-format text > /tmp/tf-review.md

      - name: Post Review
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('/tmp/tf-review.md', 'utf8');
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Terraform Review\n\n${review}`
            });
```

### Kubernetes Manifest Review

```yaml
      - name: Review K8s Manifests
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "$(cat <<'PROMPT'
          Review all Kubernetes manifests in the k8s/ directory.

          Check for:
          1. Missing resource limits and requests
          2. Running as root / missing security context
          3. Missing health checks (readiness, liveness probes)
          4. Missing pod disruption budgets
          5. Using latest tag instead of pinned versions
          6. Missing network policies
          7. Secrets not using external secret management
          8. Missing horizontal pod autoscaler
          9. No anti-affinity rules for high availability
          10. Missing pod security standards

          For each issue, provide the fix as a YAML snippet.
          PROMPT
          )" \
            --allowedTools "Read,Glob,Grep" \
            --output-format text
```

---

## 8. Advanced Automation Patterns

### Conditional Reviews Based on File Types

```yaml
      - name: Determine review type
        id: review-type
        run: |
          CHANGED=$(git diff --name-only origin/main...HEAD)

          echo "has_java=$(echo "$CHANGED" | grep -c '\.java$' || true)" >> $GITHUB_OUTPUT
          echo "has_terraform=$(echo "$CHANGED" | grep -c '\.tf$' || true)" >> $GITHUB_OUTPUT
          echo "has_docker=$(echo "$CHANGED" | grep -c 'Dockerfile' || true)" >> $GITHUB_OUTPUT
          echo "has_k8s=$(echo "$CHANGED" | grep -c 'k8s/' || true)" >> $GITHUB_OUTPUT

      - name: Java Code Review
        if: steps.review-type.outputs.has_java > 0
        run: # ... Java-specific review

      - name: Terraform Review
        if: steps.review-type.outputs.has_terraform > 0
        run: # ... Terraform-specific review

      - name: Docker Review
        if: steps.review-type.outputs.has_docker > 0
        run: # ... Docker-specific review
```

### Cost-Controlled Automation

```yaml
      - name: Check automation budget
        id: budget
        run: |
          # Track daily spend in a simple file (or use a proper store)
          TODAY=$(date +%Y-%m-%d)
          BUDGET_FILE="/tmp/claude-budget-$TODAY.txt"

          if [ -f "$BUDGET_FILE" ]; then
            SPENT=$(cat "$BUDGET_FILE")
          else
            SPENT=0
          fi

          DAILY_LIMIT=10  # $10 daily limit for CI/CD
          if [ "$SPENT" -ge "$DAILY_LIMIT" ]; then
            echo "skip=true" >> $GITHUB_OUTPUT
            echo "Budget exceeded for today ($SPENT / $DAILY_LIMIT USD)"
          else
            echo "skip=false" >> $GITHUB_OUTPUT
          fi

      - name: Run Claude Review
        if: steps.budget.outputs.skip != 'true'
        run: # ... review steps
```

### Caching Claude Code Installation

```yaml
      - name: Cache Claude Code
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: claude-code-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
          restore-keys: claude-code-${{ runner.os }}-
```

---

## 9. Practical GitHub Actions Workflows

### All-in-One PR Quality Gate

```yaml
# .github/workflows/pr-quality-gate.yml
name: PR Quality Gate

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  quality-gate:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
          cache: 'maven'

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Build and Test
        run: mvn verify -B

      - name: Claude Analysis
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          DIFF=$(git diff origin/${{ github.event.pull_request.base.ref }}...HEAD)

          echo "$DIFF" | claude -p "$(cat <<'PROMPT'
          Perform a comprehensive review of this PR for a Spring Boot microservice.

          Produce a single report with these sections:

          ## Code Quality
          - Logic errors, edge cases, error handling

          ## Security
          - Injection, auth, secrets, input validation

          ## Performance
          - N+1 queries, missing indexes, resource leaks, unnecessary allocations

          ## API Design
          - REST conventions, backward compatibility, proper status codes

          ## Testing
          - Missing test coverage, test quality

          ## Overall
          - Score: X/10
          - Verdict: APPROVE / REQUEST_CHANGES

          Be concise. Only flag real issues, not style preferences.
          PROMPT
          )" \
            --allowedTools "Read,Glob,Grep" \
            --output-format text > /tmp/quality-report.md

      - name: Post Report
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('/tmp/quality-report.md', 'utf8');
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## PR Quality Report\n\n${report}\n\n---\n*Automated by Claude Code*`
            });
```

### Nightly Code Health Check

```yaml
# .github/workflows/nightly-health.yml
name: Nightly Code Health

on:
  schedule:
    - cron: '0 2 * * 1-5'  # 2 AM UTC, weekdays

jobs:
  health-check:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run Health Check
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "$(cat <<'PROMPT'
          Perform a codebase health check on this Spring Boot project.

          Analyse and report on:
          1. Code smells and technical debt (top 10)
          2. Dependency freshness (outdated libraries)
          3. Configuration issues (security misconfigs, missing profiles)
          4. Test health (flaky test indicators, low coverage areas)
          5. Documentation gaps
          6. Dead code (unused classes, methods, endpoints)

          Provide a health score (A-F) for each category
          and an overall health grade.
          PROMPT
          )" \
            --allowedTools "Read,Glob,Grep" \
            --output-format text > /tmp/health-report.md

      - name: Create Issue with Report
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh issue create \
            --title "Nightly Health Check - $(date +%Y-%m-%d)" \
            --body "$(cat /tmp/health-report.md)" \
            --label "tech-debt,automated"
```

---

## 10. Try This Exercises

### Exercise 1: Basic PR Review
Set up the basic PR review workflow from Section 3 in one of your repositories. Open a test PR and see the automated review in action.

### Exercise 2: Security Scan
Implement the security scanning workflow. Intentionally introduce a vulnerability (e.g., SQL concatenation) in a test PR and verify Claude catches it.

### Exercise 3: Test Generation
Set up the manual test generation workflow. Trigger it for a specific package in your project and review the generated tests.

### Exercise 4: Terraform Review
If you have Terraform files, set up the Terraform review workflow. Create a PR that adds a new resource and see the cost and security analysis.

### Exercise 5: Custom Workflow
Create a custom GitHub Actions workflow that:
1. Triggers on push to main
2. Analyses new code for patterns specific to your team's conventions
3. Creates an issue if any violations are found

---

## 11. Tips and Gotchas

### Tips

1. **Use `--allowedTools "Read,Glob,Grep"` for review-only workflows.** This prevents Claude Code from making any changes, which is exactly what you want for CI/CD analysis.

2. **Size-gate your diffs.** Very large PRs can exceed token limits and cost a lot. Skip or split reviews for diffs over 100KB.

3. **Use concurrency groups.** Cancel outdated reviews when a new commit is pushed to the same PR. This saves costs.

4. **Cache the npm install.** Claude Code installation adds 15-30 seconds. Caching speeds up subsequent runs.

5. **Sonnet is sufficient for most CI/CD tasks.** Opus is overkill and 5x more expensive. Use Sonnet for reviews and Haiku for simple classification tasks.

6. **Post results as PR comments, not as check annotations.** Comments are more visible and easier to read.

7. **Include a "skip" label.** Let developers add a `skip-ai-review` label to bypass automated reviews when they know it is unnecessary (e.g., docs-only changes).

### Gotchas

1. **GitHub Actions runners have no persistent state.** Every run starts fresh. You cannot carry over Claude Code conversation context between runs.

2. **API key rotation requires secret updates.** If you rotate your Anthropic API key, remember to update the GitHub secret.

3. **Cost surprises.** A busy repository with many PRs can generate significant API costs. Implement budget tracking.

4. **Token limit on large repositories.** Claude Code in headless mode still has the 200K token context limit. For very large codebases, focus reviews on changed files only.

5. **Rate limits in CI/CD.** Multiple PRs triggering simultaneously share the same API rate limit. Use concurrency groups or queuing.

6. **Do not let Claude Code auto-commit to main.** Always create PRs for Claude Code's changes, never push directly.

7. **Output format matters.** Claude's output in headless mode can include markdown formatting. Make sure your comment posting step handles this correctly.

8. **GitHub token permissions.** The `GITHUB_TOKEN` needs `pull-requests: write` permission to post comments. Add this to the job's permissions block.

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Headless mode | `-p` flag for non-interactive CI/CD usage |
| Permissions | `--allowedTools "Read,Glob,Grep"` for read-only reviews |
| PR Review | Pipe diff to Claude Code, post result as PR comment |
| Security | Scan diffs against OWASP Top 10, fail the build on critical findings |
| Test Gen | Generate tests, run them, fix failures, create PR |
| Terraform | Review plans for cost, security, and best practices |
| Cost Control | Use Sonnet, size-gate diffs, implement budget limits, use concurrency groups |

**Next:** [Tutorial 08 — Daily Workflow Playbook](08-daily-workflow-playbook.md)
