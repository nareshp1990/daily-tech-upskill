# Tutorial 4: Copilot CLI

## Table of Contents

- [Installation and Setup](#installation-and-setup)
- [gh copilot suggest](#gh-copilot-suggest)
- [gh copilot explain](#gh-copilot-explain)
- [Shell Integration and Aliases](#shell-integration-and-aliases)
- [Practical Examples: Docker](#practical-examples-docker)
- [Practical Examples: Kubernetes](#practical-examples-kubernetes)
- [Practical Examples: Git](#practical-examples-git)
- [Practical Examples: Terraform](#practical-examples-terraform)
- [Practical Examples: MySQL](#practical-examples-mysql)
- [Practical Examples: Azure CLI](#practical-examples-azure-cli)
- [Combining with gh CLI Features](#combining-with-gh-cli-features)
- [Try This Exercises](#try-this-exercises)

---

## Installation and Setup

### Prerequisites

- GitHub CLI (`gh`) version 2.0+
- Copilot subscription (Individual, Business, or Enterprise)

### Install

```bash
# Install the Copilot CLI extension
gh extension install github/gh-copilot

# Verify installation
gh copilot --version

# Upgrade to latest
gh extension upgrade gh-copilot
```

### Authenticate

```bash
# If not already logged in to gh
gh auth login

# Grant Copilot access
gh auth refresh -s copilot
```

### Verify

```bash
gh copilot suggest "list files in the current directory"
```

---

## gh copilot suggest

`gh copilot suggest` takes a natural language description and suggests a shell command.

### Basic Usage

```bash
gh copilot suggest "find all Java files modified in the last 24 hours"
```

Copilot responds with something like:
```
find . -name "*.java" -mtime -1
```

You are then prompted:
```
? Select an option:
> Copy to clipboard
  Execute command
  Revise command
  Rate response
  Exit
```

### Command Types

Copilot suggest understands three command types:

```bash
# Generic shell command
gh copilot suggest -t shell "compress all log files older than 7 days"

# Git command
gh copilot suggest -t git "undo the last 3 commits but keep the changes"

# GitHub CLI command
gh copilot suggest -t gh "list all open PRs assigned to me"
```

If you omit `-t`, Copilot infers the type from your description.

### Multi-Step Suggestions

For complex tasks, Copilot can suggest piped commands:

```bash
gh copilot suggest "find the 10 largest files in this repo excluding node_modules"
# Suggests: find . -path ./node_modules -prune -o -type f -print0 | xargs -0 du -h | sort -rh | head -10

gh copilot suggest "show disk usage of all Docker volumes sorted by size"
# Suggests: docker system df -v | grep -A 100 "Local Volumes" | sort -k3 -rh
```

---

## gh copilot explain

`gh copilot explain` takes a command and explains what it does in plain language.

### Basic Usage

```bash
gh copilot explain "kubectl get pods -n staging -o jsonpath='{.items[*].spec.containers[*].image}'"
```

Response:
```
This command lists all container images running in the "staging" namespace.
- kubectl get pods: retrieves pod resources
- -n staging: targets the "staging" namespace
- -o jsonpath='...': formats output using JSONPath
- .items[*].spec.containers[*].image: extracts the image field from all
  containers in all pods
```

### Explain Complex Pipelines

```bash
gh copilot explain "docker stats --no-stream --format '{{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}' | sort -k2 -rn | head -5"
```

```bash
gh copilot explain "awk '{sum+=$1} END {print sum/NR}' response_times.log"
```

```bash
gh copilot explain "git log --oneline --graph --all --decorate --since='2 weeks ago' --author='naresh'"
```

---

## Shell Integration and Aliases

### Setting Up Aliases

Add these to your `~/.zshrc` or `~/.bashrc`:

```bash
# Quick suggest alias — type: ?? "your question"
alias '??'='gh copilot suggest -t shell'

# Git-specific suggest
alias 'git?'='gh copilot suggest -t git'

# GitHub CLI suggest
alias 'gh?'='gh copilot suggest -t gh'

# Explain alias
alias 'explain'='gh copilot explain'
```

After adding, reload your shell:

```bash
source ~/.zshrc
```

### Usage with Aliases

```bash
?? "find all Docker containers using more than 1GB memory"

git? "squash the last 5 commits into one"

gh? "create a draft PR from current branch to main"

explain "tar -czf backup.tar.gz --exclude='*.log' /var/app/data"
```

### Advanced Shell Function

Create a function that suggests and immediately executes:

```bash
# Add to ~/.zshrc
copilot-run() {
    local cmd
    cmd=$(gh copilot suggest -t shell "$*" 2>/dev/null)
    if [ -n "$cmd" ]; then
        echo "Running: $cmd"
        eval "$cmd"
    fi
}
alias '!!'='copilot-run'
```

> **Caution**: Auto-executing commands can be dangerous. Always review first. The built-in `gh copilot suggest` workflow (with Copy/Execute/Revise options) is safer.

---

## Practical Examples: Docker

### Container Management

```bash
?? "list all running containers with their ports and image names"
# docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}"

?? "stop and remove all containers that exited more than 24 hours ago"
# docker container prune --filter "until=24h"

?? "show real-time logs for the order-service container, last 100 lines"
# docker logs -f --tail 100 order-service

?? "copy a file from inside a running container to the host"
# docker cp order-service:/app/logs/error.log ./error.log
```

### Image Management

```bash
?? "list all Docker images sorted by size, show only repo, tag, and size"
# docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | sort -k3 -rh

?? "remove all dangling images and unused build cache"
# docker system prune -a --volumes

?? "build a Docker image tagged with the current git commit hash"
# docker build -t myapp:$(git rev-parse --short HEAD) .
```

### Docker Compose

```bash
?? "start only MySQL and Kafka from docker-compose in detached mode"
# docker compose up -d mysql kafka

?? "rebuild and restart only the order-service without affecting other services"
# docker compose up -d --build --force-recreate order-service

?? "show logs from all services but filter only ERROR lines"
# docker compose logs -f | grep -i error
```

### Resource Monitoring

```bash
?? "show CPU and memory usage of all containers in real-time"
# docker stats

?? "check total disk space used by Docker"
# docker system df

?? "find which container is using the most disk space"
# docker system df -v
```

---

## Practical Examples: Kubernetes

### Pod Operations

```bash
?? "list all pods in the staging namespace that are not in Running state"
# kubectl get pods -n staging --field-selector=status.phase!=Running

?? "get the logs from the previous crashed instance of order-service pod"
# kubectl logs -n production -p deployment/order-service

?? "exec into the order-service pod and open a shell"
# kubectl exec -it -n production deployment/order-service -- /bin/sh

?? "show resource usage (CPU/memory) for all pods in production"
# kubectl top pods -n production --sort-by=memory
```

### Debugging

```bash
?? "describe the pod that is in CrashLoopBackOff in staging"
# kubectl get pods -n staging | grep CrashLoopBackOff | awk '{print $1}' | xargs kubectl describe pod -n staging

?? "show events in production namespace sorted by time"
# kubectl get events -n production --sort-by='.lastTimestamp'

?? "check why a deployment rollout is stuck"
# kubectl rollout status deployment/order-service -n production

?? "get all pods and their node assignments"
# kubectl get pods -o wide -n production
```

### Deployments and Scaling

```bash
?? "scale the order-service deployment to 5 replicas in production"
# kubectl scale deployment order-service -n production --replicas=5

?? "rollback the last deployment of order-service"
# kubectl rollout undo deployment/order-service -n production

?? "show deployment history for order-service"
# kubectl rollout history deployment/order-service -n production

?? "set a new image for the order-service deployment"
# kubectl set image deployment/order-service order-service=myregistry.azurecr.io/order-service:v2.1.0 -n production
```

### ConfigMaps and Secrets

```bash
?? "create a configmap from a properties file"
# kubectl create configmap app-config --from-file=application.properties -n production

?? "decode a secret value in Kubernetes"
# kubectl get secret db-credentials -n production -o jsonpath='{.data.password}' | base64 -d

?? "edit a configmap in place"
# kubectl edit configmap app-config -n production
```

---

## Practical Examples: Git

### Branch Management

```bash
git? "list all branches merged into main that can be safely deleted"
# git branch --merged main | grep -v "main" | grep -v "develop"

git? "delete all local branches that have been merged into main"
# git branch --merged main | grep -v "main" | xargs git branch -d

git? "find which branch a specific commit came from"
# git branch --contains <commit-hash>
```

### History and Blame

```bash
git? "show all commits that changed OrderService.java in the last month"
# git log --since="1 month ago" --follow -- src/main/java/**/OrderService.java

git? "find who last modified each line in a file"
# git blame src/main/java/com/example/service/OrderServiceImpl.java

git? "show the diff between what's on main and my current branch"
# git diff main...HEAD

git? "list all files changed between two tags"
# git diff --name-only v1.2.0..v1.3.0
```

### Undoing and Fixing

```bash
git? "undo the last commit but keep all changes staged"
# git reset --soft HEAD~1

git? "remove a file from the last commit without losing it"
# git reset HEAD~1 -- path/to/file && git commit --amend

git? "find a commit that introduced a bug between v1.0 and v1.5"
# git bisect start v1.5 v1.0
```

### Stash Operations

```bash
git? "stash only specific files"
# git stash push -m "WIP: order validation" -- src/main/java/com/example/service/OrderService.java

git? "apply a specific stash without removing it from the stash list"
# git stash apply stash@{2}

git? "show what's in a stash without applying it"
# git stash show -p stash@{0}
```

---

## Practical Examples: Terraform

### State Management

```bash
?? "list all resources in the Terraform state"
# terraform state list

?? "show the details of a specific resource in state"
# terraform state show azurerm_kubernetes_cluster.main

?? "remove a resource from state without destroying it"
# terraform state rm azurerm_kubernetes_cluster.main

?? "move a resource to a different state path after refactoring"
# terraform state mv azurerm_resource_group.old azurerm_resource_group.new
```

### Planning and Applying

```bash
?? "create a Terraform plan targeting only the AKS cluster"
# terraform plan -target=azurerm_kubernetes_cluster.main

?? "apply Terraform changes with auto-approve for CI pipeline"
# terraform apply -auto-approve

?? "destroy only the staging environment resources"
# terraform destroy -target=module.staging

?? "show what Terraform would change without applying"
# terraform plan -out=tfplan && terraform show -json tfplan | jq '.resource_changes[] | {address, actions: .change.actions}'
```

### Workspace Management

```bash
?? "list all Terraform workspaces"
# terraform workspace list

?? "create and switch to a new workspace called staging"
# terraform workspace new staging

?? "switch to the production workspace"
# terraform workspace select production
```

---

## Practical Examples: MySQL

```bash
?? "connect to MySQL running in Docker on port 3307"
# mysql -h 127.0.0.1 -P 3307 -u root -p

?? "dump a MySQL database excluding certain tables"
# mysqldump -u root -p mydb --ignore-table=mydb.audit_logs --ignore-table=mydb.sessions > backup.sql

?? "import a SQL dump file into a database"
# mysql -u root -p mydb < backup.sql

?? "show all MySQL processes and kill long-running queries"
# mysql -e "SHOW PROCESSLIST" | grep -v "Sleep" | awk '$6 > 30 {print $1}' | xargs -I{} mysql -e "KILL {}"

?? "check the size of all tables in a database sorted by size"
# mysql -e "SELECT table_name, ROUND(data_length/1024/1024, 2) as size_mb FROM information_schema.tables WHERE table_schema='mydb' ORDER BY data_length DESC"
```

---

## Practical Examples: Azure CLI

```bash
?? "list all AKS clusters in my subscription"
# az aks list --output table

?? "get credentials for the production AKS cluster"
# az aks get-credentials --resource-group production-rg --name production-aks

?? "check the status of an Azure Container Registry build"
# az acr build --registry myacr --image myapp:latest .

?? "create an Azure Key Vault secret"
# az keyvault secret set --vault-name my-keyvault --name db-password --value "secret123"

?? "list all resources in a resource group with their costs"
# az resource list --resource-group production-rg --output table

?? "scale an AKS node pool"
# az aks nodepool scale --resource-group production-rg --cluster-name production-aks --name apppool --node-count 5
```

---

## Combining with gh CLI Features

Copilot CLI works alongside all other `gh` commands.

### Issue Management

```bash
gh? "list all open issues assigned to me with the bug label"
# gh issue list --assignee @me --label bug --state open

gh? "create an issue from the command line with a template"
# gh issue create --title "Fix order processing timeout" --body "..." --label bug --assignee @me
```

### Pull Request Workflows

```bash
gh? "list all PRs that need my review"
# gh pr list --search "review-requested:@me"

gh? "check out a PR locally for testing"
# gh pr checkout 123

gh? "merge a PR with squash and delete the branch"
# gh pr merge 123 --squash --delete-branch

gh? "view the CI status of the current PR"
# gh pr checks
```

### Actions and Workflows

```bash
gh? "list recent workflow runs for the CI pipeline"
# gh run list --workflow=ci.yml --limit 10

gh? "view the logs of a failed workflow run"
# gh run view 12345 --log-failed

gh? "re-run a failed workflow"
# gh run rerun 12345 --failed

gh? "trigger a workflow manually with parameters"
# gh workflow run deploy.yml -f environment=staging -f version=v1.2.3
```

### Releases

```bash
gh? "create a release with auto-generated notes from merged PRs"
# gh release create v1.3.0 --generate-notes

gh? "list all releases"
# gh release list

gh? "download assets from a release"
# gh release download v1.3.0 --pattern "*.jar"
```

---

## Try This Exercises

### Exercise 1: Set Up Aliases

1. Add the `??`, `git?`, `gh?`, and `explain` aliases to your `~/.zshrc`
2. Reload your shell
3. Try: `?? "find all YAML files in the current directory recursively"`

### Exercise 2: Docker Discovery

1. Use `??` to find a command that shows all Docker networks and their connected containers
2. Use `explain` on the suggested command to understand each flag
3. Execute the command

### Exercise 3: Kubernetes Debugging

1. Use `??` to find a command to get all pods in CrashLoopBackOff across all namespaces
2. Use `??` to find a command to check the OOMKilled events
3. Use `explain` on both commands

### Exercise 4: Git Archaeology

1. Use `git?` to find who introduced a specific line in a file
2. Use `git?` to find all commits between two releases that touched a specific directory
3. Use `explain` on the `git log` command Copilot suggests

### Exercise 5: Terraform State

1. Use `??` to get a command that shows all resources in Terraform state as a tree
2. Use `??` to find a command that compares two Terraform state files
3. Execute on your actual Terraform project

---

## Next Steps

Now that you can use Copilot from the command line, learn how to use [Tutorial 5: Copilot for Pull Requests](05-copilot-for-pull-requests.md) to automate your PR workflow.
