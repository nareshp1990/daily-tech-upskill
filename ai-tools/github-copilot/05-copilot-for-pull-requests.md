# Tutorial 5: Copilot for Pull Requests

## Table of Contents

- [Overview of Copilot PR Features](#overview-of-copilot-pr-features)
- [Copilot PR Summaries](#copilot-pr-summaries)
- [Copilot Code Review](#copilot-code-review)
- [Copilot Workspace](#copilot-workspace)
- [Custom Review Instructions](#custom-review-instructions)
- [Setting Up in Repository Settings](#setting-up-in-repository-settings)
- [Practical Workflow: Issue to Merged PR](#practical-workflow-issue-to-merged-pr)
- [Try This Exercises](#try-this-exercises)

---

## Overview of Copilot PR Features

Copilot integrates into the GitHub pull request workflow at every stage:

| Feature | What It Does | Plan Required |
|---------|-------------|---------------|
| **PR Summaries** | Auto-generates PR description from diffs | Business+ |
| **Code Review** | Automated review comments on PR diffs | Business+ |
| **Copilot Workspace** | Issue → Plan → Implementation → PR | Enterprise |
| **PR Chat** | Ask questions about the PR in a chat sidebar | All plans |
| **Commit Message Suggestions** | Suggests commit messages based on staged changes | All plans |

---

## Copilot PR Summaries

### What It Does

When you create a PR, Copilot can automatically generate a description that summarizes:
- What files were changed
- What the changes do (business logic perspective)
- Key modifications and their purpose

### How to Use

#### Option 1: Auto-Generate on PR Creation

1. Push your branch and open a new PR on GitHub.com
2. In the PR description box, click the Copilot icon (sparkle icon)
3. Copilot generates a summary based on the diff
4. Review and edit before creating the PR

#### Option 2: Use the Copilot Tag

In the PR description template, add:

```markdown
copilot:summary
```

Copilot replaces this tag with a generated summary when the PR is created.

#### Option 3: From the CLI

```bash
# Create PR and let Copilot generate the body
gh pr create --title "Add order validation" --body "copilot:summary"
```

### Customizing PR Summary Format

Create `.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Summary
copilot:summary

## Changes
copilot:walkthrough

## Checklist
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Documentation updated
- [ ] No secrets or credentials committed
- [ ] Database migrations are backward-compatible
- [ ] Kafka schema changes are backward-compatible
```

### What a Generated Summary Looks Like

For a PR that adds a new order cancellation feature:

```markdown
## Summary

This PR adds order cancellation functionality to the order service.

### Changes

- **OrderController.java**: Added `DELETE /api/v1/orders/{id}` endpoint
  that accepts a cancellation reason
- **OrderService.java / OrderServiceImpl.java**: New `cancelOrder` method
  that validates cancellation eligibility (only PENDING or CONFIRMED orders),
  updates status to CANCELLED, and publishes an `OrderCancelled` event to Kafka
- **Order.java**: Added `cancellationReason` and `cancelledAt` fields
- **V005__add_cancellation_fields.sql**: Flyway migration adding columns
  to the orders table
- **OrderServiceImplTest.java**: Tests for cancellation happy path,
  already-shipped rejection, and Kafka event verification
```

### Tips for Better Summaries

- Make small, focused PRs — summaries are more accurate for targeted changes
- Use meaningful commit messages — Copilot reads them for context
- Keep PRs under 500 lines of diff — large PRs produce vague summaries

---

## Copilot Code Review

### What It Does

Copilot can review your PR code and leave comments identifying:
- Potential bugs
- Security vulnerabilities
- Performance issues
- Code style inconsistencies
- Missing error handling
- Suggested improvements

### How to Request a Review

#### Option 1: Add Copilot as a Reviewer

1. Open your PR on GitHub.com
2. In the right sidebar, click "Reviewers"
3. Select "Copilot" from the list
4. Copilot reviews the diff and leaves inline comments

#### Option 2: Automatic on PR Creation

Configure in repository settings (see [Setting Up](#setting-up-in-repository-settings)) to automatically request Copilot review on all PRs.

#### Option 3: From the CLI

```bash
# Request Copilot review on an existing PR
gh pr edit 123 --add-reviewer "copilot"
```

### What Copilot Review Comments Look Like

On a PR that adds a new REST endpoint:

**Comment on Controller (Potential Issue)**:
```
💡 Suggestion: The `@PathVariable` `id` is not validated. Consider adding
`@Min(1)` to prevent negative IDs from hitting the database.

```java
// Before
public ResponseEntity<OrderResponse> getOrder(@PathVariable Long id) {

// Suggested
public ResponseEntity<OrderResponse> getOrder(@PathVariable @Min(1) Long id) {
```
```

**Comment on Service (Security)**:
```
⚠️ Security: The `deleteOrder` method does not verify that the requesting
user owns the order. This could allow any authenticated user to delete
any order. Consider adding an ownership check.
```

**Comment on SQL Migration (Performance)**:
```
💡 Performance: The new query on `orders.customer_id` would benefit from
an index. Consider adding:
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

### Responding to Copilot Comments

- **Accept**: Click "Resolve" if you agree and will implement the change
- **Dismiss**: Click "Dismiss" with a reason if the suggestion does not apply
- **Discuss**: Reply to the comment to ask for clarification or provide context

### Copilot Review Quality

Copilot reviews are strongest at:
- Null pointer / null safety issues
- Missing validation
- SQL injection risks
- Resource leaks (unclosed streams, connections)
- Obvious concurrency bugs
- Missing error handling

Copilot reviews are weaker at:
- Business logic correctness
- Architectural decisions
- Performance of complex algorithms
- Domain-specific rules

> **Best practice**: Use Copilot as a first-pass reviewer. It catches mechanical issues, freeing human reviewers to focus on architecture and business logic.

---

## Copilot Workspace

### What It Is

Copilot Workspace (Enterprise plan) is an end-to-end development environment that takes you from an issue to a complete PR:

1. **Specification**: Copilot reads the issue and creates a detailed spec
2. **Plan**: Copilot proposes which files to create/modify
3. **Implementation**: Copilot writes the code changes
4. **Validation**: You review, edit, and test
5. **PR Creation**: Copilot creates a PR with the changes

### How to Use

1. Open any GitHub issue
2. Click "Open in Workspace" button
3. Copilot shows a specification — edit if needed
4. Copilot shows a plan (files to modify) — adjust if needed
5. Click "Implement" — Copilot generates code for each file
6. Review the generated code in a VS Code-like editor
7. Run tests using integrated terminal
8. Click "Create Pull Request"

### Example: Implementing an Issue

**Issue**: "Add rate limiting to the order creation endpoint. Limit to 10 orders per minute per user. Return 429 when exceeded."

**Copilot Workspace generates**:

Specification:
```
- Add rate limiting using Spring Boot's Bucket4j or a custom RateLimiter
- Apply to POST /api/v1/orders endpoint only
- Limit: 10 requests per minute per authenticated user (by user ID)
- Return HTTP 429 Too Many Requests with Retry-After header
- Store rate limit state in Redis for distributed deployment
- Add configuration properties for rate limit values
```

Plan:
```
1. Create: RateLimitFilter.java (Servlet filter for rate limiting)
2. Create: RateLimitConfig.java (Configuration for Redis-backed rate limiter)
3. Modify: application.yml (Add rate limit properties)
4. Modify: OrderController.java (Add rate limit annotation)
5. Create: RateLimitExceededException.java (Custom exception)
6. Modify: GlobalExceptionHandler.java (Handle 429 response)
7. Create: RateLimitFilterTest.java (Unit tests)
8. Create: docker-compose.yml (Add Redis service if not present)
```

You can add/remove files from the plan, then click Implement.

### When to Use Copilot Workspace

- Well-defined issues with clear requirements
- Feature additions that touch multiple files
- Bug fixes where the issue clearly describes the problem
- Routine CRUD endpoint additions

### When NOT to Use Copilot Workspace

- Complex architectural changes
- Sensitive security code
- Performance-critical algorithms
- Changes requiring deep domain knowledge

---

## Custom Review Instructions

### Repository-Level Review Instructions

Create `.github/copilot-review-instructions.md`:

```markdown
# Code Review Instructions

## General Rules
- Flag any use of field injection (@Autowired on fields) — we use constructor injection only
- Flag any method longer than 50 lines
- Flag any class with more than 5 dependencies (constructor parameters)
- Ensure all public API endpoints have @Valid on request body parameters
- Check that all database queries use parameterized values (no string concatenation)

## Spring Boot Specific
- Controllers should not contain business logic — delegate to services
- Services should be transactional where appropriate
- Repository methods should use Spring Data naming conventions when possible
- Custom @Query annotations must have corresponding tests

## Kafka Specific
- All consumers must be idempotent
- Consumer methods must handle exceptions and not let them propagate uncaught
- Check for proper acknowledgment handling

## Security
- No secrets or credentials in code (even in comments or TODOs)
- All user input must be validated
- SQL queries must use prepared statements
- Sensitive data (PII) must not be logged

## Testing
- New features must include unit tests
- Service tests must mock external dependencies
- Integration tests should use Testcontainers
- Test method names should describe the scenario (given_when_then)

## Database Migrations
- Migrations must be backward-compatible (no column renames or drops without migration strategy)
- All new columns should have default values or be nullable
- Add indexes for foreign key columns
```

### How It Works

When Copilot reviews a PR in your repository, it reads these instructions and applies them in addition to its default review behavior. Your custom rules get higher priority.

---

## Setting Up in Repository Settings

### Enable Copilot Features

1. Go to your repository on GitHub.com
2. Click **Settings** → **Copilot** (in the left sidebar under "Code and automation")

### Configure Auto-Review

```
Settings → Copilot → Code Review:
  ✅ Automatically request Copilot review on new PRs

  Review scope:
  ○ All files
  ● Only changed files (recommended)

  Severity threshold:
  ✅ Critical (bugs, security)
  ✅ Warning (performance, best practices)
  ☐ Info (style, suggestions)
```

### Configure PR Summaries

```
Settings → Copilot → Pull Request Summaries:
  ✅ Auto-generate summaries for new PRs

  Summary style:
  ● Detailed (file-by-file walkthrough)
  ○ Concise (bullet point summary)
  ○ Changelog (formatted for release notes)
```

### Organization-Level Settings (Business/Enterprise)

Organization admins can set defaults:

```
Organization Settings → Copilot → Policies:
  ✅ Allow Copilot code review for all repositories
  ✅ Allow Copilot PR summaries for all repositories
  ✅ Allow Copilot Workspace for all repositories

Content Exclusions:
  - **/secrets/**
  - **/.env*
  - **/credentials*
```

---

## Practical Workflow: Issue to Merged PR

Here is a complete workflow using Copilot at every stage.

### Step 1: Triage the Issue

Open the issue on GitHub. Use Copilot Chat on GitHub.com:

```
What would be the best approach to implement this feature?
What files in this repo would need to change?
```

### Step 2: Create a Branch

```bash
git checkout -b feature/order-cancellation
```

### Step 3: Implement with Copilot in IDE

Open the relevant files in your IDE. Use completions and chat:

```
# In Copilot Chat:
@workspace I need to add an order cancellation feature.
The endpoint should be DELETE /api/v1/orders/{id} with a cancellation reason in the body.
Only PENDING and CONFIRMED orders can be cancelled.
Generate the controller method, service method, and entity changes.
```

### Step 4: Generate Tests with Copilot

```
/tests Generate tests for the cancelOrder method in OrderServiceImpl.
Cover: successful cancellation, already shipped (should reject),
already cancelled (should reject), order not found.
```

### Step 5: Commit with Copilot-Suggested Message

In VS Code, stage your changes. Copilot suggests a commit message in the Source Control panel:

```
feat: add order cancellation with eligibility validation

- Add DELETE /api/v1/orders/{id} endpoint
- Validate order status before cancellation
- Publish OrderCancelled event to Kafka
- Add Flyway migration for cancellation fields
```

### Step 6: Create PR with Copilot Summary

```bash
gh pr create --title "Add order cancellation feature" --body "copilot:summary"
```

Or create the PR on GitHub.com and click the Copilot icon to generate the description.

### Step 7: Copilot Reviews the PR

Copilot automatically reviews and leaves comments:
- "Consider adding `@Transactional` to the cancel method"
- "The cancellation reason should be validated for max length"
- "Add an index on the `status` column for the new query"

### Step 8: Address Review Comments

Fix the issues Copilot found, push updates. Copilot re-reviews the new changes.

### Step 9: Human Review

Your teammate reviews the PR, focusing on:
- Business logic correctness (Copilot already checked mechanical issues)
- Architecture alignment
- Edge cases specific to your domain

### Step 10: Merge

```bash
gh pr merge --squash --delete-branch
```

---

## Try This Exercises

### Exercise 1: PR Summary Generation

1. Create a branch with a small change (add a field to an entity, update a migration)
2. Push and create a PR
3. Use the Copilot icon to generate a summary
4. Compare the generated summary with what you would have written

### Exercise 2: Request Copilot Review

1. On the same PR, add Copilot as a reviewer
2. Read through all Copilot comments
3. For each comment, decide: accept, dismiss, or discuss
4. Make the suggested fixes and push

### Exercise 3: Custom Review Instructions

1. Create `.github/copilot-review-instructions.md` in your repo
2. Add 10 rules specific to your project
3. Open a new PR and request Copilot review
4. Check if Copilot follows your custom instructions

### Exercise 4: Full Workflow

1. Create a GitHub issue: "Add health check endpoint that reports database and Kafka connectivity"
2. Follow the complete workflow from Step 1 to Step 10 above
3. Use Copilot at every stage
4. Time yourself — compare with your usual PR cycle time

### Exercise 5: Commit Message Suggestions

1. Stage some changes in VS Code
2. Open the Source Control panel
3. Click the Copilot sparkle icon in the commit message input
4. Review the suggested commit message — edit if needed
5. Practice this for your next 5 commits

---

## Next Steps

You now know how to use Copilot across the entire PR lifecycle. Move to [Tutorial 6: Advanced Patterns & Tips](06-advanced-patterns-and-tips.md) to learn power-user techniques.
