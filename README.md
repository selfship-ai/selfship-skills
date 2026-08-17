# Selfship skills

Customer-installable skills for [Selfship](https://selfship.ai): instrument a repo, discover agentic workflows, and reason about the product-facet taxonomy — from your own coding agent.

These skills edit (or read) **your** tree. They do not open a GitHub PR and they do not enqueue hosted first-PR. Pair them with the hosted MCP at `https://mcp.selfship.ai` (mint an `ssa_` secret in Selfship **Settings → Agent**).

Canonical folders (copy-paste these):

| Folder | Skill | Side effects |
| --- | --- | --- |
| `instrument/` | `selfship-instrument` | Edits the local tree, then **stops** |
| `explore-workflows/` | `selfship-explore-workflows` | Read-only inventory. Must not call `workflows.upsert` |
| `product-facets/` | `selfship-product-facets` | Read + propose. No MCP write |

| Agent | Install |
| --- | --- |
| [Cursor](#cursor) | Plugin from this GitHub URL, or copy the three canonical folders |
| [Visual Studio Code](#vscode) | Copilot agent mode — copy `plugins/vscode/skills/selfship-*` (names must stay `selfship-*`) |
| [Claude Code](#claude-code) | `/plugin marketplace add selfship-ai/selfship-skills` |

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

<a id="vscode"></a>

## Visual Studio Code

Requires [GitHub Copilot](https://code.visualstudio.com/docs/copilot/overview) **agent mode**. Plain VS Code without Copilot will not load skills.

Copilot requires the skill **folder name** to match `name:` in `SKILL.md`. Use the wrappers in `plugins/vscode/skills/` (`selfship-instrument`, `selfship-explore-workflows`, `selfship-product-facets`) — do not copy the canonical `instrument/` folder names as-is.

**Skills (pick one):**

| Install | What to do |
| --- | --- |
| Repo (share with the team) | Copy `plugins/vscode/skills/selfship-*` into `.github/skills/` |
| User (every workspace) | Copy those same three folders into `~/.copilot/skills/` |
| Clone in place | Clone this repo and add `plugins/vscode/skills` to `chat.agentSkillsLocations` |

`plugins/vscode/package.json` is a contribution-only extension (`chatSkills`). Package with `vsce` when publishing to the Marketplace; until then use the copy path above.

**MCP:** Command Palette → `MCP: Open User Configuration`, or add `.vscode/mcp.json`. VS Code uses a top-level `servers` key (not `mcpServers`):

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

Or copy `plugins/vscode/mcp.json` and set `SELFSHIP_ORG_ID` / `SELFSHIP_ORG_SECRET` in the environment. Mint the `ssa_` secret in Selfship **Settings → Agent**.

Switch Copilot Chat to **Agent**, then ask to instrument the repo, explore workflows, or propose product facets.

<a id="claude-code"></a>

## Claude Code

```text
/plugin marketplace add selfship-ai/selfship-skills
/plugin install selfship@selfship-skills
```

Or copy the three folders into `~/.claude/skills/` / `.claude/skills/`.

## MCP

Hosted streamable HTTP. Auth is `X-SelfShip-Org-ID` + `X-SelfShip-Org-Secret` (`ssa_…`), or HTTP Basic `org_id:org_secret`. A repo ingest `bps_` / `SELFSHIP_REPO_SECRET` is rejected.

`plugins/selfship/mcp.json` is the Cursor / Claude snippet. `plugins/vscode/mcp.json` is the VS Code snippet. Both point at the URL only — no secrets.

## Versioning

Git tags on this repo (`v0.x.y`). Marketplace `plugin.json` versions match the tag.
