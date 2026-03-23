# Tutorial 05: Claude Code in IDE

## Table of Contents

1. [Overview: Claude Code IDE Integration](#1-overview-claude-code-ide-integration)
2. [VS Code Integration](#2-vs-code-integration)
3. [JetBrains / IntelliJ IDEA Integration](#3-jetbrains--intellij-idea-integration)
4. [Terminal Integration](#4-terminal-integration)
5. [Side-by-Side with Other AI Tools](#5-side-by-side-with-other-ai-tools)
6. [Practical Workflow: Spring Boot Development in IntelliJ](#6-practical-workflow-spring-boot-development-in-intellij)
7. [Keyboard Shortcuts Reference](#7-keyboard-shortcuts-reference)
8. [Try This Exercises](#8-try-this-exercises)
9. [Tips and Gotchas](#9-tips-and-gotchas)

---

## 1. Overview: Claude Code IDE Integration

Claude Code works in three modes with IDEs:

| Mode | Description | Best For |
|------|-------------|----------|
| **Terminal inside IDE** | Run `claude` in the IDE's built-in terminal | All IDEs, simplest setup |
| **VS Code Extension** | Dedicated extension with chat panel and inline features | VS Code users |
| **JetBrains Plugin** | Plugin for IntelliJ IDEA and other JetBrains IDEs | IntelliJ/Java developers |

The terminal approach works universally, while the extensions add IDE-specific features like inline diff views and editor integration.

---

## 2. VS Code Integration

### Installing the Extension

```bash
# Install from the VS Code marketplace
# Search for "Claude Code" by Anthropic
# Or install from command line:
code --install-extension anthropic.claude-code
```

Alternatively, in VS Code:
1. Open the Extensions panel (`Cmd+Shift+X` on macOS)
2. Search for "Claude Code"
3. Install the extension by Anthropic

### Setting Up

After installation:
1. The extension uses your existing Claude Code authentication
2. If not authenticated, it will prompt you to run `claude auth login` in the terminal
3. Verify by opening the Claude Code panel

### Key Features

#### Chat Panel

The Claude Code chat panel appears in the sidebar (or can be docked anywhere).

```
1. Click the Claude icon in the Activity Bar (left sidebar)
2. Or use Cmd+Shift+P → "Claude Code: Open Chat"
```

The chat panel provides:
- Full conversation interface
- File context awareness (knows which files are open)
- Inline code suggestions with diff view
- One-click apply for code changes

#### Inline Editing

Select code in the editor and invoke Claude Code:

```
1. Select a block of code
2. Right-click → "Edit with Claude Code"
3. Or use keyboard shortcut: Cmd+Shift+E (configurable)
4. Describe the change you want
5. Review the inline diff
6. Accept or reject
```

**Example:**
```
Select a method → Right-click → "Edit with Claude Code"
Prompt: "Add input validation and proper error handling"
```

#### File Context

The extension automatically includes context about:
- Currently open file
- Selected text (if any)
- Files open in other tabs
- Project structure from CLAUDE.md

#### Terminal Integration

The extension can also run Claude Code in VS Code's integrated terminal:

```
1. Open terminal (Ctrl+`)
2. Run: claude
3. Works exactly like standalone CLI
```

### VS Code Settings

```json
// .vscode/settings.json
{
  "claude-code.model": "claude-sonnet-4-0520",
  "claude-code.autoAcceptEdits": false,
  "claude-code.showInlineHints": true
}
```

---

## 3. JetBrains / IntelliJ IDEA Integration

### Installing the Plugin

```
1. Open IntelliJ IDEA
2. Go to Settings → Plugins → Marketplace
3. Search for "Claude Code"
4. Install and restart IntelliJ
```

Or install via the command line:
```bash
# The plugin uses your existing Claude Code CLI installation
# Make sure Claude Code is installed first:
claude --version
```

### Setting Up in IntelliJ

After installing the plugin:

1. Open Settings (`Cmd+,` on macOS)
2. Navigate to Tools → Claude Code
3. Verify the Claude Code CLI path is detected
4. Check authentication status

### Using the Terminal Approach (Universal)

Even without the plugin, you can use Claude Code effectively in IntelliJ's terminal:

```
1. Open the Terminal tab (Alt+F12)
2. Run: claude
3. Claude Code can read and edit files in your IntelliJ project
4. IntelliJ will automatically detect file changes and reload
```

**Key advantage:** IntelliJ's auto-save and file watcher mean that when Claude Code edits a file, IntelliJ immediately reflects the change, including re-running inspections and updating the error list.

### Workflow: Terminal Claude Code + IntelliJ

This is the recommended approach for Java/Spring Boot developers:

```
┌─────────────────────────────────────────────────────┐
│ IntelliJ IDEA                                        │
│ ┌─────────────────────┬─────────────────────────────┐│
│ │                     │                              ││
│ │  Editor             │  Right Panel                 ││
│ │  (Java files)       │  (Structure / Database)      ││
│ │                     │                              ││
│ │                     │                              ││
│ ├─────────────────────┴─────────────────────────────┤│
│ │ Terminal Tab: claude                                ││
│ │ > Add retry logic to PaymentService.processPayment ││
│ │   using Resilience4j with exponential backoff      ││
│ └───────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

**Step-by-step:**

1. Open your Spring Boot project in IntelliJ
2. Open the Terminal tab at the bottom
3. Run `claude`
4. Ask Claude Code to make changes
5. Watch the files update in the editor above
6. IntelliJ's inspections run automatically on the changed files
7. Any compilation errors or warnings appear immediately
8. Use IntelliJ's normal tools (Run, Debug, Test) alongside

### Split Terminal Workflow

Use IntelliJ's split terminal to run Claude Code and tests simultaneously:

```
Terminal 1 (left): claude
Terminal 2 (right): mvn test -pl order-service --watch (or use IntelliJ test runner)
```

### IntelliJ Features That Complement Claude Code

| IntelliJ Feature | How It Helps with Claude Code |
|------------------|-------------------------------|
| Auto-import | Automatically adds imports after Claude Code edits |
| Code inspection | Immediately flags issues in Claude-generated code |
| Refactoring tools | Use IntelliJ's refactor for renames after Claude Code structural changes |
| Database tool | View database schema while asking Claude Code about queries |
| HTTP Client | Test endpoints Claude Code creates, right in the IDE |
| Git panel | Review Claude Code's changes in the Git diff view |
| Structure view | Quickly navigate code Claude Code has modified |

---

## 4. Terminal Integration

### Recommended Terminal Setup

For the best Claude Code experience in a terminal:

**iTerm2 (macOS):**
```bash
# Split panes: Cmd+D (horizontal), Cmd+Shift+D (vertical)
# Left pane: claude
# Right pane: your normal terminal workflow
```

**tmux:**
```bash
# Create a Claude Code layout
tmux new-session -s dev
tmux split-window -h
tmux select-pane -t 0
# Left pane: run claude
# Right pane: run tests, builds, etc.
```

**tmux config for Claude Code:**
```bash
# ~/.tmux.conf
# Create a dev layout with Claude Code
bind C-c new-window -n 'claude' 'claude'
bind C-t split-window -h 'mvn test'
```

### Terminal Multiplexer Workflows

```
┌──────────────────┬──────────────────┐
│ claude            │ Terminal          │
│                   │                   │
│ > Add pagination  │ $ mvn test       │
│   to OrderRepo    │ [INFO] Tests run │
│                   │ [INFO] BUILD OK  │
│                   │                   │
│                   │ $ git status     │
│                   │ modified: 3 files│
└──────────────────┴──────────────────┘
```

### Warp Terminal

Warp terminal has built-in AI features but you can still use Claude Code:
- Claude Code runs in Warp like any other CLI tool
- Warp's block-based output makes it easy to review Claude Code's responses
- Use Warp's command search (`Ctrl+R`) to recall previous Claude Code prompts

---

## 5. Side-by-Side with Other AI Tools

### Claude Code + GitHub Copilot

These tools complement each other well:

| Feature | Claude Code | GitHub Copilot |
|---------|-------------|----------------|
| Inline completions | No | Yes (real-time as you type) |
| Multi-file changes | Yes (excellent) | Limited |
| Code review | Yes | Limited |
| Terminal commands | Yes | No |
| Conversation context | Yes (200K tokens) | Limited |
| Architecture discussions | Yes | No |

**Recommended workflow:**
- Use **Copilot** for inline code completion while typing
- Use **Claude Code** for multi-file changes, reviews, debugging, and architecture

```
IntelliJ with Copilot enabled:
- Type code → Copilot suggests completions
- Need a bigger change? → Switch to terminal → use Claude Code
- Back to typing → Copilot helps again
```

### Claude Code + ChatGPT / Other AI

You might use different AI tools for different purposes:

| Task | Tool |
|------|------|
| Quick inline completion | GitHub Copilot |
| Multi-file code changes | Claude Code |
| Research / learning | Claude.ai web interface |
| Image/diagram analysis | Claude.ai (multimodal) |
| Quick code questions | Either Claude Code or ChatGPT |

### Avoiding Conflicts

- Claude Code and Copilot do not conflict; they operate at different levels
- If using multiple AI chat tools in terminals, keep them in separate terminal sessions to avoid confusion
- Be consistent about which tool you use for which task

---

## 6. Practical Workflow: Spring Boot Development in IntelliJ

### Scenario: Building a New Feature

You need to add an "order export" feature to your Spring Boot service.

**Step 1: Open IntelliJ with terminal**
```
Open your Spring Boot project in IntelliJ IDEA
Open Terminal tab (Alt+F12)
Run: claude
```

**Step 2: Plan the feature**
```
> Plan the implementation of an order export feature:
> - Endpoint: GET /api/v1/orders/export?format=csv&from=2024-01-01&to=2024-01-31
> - Supported formats: CSV, Excel (XLSX)
> - Should stream large exports (don't load everything into memory)
> - Include all order fields plus customer name and email
> - Require ADMIN or EXPORT role
```

**Step 3: Generate the code**
```
> Implement step 1 from the plan: create the ExportService with streaming CSV support.
```

Watch IntelliJ update the file in real time. Check the editor for any red underlines (compilation errors).

**Step 4: Use IntelliJ's tools**
```
- IntelliJ auto-imports the new classes
- Run the existing tests to make sure nothing broke (Ctrl+Shift+F10)
- Use IntelliJ's "Find Usages" to verify no unintended side effects
```

**Step 5: Test the new code**
```
> Write an integration test for the CSV export endpoint.
> Use Testcontainers for PostgreSQL and seed test data.
```

**Step 6: Run the test in IntelliJ**
```
Right-click the test class → Run
Or use the green play button next to the test method
If it fails, copy the error back to Claude Code:
> The export test fails with this error: [paste error]
```

**Step 7: Add the Excel format**
```
> Now add XLSX export support using Apache POI.
> Add the dependency and implement the Excel writer.
```

**Step 8: Manual review in IntelliJ**
```
- Open the Git tool window (Alt+9)
- Review all changes in IntelliJ's diff viewer
- Use "Annotate with Git Blame" to confirm only new code changed
```

**Step 9: Commit**
```
> /review
> /commit
```

### Scenario: Debugging with IntelliJ + Claude Code

**Step 1: Reproduce in IntelliJ**
```
Set a breakpoint in IntelliJ (click the gutter)
Run the application in Debug mode (Shift+F9)
Hit the endpoint to trigger the breakpoint
```

**Step 2: Describe to Claude Code what you see**
```
> I'm debugging OrderService.processOrder.
> At line 87, the `discount` variable is null even though the
> customer has an active promotion.
> Here's the relevant code path:
> [paste the method or describe what you see in the debugger]
```

**Step 3: Apply the fix**
```
> Fix the null discount issue by loading promotions before
> calculating the order total.
```

**Step 4: Verify in IntelliJ**
```
- IntelliJ hot-reloads the change (if using spring-boot-devtools)
- Re-run the debug scenario
- Confirm the fix
```

### Scenario: Database Work

**Step 1: Use IntelliJ's Database tool**
```
- Open Database tool window (right sidebar or View → Tool Windows → Database)
- Connect to your local PostgreSQL
- Browse tables, run queries, view data
```

**Step 2: Ask Claude Code about optimisation**
```
> This query from our OrderRepository takes 3 seconds:
>
> SELECT o.*, c.name, c.email
> FROM orders o
> JOIN customers c ON o.customer_id = c.id
> WHERE o.status = 'PENDING'
>   AND o.created_at > '2024-01-01'
> ORDER BY o.created_at DESC
> LIMIT 50
>
> The orders table has 5M rows. Suggest indexes and query optimisations.
```

**Step 3: Apply the fix**
```
> Create a Flyway migration for the recommended index.
> Also update the JPA repository method to use the optimised query.
```

**Step 4: Test in IntelliJ**
```
- Run the migration (Flyway auto-runs on app start)
- Use IntelliJ's Database tool to verify the index was created
- Run the query again and compare execution time
```

---

## 7. Keyboard Shortcuts Reference

### Claude Code in Terminal

| Shortcut | Action |
|----------|--------|
| `Enter` | Submit prompt |
| `Shift+Enter` | New line in prompt |
| `Ctrl+C` | Cancel current generation |
| `Ctrl+C` (twice) | Exit Claude Code |
| `Up Arrow` | Previous prompt in history |
| `Down Arrow` | Next prompt in history |
| `Escape` | Dismiss or cancel |
| `Tab` | Accept autocomplete suggestion |

### IntelliJ IDEA (Common Shortcuts Used with Claude Code)

| Shortcut | Action | Why Useful |
|----------|--------|------------|
| `Alt+F12` | Open Terminal | Quick access to Claude Code |
| `Ctrl+Shift+F10` | Run current test | Verify Claude Code's changes |
| `Shift+F9` | Debug | Debug issues found by Claude Code |
| `Alt+9` | Git tool window | Review Claude Code's file changes |
| `Ctrl+Shift+A` | Find Action | Quick access to any IntelliJ feature |
| `Ctrl+Shift+F` | Find in project | Verify Claude Code's refactoring |
| `Ctrl+Alt+L` | Reformat code | Clean up Claude Code's formatting |
| `Ctrl+Alt+O` | Optimise imports | Fix imports after Claude Code edits |
| `Double Shift` | Search Everywhere | Find files Claude Code created |

### VS Code (Common Shortcuts)

| Shortcut | Action |
|----------|--------|
| `` Ctrl+` `` | Toggle terminal |
| `Cmd+Shift+P` | Command palette |
| `Cmd+Shift+E` | Claude Code inline edit (with extension) |
| `Cmd+B` | Toggle sidebar |
| `Cmd+J` | Toggle bottom panel |
| `Cmd+K, Cmd+S` | Keyboard shortcuts settings |

---

## 8. Try This Exercises

### Exercise 1: Terminal in IDE Setup
1. Open your Spring Boot project in IntelliJ (or VS Code)
2. Open the integrated terminal
3. Run `claude`
4. Ask it to describe your project architecture
5. Ask it to make a small change (add a method, fix a typo)
6. Observe the file update in your IDE

### Exercise 2: Debug Workflow
1. Find a method in your code that handles edge cases
2. Set a breakpoint at the start of the method
3. Run in debug mode
4. Describe the debugging session to Claude Code
5. Ask Claude Code to improve the edge case handling

### Exercise 3: Split Workflow
1. Set up a split terminal (or two terminal tabs)
2. Run Claude Code in one, run tests in the other
3. Ask Claude Code to add a feature
4. Run tests in the other terminal
5. Feed test results back to Claude Code if anything fails

### Exercise 4: IDE + Claude Code Review
1. Make some code changes manually
2. Open IntelliJ's Git diff viewer (Alt+9)
3. In the terminal, run `/review`
4. Compare Claude Code's review with IntelliJ's code inspections
5. Apply suggestions and commit

### Exercise 5: Full Feature Cycle
Build a complete feature using the IntelliJ + Claude Code workflow:
1. Plan in Claude Code
2. Generate code in Claude Code
3. Review in IntelliJ's diff viewer
4. Run tests in IntelliJ
5. Debug any failures using IntelliJ's debugger + Claude Code
6. Commit using `/commit`

---

## 9. Tips and Gotchas

### Tips

1. **Let IntelliJ handle imports and formatting.** After Claude Code edits a Java file, run `Ctrl+Alt+O` (optimise imports) and `Ctrl+Alt+L` (reformat). IntelliJ is better at matching your project's formatting rules.

2. **Use IntelliJ's Local History as a safety net.** Even if you forget to commit before a large Claude Code change, IntelliJ's Local History (right-click file → Local History → Show History) tracks all changes.

3. **Keep the terminal visible while coding.** Dock the terminal at the bottom so you can see both the editor and Claude Code. Use a 70/30 split (editor/terminal).

4. **Use IntelliJ's "Mark as Modified" feature.** After Claude Code changes multiple files, use the Project view's "Show Diff" to quickly review all changes.

5. **Pipe IntelliJ's test output to Claude Code.** If a test fails, copy the failure output from IntelliJ's test runner and paste it into Claude Code for diagnosis.

6. **Use IntelliJ's HTTP Client for API testing.** When Claude Code creates a new endpoint, test it immediately with IntelliJ's `.http` files:
   ```http
   ### Test new export endpoint
   GET http://localhost:8080/api/v1/orders/export?format=csv&from=2024-01-01
   Authorization: Bearer {{auth_token}}
   ```

7. **IntelliJ's "Run Anything" (Double Ctrl) is great for quick commands.** Run Maven goals, Docker commands, or gradle tasks without leaving the IDE.

### Gotchas

1. **IntelliJ may show stale errors after Claude Code edits.** If you see red underlines that should not be there, try Build → Rebuild Project to refresh IntelliJ's analysis.

2. **Claude Code and IntelliJ auto-save can conflict.** IntelliJ auto-saves frequently. If Claude Code is in the middle of a multi-file edit, IntelliJ might trigger a build on a partially edited codebase. This is usually harmless but can show temporary compilation errors.

3. **Large file edits may cause IntelliJ to re-index.** This is normal and takes a few seconds. Let it complete before running tests.

4. **The VS Code extension and CLI are separate sessions.** A conversation in the VS Code extension panel is not shared with the terminal `claude` instance. They are independent.

5. **IntelliJ's memory consumption increases.** Running IntelliJ + Claude Code + Docker + tests simultaneously can be memory-intensive. Allocate at least 4GB to IntelliJ (`Help → Edit Custom VM Options → -Xmx4g`).

6. **Git changes from Claude Code may not auto-refresh in IntelliJ's Git panel.** Click the refresh button in the Git tool window to see the latest status.

7. **If using Copilot + Claude Code, be aware of context differences.** Copilot sees only the current file and nearby tabs. Claude Code sees whatever files it has read in the conversation. They may give different suggestions for the same problem.

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| VS Code | Install the extension for chat panel and inline editing |
| IntelliJ | Terminal approach works perfectly; let IntelliJ handle formatting and imports |
| Terminal | Split panes (tmux, iTerm2) for Claude Code + build/test side by side |
| Copilot | Use alongside Claude Code; Copilot for inline, Claude Code for multi-file |
| Debugging | Describe debugger state to Claude Code; use IntelliJ for stepping through code |
| Workflow | Editor for viewing, terminal for Claude Code, test runner for verification |

**Next:** [Tutorial 06 — Claude API and Spring AI](06-claude-api-and-spring-ai.md)
