# CI/CD & GitHub Actions — Daily Prompts

> Tech stack: GitHub Actions, Maven/Gradle, Docker, ACR, AKS, Terraform, SonarQube

---

## Pipeline Creation

### Full CI/CD pipeline for Spring Boot
```
Create a GitHub Actions CI/CD pipeline for my Spring Boot application.

Repository: Java 17, Maven, Spring Boot 3.x
Target: Docker → ACR → AKS (via Helm)

Workflow stages:
1. **Build & Test** (on PR and push to main):
   - Checkout, setup JDK 17
   - Maven cache (restore + save)
   - mvn verify (compile, unit tests, integration tests)
   - Upload test reports as artifacts
   - Code coverage threshold check (JaCoCo >= 80%)

2. **Code Quality** (on PR):
   - SonarQube analysis (sonar-scanner via Maven)
   - Quality gate check — fail PR if gate fails

3. **Build & Push Docker Image** (on push to main):
   - Build multi-stage Docker image
   - Tag with: git SHA, branch name, latest
   - Push to Azure Container Registry (ACR)
   - Trivy vulnerability scan on image

4. **Deploy to Dev** (on push to main, auto):
   - Helm upgrade to dev namespace on AKS
   - Smoke test: curl health endpoint
   - Notify Teams channel

5. **Deploy to Staging** (manual approval):
   - environment: staging (with reviewers)
   - Helm upgrade to staging namespace
   - Integration test suite

6. **Deploy to Prod** (manual approval):
   - environment: production (with reviewers)
   - Helm upgrade with --atomic (auto-rollback on failure)
   - Post-deploy health check

Secrets needed: ACR_LOGIN_SERVER, AZURE_CREDENTIALS, SONAR_TOKEN, KUBE_CONFIG
Show: .github/workflows/ci-cd.yml with all stages.
```

### Pipeline for Terraform
```
Create a GitHub Actions pipeline for Terraform (Azure infrastructure).

Workflow:
1. **On PR** (plan only):
   - terraform init (Azure Storage backend)
   - terraform fmt --check
   - terraform validate
   - terraform plan -out=tfplan
   - Post plan output as PR comment (using github-script)
   - No apply — just review the plan

2. **On merge to main** (apply):
   - terraform init
   - terraform apply -auto-approve (using saved plan)
   - Post apply summary to Teams

3. **Scheduled drift detection** (weekly cron):
   - terraform plan -detailed-exitcode
   - If drift detected (exit code 2) → create GitHub issue

Auth: Azure Service Principal via OIDC (federated credentials, no stored secrets)
State: Azure Storage Account with state locking

Show: .github/workflows/terraform.yml and OIDC setup instructions.
```

---

## Common Workflow Patterns

### Reusable workflow
```
Create a reusable GitHub Actions workflow for [task: Java build / Docker build+push /
Helm deploy / run tests] that can be called from multiple repositories.

.github/workflows/reusable-[task].yml:
- workflow_call trigger
- Inputs: [list parameterizable values]
- Secrets: [list required secrets]
- Outputs: [list outputs, e.g., image tag, test results]

Show the reusable workflow AND an example caller workflow using:
jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with: ...
    secrets: inherit
```

### Matrix build
```
Create a GitHub Actions matrix strategy to [run tests on multiple Java versions /
build for multiple environments / deploy to multiple clusters].

Matrix: [describe dimensions, e.g., java: [17, 21], db: [mysql-8.0, mysql-8.4]]

Requirements:
- Fail-fast: false (continue other matrix jobs if one fails)
- Max parallel: [N] to not overwhelm resources
- Exclude specific combinations if needed
- Upload artifacts per matrix combination
```

### Caching strategy
```
Optimize my GitHub Actions pipeline with proper caching.

Current build time: [X minutes], target: [Y minutes]

My stack uses:
- Maven/Gradle dependencies
- Docker layers
- Node.js (if any frontend)
- Terraform providers

For each, show:
1. The cache action configuration (actions/cache)
2. Key strategy (hash of lock file, restore-keys for partial match)
3. Expected time savings
4. Cache size management (GitHub has 10GB limit per repo)
```

