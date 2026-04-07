# knowledge-work-plugins — What It Is and How It Works

## What Is It?

A collection of pre-built **Claude plugins** for common job functions — data, sales, legal, finance, product, marketing, etc. Built for [Claude Cowork](https://claude.com/product/cowork), also compatible with Claude Code.

Each plugin turns Claude into a role-specific expert, with slash commands, auto-invoked skills, and live connections to tools your role actually uses (Snowflake, Jira, Slack, BigQuery, etc.).

**Repo:** `Repos/knowledge-work-plugins/`

---

## Plugins vs Claude Code Skills

| | Claude Code Skills | Plugins |
|---|---|---|
| **What it is** | A markdown prompt template | Skills + commands + live tool connections, packaged together |
| **Lives in** | `.claude/skills/<name>/SKILL.md` | Folder with `plugin.json`, `.mcp.json`, `skills/`, `commands/` |
| **Invoked by** | `/skill-name` | `/plugin:command` or auto-fires when relevant |
| **External tools** | No — Claude only sees what you paste | Yes — MCP connects Claude to live Snowflake, BigQuery, Jira, Slack, etc. |
| **Who it's for** | One person, local session | A team — installable and shared |
| **Complexity** | One markdown file | Manifest + connectors + multiple skills |

**In short:** Skills = smarter prompts. Plugins = smarter prompts + live data + team distribution.

---

## How Skills Work (the shared foundation)

A skill is a markdown file loaded into the conversation when invoked. Claude reads the instructions and follows them. That's it.

**Example — the `write-query` skill inside the data plugin:**
```markdown
---
name: write-query
description: Write optimized SQL for your dialect. Use when translating a natural-language data need into SQL.
argument-hint: "<description of what data you need>"
---

## Workflow
1. Parse the request: output columns, filters, aggregations, joins
2. Determine dialect: Snowflake, BigQuery, Postgres, Presto, etc.
3. Discover schema (if warehouse connected)
4. Write the query with best practices: partition filters, CTEs, comments
5. Explain key decisions
```

Your WOW skills follow the same pattern — `/sql-review`, `/metric-design`, `/metric-breakdown` are all exactly this: structured prompts you'd find tedious to type every time.

---

## How Plugins Add to This

Plugins wrap skills in two additional layers:

### 1. The MCP Connector (the big differentiator)

`.mcp.json` wires Claude to external services. Without it, Claude only knows what you paste. With it, Claude queries live systems:

```json
{
  "mcpServers": {
    "bigquery": { "type": "http", "url": "https://bigquery.googleapis.com/mcp" },
    "snowflake": { "type": "http", "url": "" },
    "atlassian": { "type": "http", "url": "https://mcp.atlassian.com/v1/mcp" }
  }
}
```

You say "show me tables with more than 1M rows" → Claude queries Snowflake directly. No copy-pasting.

### 2. Explicit Commands vs Auto-Invoked Skills

- **Skills** fire silently when Claude detects they're relevant (you ask a SQL question → `write-query` skill activates)
- **Commands** are explicit — `/data:write-query <what I need>` — you trigger them deliberately

### Plugin folder structure

```
data/
├── .claude-plugin/plugin.json   # Manifest: name, version, description
├── .mcp.json                    # Tool connections
├── skills/
│   ├── write-query/SKILL.md     # Auto-invoked
│   ├── validate-data/SKILL.md
│   ├── sql-queries/SKILL.md
│   └── explore-data/SKILL.md
└── commands/
    ├── analyze/                 # /data:analyze
    ├── build-dashboard/         # /data:build-dashboard
    └── create-viz/              # /data:create-viz
```

---

## Available Plugins in This Repo

| Plugin | What it does | Key connectors |
|---|---|---|
| `data` | SQL, exploration, visualization, dashboards, validation | Snowflake, BigQuery, Databricks, Hex, Amplitude |
| `productivity` | Tasks, calendars, daily workflows | Slack, Notion, Asana, Linear, Jira |
| `sales` | Prospect research, call prep, pipeline review, outreach | HubSpot, Clay, Fireflies, ZoomInfo |
| `product-management` | Specs, roadmaps, user research synthesis | Linear, Jira, Figma, Amplitude, Intercom |
| `marketing` | Content, campaigns, brand voice, competitive briefs | Canva, HubSpot, Ahrefs, Klaviyo |
| `legal` | Contract review, NDA triage, compliance, risk | Box, Jira, Microsoft 365 |
| `finance` | Journal entries, reconciliation, close management | Snowflake, BigQuery, Databricks |
| `customer-support` | Ticket triage, draft responses, KB articles | Intercom, HubSpot, Jira, Guru |
| `enterprise-search` | Find anything across email, chat, docs, wikis | Slack, Notion, Jira, Asana |
| `bio-research` | Literature search, genomics, target prioritization | PubMed, ClinicalTrials.gov, ChEMBL |
| `cowork-plugin-management` | Create or customize plugins | — |

---

## Building Your Own Plugin

Plugins are just markdown and JSON — no code, no build step.

**Step 1 — Manifest:**
```json
// .claude-plugin/plugin.json
{
  "name": "de-toolkit",
  "version": "1.0.0",
  "description": "Data engineering toolkit for pipeline debugging and metric analysis."
}
```

**Step 2 — Add a skill:**
```markdown
// skills/pipeline-debug/SKILL.md
---
name: pipeline-debug
description: Debug a failing pipeline. Use when a DAG fails, a table is late, or data quality drops.
---

## Workflow
1. Classify: upstream missing data / logic error / resource exhaustion / flaky dependency
2. What was the last successful run? What changed since?
3. Downstream impact: who consumes this table and what SLA are they on?
4. Remediation: backfill scope, communication needed, prevention steps
```

**Step 3 — Wire tools (optional):**
```json
// .mcp.json
{
  "mcpServers": {
    "bigquery": { "type": "http", "url": "https://bigquery.googleapis.com/mcp" }
  }
}
```

**Step 4 — Install:**
```bash
claude plugin install ./de-toolkit
```

---

## When to Build a Skill vs a Plugin

**Skill** — when it's personal and needs no external connections:
- A prompt pattern you repeat (interview prep, SQL review, metric framework)
- Single person, local Claude Code session

**Plugin** — when it's for a team or needs live data:
- You want consistent behavior across multiple engineers
- You need Claude to query real systems (warehouse, ticketing, comms)
- You're packaging domain expertise to install and share
