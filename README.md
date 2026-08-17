# Selfship skills

Customer-installable skills for [Selfship](https://selfship.ai): instrument a repo, discover agentic workflows, and reason about the product-facet taxonomy — from your own coding agent.

These skills edit (or read) **your** tree. They do not open a GitHub PR and they do not enqueue hosted first-PR. Pair them with the hosted MCP at `https://mcp.selfship.ai` (mint an `ssa_` secret in Selfship **Settings → your org → Agent**).

[![skills.sh](https://skills.sh/b/selfship-ai/selfship-skills)](https://skills.sh/selfship-ai/selfship-skills)

## Install

```bash
npx skills add selfship-ai/selfship-skills
```

That is the whole install. The [skills CLI](https://github.com/vercel-labs/skills) copies the three skills into the agents on this machine (Cursor, Claude Code, GitHub Copilot / VS Code, and others).

| Flag | When |
| --- | --- |
| (none) | Project install — run it in the repo you want to instrument |
| `-g` | Every workspace on this machine |
| `-y` | Skip prompts (CI / already know the agents) |

```bash
npx skills add selfship-ai/selfship-skills -g
```

Then add MCP ([docs](https://docs.selfship.ai/mcp-server)). Skills without MCP still work if you pass a local inventory or name the workflows yourself.

## The three skills

| Skill | Side effects |
| --- | --- |
| `selfship-instrument` | Edits the local tree, then **stops** |
| `selfship-explore-workflows` | Read-only inventory. Must not call `workflows.upsert` |
| `selfship-product-facets` | Read + propose. No MCP write |

Canonical folders in this repo: `instrument/`, `explore-workflows/`, `product-facets/`.

## MCP

Hosted streamable HTTP. Auth is `X-SelfShip-Org-ID` + `X-SelfShip-Org-Secret` (`ssa_…`), or HTTP Basic `org_id:org_secret`. A repo ingest `bps_` / `SELFSHIP_REPO_SECRET` is rejected.

Cursor / Claude Code (`mcp.json` or Claude config):

```json
{
  "mcpServers": {
    "selfship": {
      "url": "https://mcp.selfship.ai",
      "headers": {
        "X-SelfShip-Org-ID": "${SELFSHIP_ORG_ID}",
        "X-SelfShip-Org-Secret": "${SELFSHIP_ORG_SECRET}"
      }
    }
  }
}
```

Visual Studio Code (Command Palette → **MCP: Open User Configuration**, or `.vscode/mcp.json`). Top-level key is `servers`, not `mcpServers`:

```json
{
  "inputs": [
    {
      "id": "selfship-org-secret",
      "type": "promptString",
      "description": "Selfship Agent secret (ssa_…)",
      "password": true
    }
  ],
  "servers": {
    "selfship": {
      "type": "http",
      "url": "https://mcp.selfship.ai",
      "headers": {
        "X-SelfShip-Org-ID": "<org_id>",
        "X-SelfShip-Org-Secret": "${input:selfship-org-secret}"
      }
    }
  }
}
```

`plugins/selfship/mcp.json` and `plugins/vscode/mcp.json` are the same snippets without secrets.

## Other install paths

Use these only if you cannot run `npx`.

| Agent | Alternative |
| --- | --- |
| Claude Code | `/plugin marketplace add selfship-ai/selfship-skills` then `/plugin install selfship@selfship-skills` |
| Cursor | Import `https://github.com/selfship-ai/selfship-skills` as a plugin / Team marketplace source (`plugins/selfship/`) |
| VS Code (Copilot agent mode) | `npx skills add selfship-ai/selfship-skills -a github-copilot` — folder names must stay `selfship-*` |

## Versioning

Git tags on this repo (`v0.x.y`). Marketplace `plugin.json` versions match the tag.
