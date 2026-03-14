# Claude Code Extension Types: A Spec Sheet

A reference guide explaining the differences between Slash Commands, Hooks, Skills, MCPs (Model Context Protocol), and Plugins in Claude Code.

---

## Quick Comparison

| Feature | Slash Commands | Hooks | Skills | MCPs | Plugins |
|---------|---------------|-------|--------|------|---------|
| **Triggered by** | User typing `/` | System events | User typing `/` | Claude (via tools) | External app |
| **Who runs the logic** | Claude (reads a prompt file) | Shell (runs a script) | Claude (follows instructions) | External server | External service |
| **Persistent connection** | No | No | No | Yes | Varies |
| **Provides new tools** | No | No | No | Yes | Varies |
| **Lives in** | `.claude/commands/` | `settings.json` / `CLAUDE.md` | `.claude/commands/` | `settings.json` | Varies |
| **Language** | Markdown prompt | Any shell script | Markdown prompt | Any (HTTP/stdio) | Varies |

---

## Slash Commands

**What they are:** Custom, reusable prompts you can trigger by typing `/command-name` in Claude Code.

**How they work:**
1. You create a Markdown file in `.claude/commands/` (e.g., `.claude/commands/review.md`)
2. That file contains the full prompt Claude will follow
3. When you type `/review`, Claude reads the file and executes the instructions

**File location:**
- **Project-level:** `.claude/commands/<name>.md` (available only in that project)
- **Global:** `~/.claude/commands/<name>.md` (available in all projects)

**Example:**
```
# File: .claude/commands/security-check.md

Review the current diff for security vulnerabilities.
Check for SQL injection, XSS, hardcoded secrets, and
insecure dependencies. Report findings by severity.
```

**Usage:** Type `/security-check` in Claude Code.

**Key characteristics:**
- Pure Markdown — no code required
- Arguments can be passed: `/command $ARGUMENTS`
- Special variables: `$ARGUMENTS`, `$CURRENT_FILE`, `$SELECTION`
- Can reference other files with `@file-path`
- Cannot execute code directly — they're prompt templates

---

## Hooks

**What they are:** Shell commands that Claude Code automatically runs in response to specific lifecycle events.

**How they work:**
1. You define hooks in your Claude settings (`settings.json` or project `CLAUDE.md`)
2. When a matching event fires, Claude Code runs your shell command
3. The hook output can be fed back to Claude or used to block/allow the action

**Available hook events:**
| Event | Fires when... |
|-------|--------------|
| `PreToolUse` | Before Claude calls any tool |
| `PostToolUse` | After a tool call completes |
| `Notification` | Claude sends a notification |
| `Stop` | Claude finishes responding |
| `UserPromptSubmit` | User submits a message |

**Example configuration (settings.json):**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'About to run a bash command' >> /tmp/claude-audit.log"
          }
        ]
      }
    ]
  }
}
```

**Key characteristics:**
- Run outside Claude's context — pure shell execution
- Can block Claude actions by returning a non-zero exit code
- Can inject additional context into Claude's flow via stdout
- Great for: audit logging, linting gates, automated testing triggers, notifications
- Security-sensitive: hooks run with your user permissions

---

## Skills

**What they are:** Pre-built, installable slash commands that Claude Code ships with or that you can add from the community. Sometimes also called "user-invocable skills."

**How they work:**
- Skills are slash commands but come pre-packaged — you don't write the Markdown yourself
- They're installed via `/EA-install` or by copying the `.md` file into `.claude/commands/`
- The Claude Code app lists built-in skills that are always available

**Built-in skills (examples):**
| Skill | What it does |
|-------|-------------|
| `/commit` | Creates a well-formatted git commit |
| `/review-pr` | Reviews a pull request |
| `/EA-prime` | Primes Claude with codebase understanding |
| `/EA-handoff` | Saves session state for later |
| `/EA-pickup` | Resumes from a saved handoff |

**Difference from custom slash commands:**
- Skills are **maintained and distributed** — you install them, not author them
- Custom slash commands are **authored by you** for project-specific needs
- Mechanically, they are the same thing once installed (a `.md` file in `.claude/commands/`)

**Key characteristics:**
- Installable globally or per-project
- Community-shareable
- Same format as slash commands under the hood
- Appear in the `/` autocomplete menu with descriptions

> **Note on "Skool":** The original request likely refers to **Skills** — Claude Code's term for installable slash commands. "Skool" may have been a voice-to-text artifact.

---

## MCPs (Model Context Protocol)

**What they are:** External servers that extend Claude's capabilities by providing new **tools**, **resources**, and **prompts** that Claude can call during a conversation.

**How they work:**
1. An MCP server runs as a separate process (local or remote)
2. Claude Code connects to it at startup via stdio or HTTP/SSE
3. The server registers tools (functions Claude can call), resources (data Claude can read), and prompts (reusable templates)
4. Claude calls these tools the same way it calls built-in tools like `Read` or `Bash`

**Configuration (settings.json):**
```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "your-token"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

