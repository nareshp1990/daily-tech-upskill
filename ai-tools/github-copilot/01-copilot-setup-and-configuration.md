# Tutorial 1: GitHub Copilot Setup & Configuration

## Table of Contents

- [Plans and Pricing](#plans-and-pricing)
- [Installation](#installation)
  - [VS Code](#vs-code)
  - [IntelliJ IDEA](#intellij-idea)
  - [Neovim](#neovim)
  - [GitHub CLI](#github-cli)
- [Configuration and Settings](#configuration-and-settings)
- [Enabling/Disabling for Specific Languages](#enablingdisabling-for-specific-languages)
- [Proxy and Network Configuration](#proxy-and-network-configuration)
- [Project-Level Custom Instructions](#project-level-custom-instructions)
- [Verification and Testing](#verification-and-testing)
- [Try This Exercises](#try-this-exercises)

---

## Plans and Pricing

### Copilot Individual

- **Cost**: $10/month or $100/year
- **Includes**:
  - Code completions in IDE
  - Copilot Chat in IDE
  - Copilot CLI
  - Copilot in GitHub.com (PR summaries, etc.)
  - Multi-model selection (GPT-4o, Claude 3.5 Sonnet, etc.)
- **Best for**: Personal projects, freelancers, open-source contributors

### Copilot Business

- **Cost**: $19/user/month
- **Everything in Individual, plus**:
  - Organization-wide policy management
  - Audit logs
  - IP indemnity
  - Content exclusion policies (exclude specific files/repos)
  - Copilot metrics and usage dashboard
  - SSO/SAML support
- **Best for**: Teams and companies

### Copilot Enterprise

- **Cost**: $39/user/month
- **Everything in Business, plus**:
  - Copilot knowledge bases (index internal repos for RAG)
  - Fine-tuned models on your codebase
  - Copilot Workspace (issue-to-PR automation)
  - Bing web search integration in chat
  - Advanced security features
- **Best for**: Large enterprises with custom codebases

### Free Tier (Copilot Free)

- Limited completions and chat messages per month
- Available to all GitHub users
- Good for trying out Copilot before subscribing

> **Recommendation**: Start with Individual if personal, or push for Business in your organization. Enterprise is worth it if your company has large proprietary codebases that Copilot can index.

---

## Installation

### VS Code

#### Step 1: Install the Extension

```
1. Open VS Code
2. Go to Extensions (Ctrl+Shift+X)
3. Search "GitHub Copilot"
4. Install "GitHub Copilot" (by GitHub) — this includes both completions and chat
```

Alternatively, from the command line:

```bash
code --install-extension GitHub.copilot
```

#### Step 2: Sign In

```
1. After installation, click the Copilot icon in the status bar (bottom right)
2. Click "Sign in to GitHub"
3. Authorize in your browser
4. Return to VS Code — you should see the Copilot icon active
```

#### Step 3: Verify

Open any `.java` file and start typing. You should see ghost text (gray suggestions) appearing.

---

### IntelliJ IDEA

#### Step 1: Install the Plugin

```
1. Open IntelliJ IDEA
2. Go to Settings/Preferences → Plugins
3. Search "GitHub Copilot" in the Marketplace tab
4. Click Install
5. Restart IntelliJ IDEA
```

#### Step 2: Sign In

```
1. After restart, go to Tools → GitHub Copilot → Login to GitHub
2. A dialog shows a device code — copy it
3. Open the URL in your browser, paste the code, authorize
4. Return to IntelliJ — you should see "GitHub Copilot" in the status bar
```

#### Step 3: Configure

```
Settings → Tools → GitHub Copilot
- Enable/disable completions
- Enable/disable Copilot Chat
- Configure keybindings
```

**IntelliJ-Specific Shortcuts**:

| Action | Shortcut |
|--------|----------|
| Accept suggestion | `Tab` |
| Dismiss | `Esc` |
| Next suggestion | `Alt + ]` |
| Previous suggestion | `Alt + [` |
| Open Copilot Chat | `Alt + C` (custom) |

---

### Neovim

#### Prerequisites

- Neovim 0.6+ (0.9+ recommended)
- Node.js 18+
- Git

#### Step 1: Install with Plugin Manager

**Using lazy.nvim**:

```lua
-- In your lazy.nvim plugin spec
{
  "zbirenbaum/copilot.lua",
  cmd = "Copilot",
  event = "InsertEnter",
  config = function()
    require("copilot").setup({
      suggestion = {
        enabled = true,
        auto_trigger = true,
        keymap = {
          accept = "<M-l>",
          accept_word = "<M-k>",
          accept_line = "<M-j>",
          next = "<M-]>",
          prev = "<M-[>",
          dismiss = "<C-]>",
        },
      },
      panel = {
        enabled = true,
      },
      filetypes = {
        yaml = true,
        markdown = true,
        java = true,
        dockerfile = true,
        terraform = true,
      },
    })
  end,
}
```

**Using vim-plug** (official plugin):

```vim
Plug 'github/copilot.vim'
```

#### Step 2: Authenticate

```vim
:Copilot auth
```

Follow the browser-based authentication flow.

#### Step 3: Verify

```vim
:Copilot status
```

Should show "Copilot: Ready".

---

### GitHub CLI

#### Step 1: Install GitHub CLI

```bash
# macOS
brew install gh

# Ubuntu/Debian
sudo apt install gh

# Windows
winget install GitHub.cli
```

#### Step 2: Install Copilot Extension

```bash
gh extension install github/gh-copilot
```

#### Step 3: Authenticate

```bash
gh auth login
gh auth refresh -s copilot
```

#### Step 4: Verify

```bash
gh copilot --help
```

---

## Configuration and Settings

### VS Code Settings (settings.json)

Open settings: `Ctrl+Shift+P` → "Preferences: Open Settings (JSON)"

```json
{
  // Enable/disable Copilot globally
  "github.copilot.enable": {
    "*": true,
    "plaintext": false,
    "markdown": true,
    "scminput": false
  },

  // Editor settings that affect Copilot
  "editor.inlineSuggest.enabled": true,
  "editor.inlineSuggest.showToolbar": "onHover",

  // Copilot Chat settings
  "github.copilot.chat.localeOverride": "en",

  // Advanced settings
  "github.copilot.advanced": {
    "debug.overrideEngine": "",
    "debug.testOverrideProxyUrl": "",
    "debug.overrideProxyUrl": ""
  }
}
```

### IntelliJ Settings

```
Settings → Tools → GitHub Copilot:
  ✅ Enable GitHub Copilot
  ✅ Enable auto-completions

Settings → Editor → General → Inline Completion:
  ✅ Show inline completions
  Delay: 0ms (immediate) or 300ms (less aggressive)
```

---

## Enabling/Disabling for Specific Languages

### VS Code

In `settings.json`:

```json
{
  "github.copilot.enable": {
    "*": true,
    "java": true,
    "yaml": true,
    "dockerfile": true,
    "terraform": true,
    "sql": true,
    "properties": true,
    "xml": true,
    "json": true,
    "markdown": true,
    "plaintext": false,
    "env": false
  }
}
```

> **Important**: Disable Copilot for `.env` files to avoid leaking secrets. Also consider disabling for `plaintext` to prevent accidental sensitive data exposure.

### Workspace-Level Override

Create `.vscode/settings.json` in your project:

```json
{
  "github.copilot.enable": {
    "*": true,
    "env": false,
    "properties": false
  }
}
```

### Per-File Toggle

Click the Copilot icon in the VS Code status bar to toggle on/off for the current file type.

---

## Proxy and Network Configuration

### Corporate Proxy Setup (VS Code)

In `settings.json`:

```json
{
  "http.proxy": "http://proxy.company.com:8080",
  "http.proxyStrictSSL": true,
  "http.proxyAuthorization": null,
  "github.copilot.advanced": {
    "debug.overrideProxyUrl": "http://proxy.company.com:8080"
  }
}
```

### Environment Variables

```bash
# In your shell profile (.bashrc, .zshrc)
export HTTP_PROXY="http://proxy.company.com:8080"
export HTTPS_PROXY="http://proxy.company.com:8080"
export NO_PROXY="localhost,127.0.0.1,.company.com"
```

### SSL Certificate Issues

If your company uses a custom CA certificate:

```bash
# Point Node.js to your company CA bundle
export NODE_EXTRA_CA_CERTS="/path/to/company-ca-bundle.crt"
```

In VS Code `settings.json`:

```json
{
  "http.proxyStrictSSL": false
}
```

> **Note**: Only set `proxyStrictSSL` to false as a last resort. Prefer adding the CA certificate properly.

### Firewall Requirements

Ensure these domains are allowed:

```
github.com
api.github.com
copilot-proxy.githubusercontent.com
default.exp-tas.com
copilot-telemetry.githubusercontent.com
```

---

## Project-Level Custom Instructions

### The `.github/copilot-instructions.md` File

Create this file in your repository root to give Copilot project-specific context:

```bash
mkdir -p .github
touch .github/copilot-instructions.md
```

### Example for a Spring Boot Microservices Project

```markdown
# Copilot Instructions

## Project Context
This is a Spring Boot 3.x microservices project using Java 21.

## Code Style
- Use constructor injection (never field injection with @Autowired)
- Use records for DTOs and value objects
- Use Optional instead of null checks
- Follow the package structure: controller → service → repository → entity
- Use Lombok only for @Slf4j; write constructors and getters explicitly or use records

## Naming Conventions
- REST controllers: `*Controller.java`
- Services: `*Service.java` (interface) and `*ServiceImpl.java` (implementation)
- Repositories: `*Repository.java`
- Entities: singular nouns, e.g., `Order.java`, not `Orders.java`
- DTOs: `*Request.java`, `*Response.java`
- Test classes: `*Test.java` (unit), `*IntegrationTest.java` (integration)

## Architecture Patterns
- Hexagonal architecture for domain services
- CQRS for read-heavy endpoints
- Event-driven communication via Kafka between services
- Circuit breaker (Resilience4j) for external API calls

## Testing
- Use JUnit 5 + Mockito for unit tests
- Use @SpringBootTest + Testcontainers for integration tests
- Use AssertJ for assertions (not Hamcrest)
- Aim for 80%+ line coverage on service layer

## Database
- MySQL 8.x with Flyway migrations
- Migration files: `V{number}__{description}.sql`
- Always use parameterized queries (never string concatenation)

## Infrastructure
- Docker Compose for local development
- Kubernetes (AKS) for production
- Terraform for Azure infrastructure
- GitHub Actions for CI/CD

## Error Handling
- Use @ControllerAdvice with @ExceptionHandler
- Return ProblemDetail (RFC 7807) for API errors
- Log at appropriate levels: ERROR for unrecoverable, WARN for recoverable, INFO for business events

## Security
- Never hardcode secrets — use Azure Key Vault or environment variables
- Validate all input with Jakarta Bean Validation
- Use Spring Security with JWT for API authentication
```

### How Copilot Uses These Instructions

- Copilot reads this file as additional context for **all** suggestions in the repository
- It applies to both code completions and Chat responses
- It works in VS Code and IntelliJ (as of 2025+)
- Keep the file concise — under 1000 words for best results
- Use clear, imperative language ("Use X", "Never do Y")

---

## Verification and Testing

### Quick Smoke Test

1. **Create a new Java file** and type:

```java
// REST controller for managing users with CRUD operations
```

Wait 1-2 seconds. Copilot should suggest a full controller class.

2. **Open Copilot Chat** (`Ctrl+Alt+I` in VS Code) and type:

```
@workspace What is the structure of this project?
```

Copilot should describe your project based on the file tree.

3. **Test CLI**:

```bash
gh copilot suggest "find all running Docker containers and their memory usage"
```

Should suggest a valid `docker` command.

### Troubleshooting

| Issue | Fix |
|-------|-----|
| No suggestions appearing | Check Copilot status bar icon; re-authenticate with `Ctrl+Shift+P → "GitHub Copilot: Sign In"` |
| Suggestions are slow | Check network/proxy; try disabling other extensions |
| Chat not working | Ensure you have the latest extension version; restart IDE |
| CLI not found | Run `gh extension upgrade gh-copilot` |
| IntelliJ plugin conflict | Disable JetBrains AI Assistant if installed — it conflicts with Copilot |

---

## Try This Exercises

### Exercise 1: Full Setup

1. Install Copilot in your primary IDE
2. Verify authentication works
3. Open an existing Spring Boot project and confirm you see inline suggestions

### Exercise 2: Custom Instructions

1. Create `.github/copilot-instructions.md` in one of your projects
2. Add at least 5 rules specific to your project
3. Open a Java file and see if Copilot follows the instructions (e.g., "use records for DTOs" — start typing a DTO class and check)

### Exercise 3: Language Configuration

1. Disable Copilot for `.env` and `.properties` files
2. Enable Copilot for `Dockerfile` and `*.tf` files
3. Toggle Copilot on/off using the status bar icon

### Exercise 4: CLI Setup

1. Install `gh copilot`
2. Run `gh copilot suggest "list all Kubernetes pods in the staging namespace"`
3. Run `gh copilot explain "kubectl get pods -n staging -o jsonpath='{.items[*].metadata.name}'"`

---

## Next Steps

Once setup is complete, move to [Tutorial 2: Code Completion Mastery](02-code-completion-mastery.md) to learn how to get the most out of inline suggestions.