---

## Deployment Strategies

### Blue-green deployment via GitHub Actions
```
Implement blue-green deployment for my AKS service via GitHub Actions.

Flow:
1. Deploy new version to "green" (inactive) deployment
2. Run smoke tests against green
3. Switch Kubernetes Service selector from "blue" to "green"
4. Verify traffic flows to new version
5. Keep old "blue" running for quick rollback
6. Clean up old version after [X] hours

Show: GitHub Actions workflow, K8s manifests (blue + green deployments),
and rollback workflow.
```

### Canary deployment
```
Implement canary deployment for [service] on AKS via GitHub Actions.

Flow:
1. Deploy canary (1 pod with new version) alongside stable (N pods)
2. Route 10% traffic to canary (via Istio/Nginx weight or pod ratio)
3. Monitor error rate and latency for [X] minutes
4. If healthy → scale up canary, scale down stable (progressive)
5. If unhealthy → auto-rollback canary
6. Final: 100% on new version, remove old

Show: workflow, K8s manifests, and monitoring check script.
```

---

## Operations

### Debug a failed pipeline
```
My GitHub Actions pipeline failed with this error:
[paste error output]

Workflow file: [paste relevant section or describe the step]

Diagnose:
1. What does this error mean?
2. Most likely cause
3. The fix (in the workflow YAML)
4. How to test locally (act CLI for local GitHub Actions testing)
5. How to prevent this: add checks, better error messages, or retry logic
```

### Manage secrets and environments
```
Set up GitHub Actions secrets and environments for my project.

Environments: dev, staging, production
Secrets needed:
- ACR credentials (shared across environments)
- AKS kubeconfig (per environment)
- Database connection strings (per environment)
- Sonar token (shared)
- Confluent Kafka credentials (per environment)

Show:
1. GitHub Environment setup with protection rules and reviewers
2. How to reference environment-specific secrets in workflow
3. OIDC federated credentials (no long-lived secrets for Azure)
4. Secret rotation strategy
5. How to audit secret access
```

### Optimize pipeline speed
```
My CI/CD pipeline takes [X minutes]. Help me optimize it.

Current steps: [list steps with approximate duration]

Optimization strategies:
1. Parallelize independent jobs
2. Cache dependencies (Maven, Docker layers)
3. Skip unchanged paths (paths-filter)
4. Smaller Docker images (alpine, distroless)
5. Conditional steps (only deploy on main, only scan on PR)
6. Self-hosted runners (faster, no queue wait)
7. Test splitting (run unit and integration tests in parallel)
8. Build-once-deploy-many (don't rebuild Docker image per environment)

Show: optimized workflow with before/after timing estimates.
```

---

## Notifications & Integrations

### Teams notification on deploy
```
Add Microsoft Teams notification to my GitHub Actions pipeline.

Notify on:
- Deployment started (which environment, who triggered, commit message)
- Deployment succeeded (with link to environment)
- Deployment failed (with link to failed run, error summary)
- PR status (build passed/failed, coverage report)

Use: Incoming Webhook connector for Teams channel.
Show: the notification step with adaptive card JSON (formatted, color-coded).
```

---

## Quick One-Liners

```
How do I trigger a GitHub Actions workflow manually (workflow_dispatch) with input parameters?
Show the workflow trigger and the Actions UI usage.
```

```
Write a GitHub Actions step that fails the pipeline if [condition] — e.g., code coverage
dropped below threshold, new TODO comments, migration file missing.
```

```
Show me how to use GitHub Actions artifacts to pass data between jobs
(e.g., test results from build job to deploy job).
```

```
Create a GitHub Actions workflow that runs on a cron schedule (e.g., nightly builds,
weekly dependency updates, scheduled smoke tests).
```

```
How do I use act CLI to test my GitHub Actions workflow locally before pushing?
Show: installation and common commands.
```

```
Write a GitHub Actions step to auto-label PRs based on files changed
(e.g., "backend" for src/main/java/**, "infra" for terraform/**).
```

---

*Fast pipelines = fast feedback = faster shipping. Invest in CI/CD like it's production code.*
