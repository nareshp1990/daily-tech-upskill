# Git & Code Review — Daily Prompts

> Tech stack: Git, GitHub PRs, branching strategies, merge conflicts, conventional commits

---

## Branch Strategy

### Design a Git branching strategy
```
Design a Git branching strategy for my team's Spring Boot microservices project.

Context:
- 6 developers, 2 QA
- Services deployed to AKS (dev → staging → prod)
- CI/CD via GitHub Actions
- 2-week sprint cycles
- Hotfix turnaround: < 2 hours

Cover:
1. Branch naming convention (feature/, bugfix/, hotfix/, release/)
2. Branch lifecycle (creation → PR → merge → delete)
3. Protection rules for main and release branches
4. When to use rebase vs merge commit
5. How release branches interact with main and develop
6. Hotfix flow that patches prod and backports to main

Show a visual diagram (ASCII) of the branch flow.
```

### Conventional commit message template
```
Create a conventional commit message guide for my Java/Spring Boot project.

Include:
1. Type prefixes: feat, fix, refactor, perf, test, docs, chore, ci, build
2. Scope conventions for our stack: (api), (service), (repo), (config), (kafka), (security), (infra)
3. Breaking change notation (BREAKING CHANGE footer or ! suffix)
4. Examples for each type in a Spring Boot context
5. Multi-line body for complex changes (what + why)
6. GitHub issue linking: "Closes #123" or "Refs #456"

Template:
<type>(<scope>): <short description>

<body — what changed and why>

<footer — breaking changes, issue refs>

Give 10 real-world examples from a Spring Boot microservices project.
```

---

## Pull Request

### Write a PR description
```
Write a GitHub PR description for the following changes.

Branch: feature/ORDER-1234-add-order-cancellation
Target: main
Changes: [paste diff summary or describe changes]

Structure:
## Summary
- What this PR does (1-3 bullet points)

## Motivation
- Why this change is needed (link to ticket/issue)

## Changes
- Detailed list of what changed (grouped by area)

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing steps (if applicable)
- [ ] Edge cases covered

## Screenshots / Logs
(if applicable)

## Deployment Notes
- Any config changes, feature flags, migrations
- Rollback plan

## Checklist
- [ ] Self-reviewed the diff
- [ ] No secrets or credentials committed
- [ ] API changes documented
- [ ] Backward compatible (or migration plan documented)

Make it concise but complete. Reviewer should understand the "why" without reading every line of code.
```

### PR review checklist by area
```
Create a comprehensive code review checklist for a Spring Boot PR.

Organise by area:

**Code Quality:**
- Naming (classes, methods, variables follow Java conventions)
- Single responsibility (methods < 30 lines, classes have one job)
- No dead code, commented-out blocks, or TODOs without tickets
- Proper use of Optional (no .get() without check)

**Spring-Specific:**
- Correct annotations (@Service, @Repository, @Transactional boundaries)
- No business logic in controllers
- Constructor injection (not field injection)
- @Transactional on service methods that need it (read-only where possible)

**API Design:**
- REST conventions (proper HTTP methods, status codes)
- Input validation (@Valid, custom validators)
- Consistent error response format
- No sensitive data in responses

**Data Layer:**
- N+1 query risks (use @EntityGraph or JOIN FETCH)
- Index considerations for new queries
- Flyway migration included for schema changes
- Proper cascade and fetch type settings

**Security:**
- No hardcoded secrets
- Authorization checks on endpoints
- Input sanitization
- SQL injection prevention (parameterized queries)

**Testing:**
- Unit tests for business logic
- Integration tests for API endpoints
- Edge cases and error paths covered
- Test data doesn't depend on execution order

**Performance:**
- Pagination for list endpoints
- Appropriate caching
- No unnecessary eager loading
- Async for long-running operations

Format as a markdown checklist a reviewer can copy-paste into a PR comment.
```

---

## Code Review

### Review this code for issues
```
Review the following Spring Boot code for issues.

[paste code here]

Check for:
1. **Bugs**: null pointer risks, race conditions, off-by-one errors
2. **Security**: injection, auth bypass, data exposure
3. **Performance**: N+1 queries, missing indexes, unnecessary allocations
4. **Spring pitfalls**: wrong @Transactional scope, missing @Async proxy issues, circular dependencies
5. **Clean code**: naming, method length, single responsibility
6. **Error handling**: swallowed exceptions, generic catches, missing error responses

For each issue:
- Severity: 🔴 Critical | 🟡 Warning | 🔵 Suggestion
- Line reference
- What's wrong
- How to fix (with code snippet)

Be honest — if the code is good, say so. Don't invent issues.
```

### Suggest refactoring for a method
```
Refactor this method to improve readability and maintainability.

[paste method here]

Guidelines:
1. Extract meaningful helper methods (name reveals intent)
2. Replace nested if-else with early returns or strategy pattern
3. Use streams where they improve clarity (not just for style)
4. Apply guard clauses for preconditions
5. Improve variable names
6. Add @Nullable / @NonNull annotations where appropriate

Show:
- Before (annotated with issues)
- After (refactored)
- Explain each change and why it matters
```

---

## Merge Conflicts

### Resolve a merge conflict
```
I have a merge conflict in the following file after merging main into my feature branch.

File: [filename]
My branch changes: [describe what you changed]
Main branch changes: [describe what changed on main]

Conflict markers:
<<<<<<< HEAD
[my code]
=======
[their code]
>>>>>>> main

Help me resolve this:
1. Understand what both sides intended
2. Determine the correct merged result that preserves both intents
3. Show the resolved code
4. Flag if the resolution needs new tests or could break something
```

