# claude-plugins-official — What It Is and How It Works

## What Is It?

The **official Claude Code plugin marketplace** — an app store for Claude Code plugins, maintained by Anthropic. Install plugins to extend Claude Code with language servers, dev workflows, and third-party service integrations.

**Repo:** `Repos/claude-plugins-official/`

**Install a plugin:**
```bash
/plugin install github@claude-plugins-official
# or browse via:
/plugin > Discover
```

---

## Structure

```
claude-plugins-official/
├── plugins/          # Anthropic-built plugins
└── external_plugins/ # Third-party / community plugins
```

Each plugin follows the standard structure:
```
plugin-name/
├── .claude-plugin/plugin.json   # Metadata (required)
├── .mcp.json                    # MCP server config (optional)
├── commands/                    # Slash commands
├── skills/                      # Auto-invoked skills
├── agents/                      # Agent definitions
└── README.md
```

---

## Anthropic-Built Plugins (`/plugins`)

### Language Servers (LSP)
Give Claude Code IDE-like intelligence: autocomplete, diagnostics, go-to-definition.

| Plugin | Language |
|---|---|
| `pyright-lsp` | Python |
| `typescript-lsp` | TypeScript / JavaScript |
| `rust-analyzer-lsp` | Rust |
| `gopls-lsp` | Go |
| `clangd-lsp` | C / C++ |
| `ruby-lsp` | Ruby |
| `swift-lsp` | Swift |
| `kotlin-lsp` | Kotlin |
| `csharp-lsp` | C# |
| `php-lsp` | PHP |
| `lua-lsp` | Lua |
| `jdtls-lsp` | Java |

### Developer Workflow
| Plugin | What it does |
|---|---|
| `code-review` | Structured code review with severity ratings |
| `pr-review-toolkit` | Pull request review workflows |
| `commit-commands` | Commit message generation and management |
| `feature-dev` | End-to-end feature development workflow |
| `hookify` | Claude Code hooks management |
| `security-guidance` | Security review and vulnerability guidance |
| `code-simplifier` | Refactor and simplify code |
| `ralph-loop` | Automated iteration loop |
| `playground` | Experimental/testing sandbox |

### Meta / Plugin Building
| Plugin | What it does |
|---|---|
| `plugin-dev` | Build and test new plugins |
| `skill-creator` | Scaffold new skills |
| `claude-md-management` | Manage CLAUDE.md files across a project |
| `agent-sdk-dev` | Develop with the Claude Agent SDK |
| `example-plugin` | Reference implementation for plugin structure |

### Output Style
| Plugin | What it does |
|---|---|
| `learning-output-style` | Claude explains as it works (good for learning) |
| `explanatory-output-style` | Verbose, step-by-step reasoning in responses |

---

## Third-Party Plugins (`/external_plugins`)

| Plugin | What it does |
|---|---|
| `github` | GitHub: PRs, issues, repos |
| `gitlab` | GitLab: MRs, CI/CD, issues |
| `slack` | Slack: send messages, search channels |
| `linear` | Linear: issues, projects, roadmaps |
| `asana` | Asana: tasks and projects |
| `stripe` | Stripe: payments, customers, subscriptions |
| `supabase` | Supabase: DB, auth, storage |
| `firebase` | Firebase: Firestore, auth, hosting |
| `playwright` | Browser automation and testing |
| `greptile` | Code search across large repos |
| `context7` | Live documentation lookup |
| `serena` | Code intelligence and navigation |
| `discord` | Discord: messages and channels |
| `telegram` | Telegram: messages and bots |
| `fakechat` | Mock chat UI for prototyping |
| `laravel-boost` | Laravel-specific PHP tooling |

---

## How It Differs from Other Plugin Repos

| | `claude-plugins-official` | `knowledge-work-plugins` |
|---|---|---|
| **Focus** | Developer tools + SaaS integrations | Job function roles (data, sales, legal, finance) |
| **Built by** | Anthropic + community partners | Anthropic |
| **Use case** | Software engineering and coding | Knowledge work productivity |
| **Key value** | LSP intelligence, service APIs | Domain expertise and workflow automation |
