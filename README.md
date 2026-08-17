# Selfship skills

Customer-installable skills for [Selfship](https://selfship.ai): instrument a repo, discover agentic workflows, and reason about the product-facet taxonomy — from your own coding agent.

These skills edit (or read) **your** tree. They do not open a GitHub PR and they do not enqueue hosted first-PR. Pair them with the hosted MCP at `https://mcp.selfship.ai` (mint an `ssa_` secret in Selfship **Settings → Agent**).

Canonical folders (copy-paste these):

| Folder | Skill | Side effects |
| --- | --- | --- |
| `instrument/` | `selfship-instrument` | Edits the local tree, then **stops** |
| `explore-workflows/` | `selfship-explore-workflows` | Read-only inventory. Must not call `workflows.upsert` |
| `product-facets/` | `selfship-product-facets` | Read + propose. No MCP write |

<a id="cursor"></a>

## Cursor

**Plugin (GitHub URL):** import `https://github.com/selfship-ai/selfship-skills` as a Cursor plugin / Team marketplace source. The plugin lives in `plugins/selfship/`.

**Raw copy:** copy `instrument/`, `explore-workflows/`, and `product-facets/` into `.cursor/skills/` (project) or `~/.cursor/skills/` (user).

MCP snippet (Settings → Agent also shows this):

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

<a id="claude-code"></a>

## Claude Code

```text
/plugin marketplace add selfship-ai/selfship-skills
/plugin install selfship@selfship-skills
```

Or copy the three folders into `~/.claude/skills/` / `.claude/skills/`.

## MCP

Hosted streamable HTTP. Auth is `X-SelfShip-Org-ID` + `X-SelfShip-Org-Secret` (`ssa_…`), or HTTP Basic `org_id:org_secret`. A repo ingest `bps_` / `SELFSHIP_REPO_SECRET` is rejected.

`plugins/selfship/mcp.json` points at the URL only — no secrets.

## Versioning

Git tags on this repo (`v0.x.y`). Marketplace `plugin.json` versions match the tag.