### Rebase strategy for long-lived branch
```
My feature branch is 47 commits behind main and has 12 commits of its own. Some files conflict.

Help me plan the rebase:

1. Should I rebase or merge? (pros/cons for this situation)
2. If rebase: interactive rebase strategy
   - Which commits to squash
   - Which to keep separate
   - Order of operations
3. How to handle conflicts incrementally
4. How to verify nothing broke after rebase
5. Force-push safety (--force-with-lease)

Branch: feature/ORDER-5678-inventory-rework
Files likely to conflict: [list files]
```

---

## Git Operations

### Undo mistakes safely
```
I need to undo a Git mistake. Help me choose the safest approach.

Scenario: [describe what happened]
- Accidentally committed to main
- Committed secrets / large file
- Need to revert a merged PR
- Lost work after reset
- Wrong commit message

For each approach:
1. Command(s) to run
2. What it actually does (data safety)
3. Impact on remote / other developers
4. Point of no return warning
5. Verification step

Prefer non-destructive options. Only suggest --force if absolutely necessary and explain the risk.
```

### Git bisect to find a bug
```
A bug was introduced somewhere in the last 30 commits. Help me use git bisect.

Bug symptom: [describe the bug]
Last known good commit: [commit hash or "about 2 weeks ago"]
Current bad commit: HEAD

Walk me through:
1. git bisect start
2. Marking good/bad commits
3. Writing a bisect script for automation:
   - Compile the project: mvn compile
   - Run the specific failing test: mvn test -pl module -Dtest=TestClass#method
   - Exit code determines good/bad
4. Interpreting the result
5. git bisect reset cleanup

Show the exact commands and a bisect.sh script.
```

### Cherry-pick across branches
```
I need to cherry-pick specific commits from one branch to another.

Source branch: [branch name]
Target branch: [branch name]
Commits to pick: [commit hashes or descriptions]

Help me:
1. Verify the commits and their dependencies
2. Determine the correct order
3. Handle potential conflicts
4. Preserve authorship
5. Create a clean commit message referencing the original

Commands:
git cherry-pick <hash>          # single
git cherry-pick A..B            # range
git cherry-pick -x <hash>       # with original ref
git cherry-pick --no-commit     # stage without committing (for squashing)
```

---

## Git History & Investigation

### Investigate who changed what and why
```
I need to understand the history of a specific file or function.

File: [path]
Function/section: [name or line range]

Run through:
1. git log --follow -- <file> (rename-aware history)
2. git log -p -S "functionName" (pickaxe search)
3. git blame <file> (line-by-line attribution)
4. git log --all --grep="TICKET-123" (find related commits)

For each significant change:
- Who made it and when
- What the commit message says
- Whether it was part of a PR (cross-reference)
- Any pattern (same area keeps changing = design issue?)
```

### Clean up commit history before PR
```
I have these commits on my feature branch that I want to clean up before creating a PR:

[list commits or paste git log --oneline]

Help me create a clean, reviewable history:
1. Squash WIP/fixup commits
2. Reorder so related changes are together
3. Write clear commit messages for each logical unit
4. Split commits that do too many things
5. Remove any "oops" commits (debug logs, accidental files)

Show the interactive rebase plan:
git rebase -i HEAD~N

pick/squash/fixup/reword/edit for each commit with reasoning.
```

---

## GitHub-Specific

### Set up branch protection rules
```
Recommend GitHub branch protection rules for my Spring Boot project.

Branches to protect: main, release/*

For main:
- Require PR with at least [N] approvals
- Dismiss stale reviews on new push
- Require status checks: build, test, sonar-quality-gate
- Require branches to be up-to-date
- Require signed commits (yes/no — trade-offs?)
- Restrict who can push (team leads only)
- No force pushes, no deletions

For release/*:
- Similar but allow release manager to push directly for version bumps
- Require CI green

Show the GitHub API / CLI commands:
gh api repos/{owner}/{repo}/branches/{branch}/protection --method PUT ...
```

### Automate PR labels and assignments
```
Create a GitHub Actions workflow that automatically:

1. Labels PRs based on file paths:
   - src/main/java/**/controller/** → "api"
   - src/main/java/**/service/** → "business-logic"
   - src/main/resources/db/migration/** → "database"
   - .github/** → "ci-cd"
   - helm/** or terraform/** → "infra"
   - src/test/** → "tests"

2. Assigns reviewers based on CODEOWNERS:
   - Backend lead for service/repo changes
   - DevOps lead for infra changes
   - Security team for auth/security changes

3. Adds size labels: XS (<10 lines), S (<50), M (<200), L (<500), XL (>500)

4. Posts a comment with:
   - Files changed summary grouped by area
   - Migration detected warning (if db/migration files changed)
   - Docker image rebuild needed (if Dockerfile changed)

Show: .github/workflows/pr-labeler.yml and .github/CODEOWNERS
```

---

## Stash & Workspace

### Smart stash workflow
```
I'm in the middle of feature work and need to context-switch. Help me stash properly.

Current situation: [describe what you're working on]
Need to switch to: [urgent bug / different feature / code review]

Show me:
1. git stash push -m "descriptive message" -- <specific files>  (targeted stash)
2. git stash list (verify)
3. Switch, do the other work, come back
4. git stash pop vs git stash apply (which to use when)
5. Handling stash conflicts on pop

Advanced:
- git stash push --keep-index (stash unstaged, keep staged)
- git stash push -u (include untracked files)
- git stash branch <new-branch> stash@{0} (pop stash into new branch)
```

---

*Tailored for: Git / GitHub / Spring Boot microservices / Team workflows*
