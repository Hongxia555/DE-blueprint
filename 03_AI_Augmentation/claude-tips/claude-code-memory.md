# Claude Code Memory System

## The 3 Layers of Memory

### 1. Project CLAUDE.md (`./CLAUDE.md`)
- **What it is**: Checked-in instructions for the current project
- **Loaded**: Fully, every session start
- **Scope**: Shared with your team via git
- **Best for**: Build commands, code style, architecture, team norms

### 2. User CLAUDE.md (`~/.claude/CLAUDE.md`)
- **What it is**: Personal preferences across all projects
- **Loaded**: Fully, every session start
- **Scope**: Machine-local, never shared
- **Best for**: Your editor preferences, rebasing vs. merging, personal habits

### 3. Auto Memory (`~/.claude/projects/<project>/memory/`)
- **What it is**: Claude-written learnings discovered across sessions
- **Loaded**: `MEMORY.md` index (first 200 lines / 25KB cap), topic files on-demand
- **Scope**: Per-repo, machine-local, not synced
- **Best for**: Discovered gotchas, debugging hints, build flags, patterns Claude learns at runtime

---

## How Auto Memory Works

Auto memory uses a two-file pattern:
- `MEMORY.md` — index file, always loaded at session start (200-line cap)
- `topic-name.md` — detailed notes, loaded on-demand when Claude needs them

Claude writes here when you say "remember X" or when it learns something worth saving across sessions.

**Storage path**: `~/.claude/projects/<encoded-project-path>/memory/`

---

## Trade-offs

| | Project CLAUDE.md | User CLAUDE.md | Auto Memory |
|---|---|---|---|
| **Who writes** | You | You | Claude |
| **Shared with team** | Yes | No | No |
| **Context cost** | Full file every session | Full file every session | Index only at start |
| **Reliability** | High (explicit rules) | High | Variable (Claude decides) |
| **Best for** | Standards, workflows | Personal habits | Discovered patterns |

---

## Key Guidelines

1. **Keep CLAUDE.md under 200 lines** — longer files reduce adherence
2. **Use `@path/to/file` imports** to split large CLAUDE.md into sub-files
3. **Use `.claude/rules/*.md` with `paths:` frontmatter** for file-type-specific rules (e.g., only load for `*.sql` files)
4. **Don't duplicate** — if it's in CLAUDE.md, don't also put it in auto memory
5. **Review auto memory periodically** — run `/memory` in a session to browse/edit/delete entries
6. **Sensitive info** stays out of auto memory — it's machine-local but still plain text files

---

## CLAUDE.md Loading Order

Claude walks **up** the directory tree, loading CLAUDE.md files from each level:

```
If you run Claude in: /my-monorepo/team-a/src/
It loads (in order):
  1. /my-monorepo/team-a/src/CLAUDE.md
  2. /my-monorepo/team-a/CLAUDE.md
  3. /my-monorepo/CLAUDE.md
  4. User CLAUDE.md (~/.claude/CLAUDE.md)
  5. Managed CLAUDE.md (org policy — cannot be excluded)
```

More specific paths take precedence over broader ones.

---

## Path-Scoped Rules (`.claude/rules/`)

Files in `.claude/rules/` support YAML frontmatter to scope rules to specific file patterns:

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "tests/**/*.test.ts"
---

# API Testing Rules
- All API tests require a local Redis instance
- Use the standard error response format
```

- Rules **without** `paths` frontmatter: load at session start (like CLAUDE.md)
- Rules **with** `paths` frontmatter: load on-demand when Claude reads matching files
- User-level rules at `~/.claude/rules/` apply to all projects

---

## Managing Memory

```bash
# In a Claude Code session:
/memory          # View all loaded files + browse/edit auto memory

# Key file paths:
~/.claude/CLAUDE.md                              # User-level preferences
./CLAUDE.md                                      # Project-level instructions
~/.claude/projects/<project>/memory/MEMORY.md   # Auto memory index
```

**Disable auto memory:**
```json
// .claude/settings.json
{
  "autoMemoryEnabled": false
}
```

**Custom auto memory location** (user or local settings only, not project settings):
```json
{
  "autoMemoryDirectory": "~/my-custom-memory-dir"
}
```

---

## Imports in CLAUDE.md

Break large CLAUDE.md files into smaller pieces using `@` imports:

```markdown
## Architecture
See @docs/architecture.md

## Personal Preferences
@~/.claude/my-preferences.md
```

- Relative paths resolve from the file containing the import
- Max import depth: 5 hops
- HTML comments (`<!-- ... -->`) are stripped before injection — useful for maintainer notes that don't consume tokens

---

## Context Cost Summary

| Source | Tokens at startup |
|---|---|
| 50-line CLAUDE.md | ~500 tokens |
| 200-line CLAUDE.md | ~1,500 tokens |
| Auto memory MEMORY.md (capped) | ~600 tokens |
| Topic files (on-demand) | 0 at startup |

---

## What NOT to Put in Auto Memory

Auto memory is Claude-written; manually avoid directing Claude to save:
- Things that belong in CLAUDE.md (coding standards, build commands)
- One-off debugging that won't recur
- Sensitive credentials or API keys

---

## Writing Effective CLAUDE.md Instructions

**Be specific:**
- ✅ `Use 2-space indentation in JavaScript files`
- ❌ `Format code properly`

**Use structured headers and bullets** — scannable format improves adherence over dense paragraphs.

**Test your instructions** — after adding a rule, ask Claude to follow it and verify behavior.
