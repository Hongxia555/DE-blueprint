---
name: ship
description: Stage, commit, and push changes in one flow. Combines /commit and /push into a single end-to-end skill.
tools: Bash
---

You are helping the user stage, commit, and push their changes end-to-end.

## Steps

1. Run `git status` to see all changes (staged and unstaged)
2. Run `git diff` to understand what changed
3. Run `git log --oneline -5` to understand this repo's commit message style
4. Draft a conventional commit message (see format below)
5. Present the message and the list of files to be committed
6. Immediately run `git add -A` then `git commit -m "<message>"` without asking for confirmation
7. Run `git log --oneline origin/$(git branch --show-current)..HEAD` to show what will be pushed
8. Immediately run `git push` without asking for confirmation

## Commit Message Format

```
<type>(<scope>): <short summary under 72 chars>

<optional body — explain the WHY, not the WHAT>
```

**Types:** `feat`, `fix`, `docs`, `refactor`, `chore`
**Scope** (optional): area changed, e.g. `meta-de`, `gumgum`, `skills`, `claude-tips`

**Rules:**
- Imperative mood: "add", not "added"
- No period at the end of the summary
- Body explains *why*, not a list of files

## Rules

- NEVER force push unless the user explicitly asks
- If `git push` fails, show the error and suggest a fix — do NOT retry automatically
- If there is nothing to commit, say so and stop