**What MCPs can provide:**
| Capability | Example |
|-----------|---------|
| **Tools** | `create_github_issue`, `query_database`, `send_slack_message` |
| **Resources** | Live database schemas, file system access, API responses |
| **Prompts** | Pre-built prompt templates the MCP server defines |

**Key characteristics:**
- Persistent connection to an external service during the session
- Claude actively calls MCP tools (not triggered by user events)
- Can connect to databases, APIs, cloud services, internal tools
- Written in any language (Node.js, Python, Go, etc.)
- The MCP protocol is an open standard — anyone can build MCP servers
- Claude cannot use MCP tools unless they are configured and connected

**Difference from slash commands/hooks:**
- MCPs give Claude new **capabilities** (calling external APIs, reading live data)
- Slash commands give Claude new **instructions** (what to do with existing capabilities)
- Hooks give the **system** new behaviors (what to do around Claude's actions)

---

## Plugins

**What they are:** Third-party integrations that connect Claude Code to external tools and services — typically through IDE extensions, browser extensions, or platform integrations.

**How they work:**
- Plugins integrate at the **environment level**, not the prompt level
- Examples: VS Code extension for Claude, JetBrains plugin, browser-based Claude integrations
- They may surface Claude's capabilities in a different UI or workflow context

**Examples:**
| Plugin type | What it does |
|------------|-------------|
| VS Code extension | Adds Claude Code as a sidebar panel in VS Code |
| JetBrains plugin | Integrates Claude into IntelliJ/WebStorm |
| Browser extension | Adds Claude to GitHub web interface |
| CI/CD integration | Runs Claude as part of a GitHub Actions workflow |

**Key characteristics:**
- Live **outside** the Claude Code CLI itself
- Extend the environments where Claude is accessible
- May offer UI-level features (inline suggestions, context menus)
- Often configured through the plugin's own settings, not Claude's `settings.json`
- The GitHub App used in this repository is a form of plugin — it connects Claude to GitHub's issue/PR workflow

**Difference from MCPs:**
- Plugins change **where you interact with Claude** (VS Code, GitHub, browser)
- MCPs change **what Claude can do** (new tools and data sources)
- A plugin might internally use MCPs, but the concepts are distinct layers

---

## Summary: When to Use Each

| You want to... | Use... |
|---------------|--------|
| Reuse a prompt you write often | **Slash Command** |
| Run checks before/after Claude acts | **Hook** |
| Install a community prompt workflow | **Skill** |
| Let Claude talk to an external API or database | **MCP** |
| Use Claude inside your IDE or on GitHub | **Plugin** |

---

## How They Stack

```
┌─────────────────────────────────────────────┐
│              User / Developer                │
└──────────────────┬──────────────────────────┘
                   │ interacts via
        ┌──────────▼──────────┐
        │  Plugin / GitHub    │  ← Where you use Claude
        │  App / VS Code      │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │    Claude Code CLI   │  ← Core engine
        └──┬──────────────┬───┘
           │              │
  ┌────────▼──────┐  ┌────▼──────────┐
  │ Slash Commands│  │     Hooks     │  ← Customize behavior
  │   / Skills    │  │               │
  └───────────────┘  └───────────────┘
           │
  ┌────────▼──────────────────────────┐
  │      MCP Servers                   │  ← Extend capabilities
  │  (GitHub, Postgres, Slack, etc.)   │
  └────────────────────────────────────┘
```

---

*Last updated: March 2026*
