# Terraform & IaC — Daily Prompts

> Tech stack: Terraform, Azure Provider (azurerm), Azure Backend (Storage Account), GitHub Actions

---

## Resource Provisioning

### Provision a new Azure resource
```
Write Terraform configuration to provision [resource, e.g., Azure MySQL Flexible Server /
AKS cluster / Blob Storage / Key Vault / Redis Cache].

Requirements:
- Use azurerm provider (latest stable)
- Variables for: resource_group, location, environment, tags
- Naming convention: [company]-[env]-[region]-[resource_type]
- Tags: environment, team, cost_center, managed_by=terraform
- Network: Private Endpoint in existing VNet/Subnet (reference via data source)
- Diagnostics: send logs to existing Log Analytics workspace
- Output: resource ID, connection string (sensitive), FQDN

Structure:
- main.tf — resources
- variables.tf — input variables with descriptions and validations
- outputs.tf — outputs
- terraform.tfvars — environment-specific values (gitignored)
- versions.tf — required providers and terraform version
```

### Create a reusable module
```
Create a reusable Terraform module for [resource type] that I can use across
multiple environments (dev, staging, prod).

Module structure:
modules/[resource]/
  ├── main.tf
  ├── variables.tf (with type, description, default, validation)
  ├── outputs.tf
  └── README.md (auto-generated format)

The module should:
- Accept environment-specific inputs via variables
- Use conditional resources (count or for_each) for optional features
- Include sensible defaults for non-critical settings
- Follow Terraform module best practices (no hard-coded values, no providers)

Show: module code AND example usage in the root module calling it.
```

### Multi-environment setup
```
Set up Terraform for managing multiple environments (dev, staging, prod) on Azure.

Approach comparison — recommend the best for my team:
1. Workspaces (terraform workspace)
2. Directory-per-environment (envs/dev/, envs/prod/)
3. Terragrunt
4. tfvars per environment

My preference: [choose one or ask for recommendation]

Show:
- Directory structure
- Backend config per environment (separate state files in Azure Storage)
- Variable definitions with environment-specific values
- How to plan/apply for a specific environment
- CI/CD integration (GitHub Actions workflow per environment)
```

---

## State Management

### Fix state issues
```
I have a Terraform state issue: [describe — e.g., resource exists in Azure but not
in state / resource is in state but was deleted / state lock stuck / state corruption].

Walk me through:
1. Diagnose: terraform state list, terraform state show [resource]
2. Fix:
   - terraform import [resource_address] [azure_resource_id]
   - terraform state rm [resource_address]
   - terraform force-unlock [lock-id]
   - terraform state pull / terraform state push (for corruption)
3. Verify: terraform plan shows no changes
4. Prevention: how to avoid this in the future

Show the exact commands for my scenario.
```

### Migrate state backend
```
Migrate Terraform state from [local / old backend] to Azure Storage Account backend.

Show:
1. Create Azure Storage Account for state (with versioning, lock, encryption)
2. Backend configuration block
3. terraform init -migrate-state command
4. Verify state was migrated
5. Lock cleanup if needed
6. Terraform code to provision the state storage account (chicken-and-egg solution)
```

---

## Day-to-Day Operations

### Review a Terraform plan
```
Review this Terraform plan output and explain what will happen:
[paste terraform plan output]

For each change:
1. Is this expected for [my intended change]?
2. Any destructive actions (destroy + recreate)?
3. Will this cause downtime?
4. Are there any concerning force-replacement triggers (tainted, ~)?
5. Cost impact of the changes
6. Should I proceed or adjust?
```

### Import existing resource
```
Import this existing Azure resource into Terraform state:

Resource type: [e.g., azurerm_kubernetes_cluster]
Azure Resource ID: [/subscriptions/.../resourceGroups/.../...]

Show:
1. Write the Terraform resource block matching the current Azure config
   (use az CLI to inspect current settings)
2. terraform import command
3. Run terraform plan — expect zero changes
4. If plan shows diffs, adjust the resource block to match reality
5. Common gotchas for this resource type during import
```

### Refactor without destroying
```
I need to refactor my Terraform code: [describe — e.g., rename resource / move to module /
restructure files / rename variable].

The resource MUST NOT be destroyed and recreated.

Show:
1. terraform state mv [old_address] [new_address] for each resource
2. Verify with terraform plan (should show no changes)
3. If moving to a module: state mv from root to module.name.resource
4. All state mv commands needed in order
5. A script that runs all moves atomically
```

---

## Security & Best Practices

### Secure Terraform setup
```
Audit my Terraform setup for security best practices:

Current setup: [describe — e.g., state in Azure Storage, secrets in tfvars, etc.]

Check:
1. State security: encrypted at rest? access-controlled? versioned?
2. Secrets: are any in plain text? Should use Key Vault data source or SOPS
3. Provider auth: Managed Identity vs service principal vs personal credentials?
4. Least privilege: does the SP/MI have more permissions than needed?
5. Drift detection: am I catching manual changes?
6. Policy: should I use Azure Policy or Sentinel for compliance?
7. .gitignore: tfstate, tfvars, .terraform/ excluded?

Show: recommended fixes for each finding.
```

---

## Quick One-Liners

```
Write a Terraform variable with validation that [only accepts certain values /
matches a regex / is within a range]. Show the validation block.
```

```
How do I reference an existing resource (not managed by Terraform) using a data source?
Resource: [type and identifying info]
```

```
Show me how to use for_each to create [N] instances of [resource] from a map variable.
```

```
My terraform plan shows a force-replacement on [resource] due to [attribute change].
How do I avoid the destroy+recreate? Can I use lifecycle { ignore_changes }?
```

```
How do I conditionally create a resource in Terraform? Show: count approach and
for_each approach with a boolean variable.
```

```
Write a terraform fmt check and terraform validate step for my CI pipeline.
```

---

*Terraform: plan before apply, always. Review the plan like a PR — every change matters in infra.*
