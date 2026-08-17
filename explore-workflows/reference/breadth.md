---
name: explore-workflows
description: >-
  Explore a repository to enumerate all agentic workflows — multi-turn chats,
  one-shot agentic calls, non-interactive pipelines, batched jobs. Produces a
  shallow structured inventory with framework, product objective, key files,
  coarse tools/paths, and rough mermaid flow diagrams. Use for the Exploration
  (breadth) job (read-only, no PR). Depth analysis is a separate skill.
disable-model-invocation: true
---

# Explore Workflows (Breadth)

Read-only repository analysis. **Do not modify any files.**

This skill is for **breadth**: find every agentic workflow and produce a shallow inventory.
Do **not** exhaustively map every tool, code path, upstream/downstream contract, or cost signal —
that is the `explore-workflow-depth` skill / depth job.

## Workflow types

| Type | Examples |
|---|---|
| `multi_turn_chat` | Chat UI, threaded conversations with forks |
| `one_shot_agentic` | Single request → agent → response |
| `non_interactive` | Pipelines, ETL, background processors |
| `batched_agentic` | Cron, queue workers processing many items |

## Framework identification

Consult bundled references:
- `reference/langfuse_integration_model.md` — how Langfuse integrates per framework
- `reference/selfship_integrations.md` — SelfShip parity matrix

You may fetch `langfuse.com/integrations/frameworks/*` for confirmation.

Common frameworks: `langchain`, `langgraph`, `openai`, `anthropic`, `vercel_ai`, `autogen`, `crewai`, `llamaindex`, `pydantic_ai`, `native` (direct API calls).

## Inventory schema (per workflow)

| Field | Required | Notes |
|---|---|---|
| `workflow_key` | yes | Stable slug; becomes trace tag `workflow:<key>` |
| `name` | yes | Human-readable |
| `description` | yes | What this workflow does |
| `type` | yes | One of the four types above |
| `framework` | yes | Primary agent framework |
| `product_objective` | yes | Inferred product goal |
| `files` | yes | Key entry-point / agent-loop paths (not exhaustive) |
| `tools` | yes | Coarse `{ name, description? }[]` |
| `paths` | yes | High-level branches (`{ name, description? }[]` or strings) |
| `flow_mermaid` | yes | Rough mermaid diagram that **must render** as `flowchart`/`graph` (depth will refine) |
| `priority_rank` | yes | 1 = highest product value |

Do **not** emit `architecture` — that field is depth-only.

## Quality gates (checker will reject otherwise)

- Mermaid must compile/render (`flowchart` or `graph`).
- Every path in `files` must exist in the checked-out repo.

## Procedure

1. Read README, package manifests, entry points.
2. Search for LLM clients, agent frameworks, tool definitions, API routes, workers.
3. Group related code into workflows (one workflow = one product surface / agent loop).
4. Capture coarse tools and major branch paths — do not chase every transitive call.
5. Rank by product value (`priority_rank`).
6. Return JSON per `task-explore.md` contract.

## Checklist

- [ ] ALL workflows enumerated (not just top 3).
- [ ] Each workflow has `workflow_key`, `files`, `tools`, `paths`, `flow_mermaid`.
- [ ] Framework identified per workflow.
- [ ] `priority_rank` assigned (1 = best).
- [ ] Mermaid renders; listed files exist.
- [ ] Breadth only — no `architecture`, no exhaustive depth analysis.
- [ ] No file modifications.
