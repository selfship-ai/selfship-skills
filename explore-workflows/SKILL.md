---
name: selfship-explore-workflows
description: >-
  Discover agentic workflows in this repo (breadth, then optional depth).
  Read-only on the tree. Produces a local inventory. Does not persist to
  Selfship and must not call workflows.upsert.
disable-model-invocation: false
---

# Selfship explore workflows

Read-only repository analysis. **Do not modify any files.**

This is the same product feature as hosted exploration: **breadth**, then optional **depth** on the workflows the user cares about.

## Must not persist

- Do **not** call MCP `workflows.upsert`. That tool requires explicit user consent in a later turn, after you have shown the inventory.
- You may *mention* that upsert exists and needs `confirm=true`.
- You may call `workflows.list` and `product_profile.get` to compare against what Selfship already knows.

## Pass 1 — breadth

Follow **`reference/breadth.md`** (inventory schema, quality gates, types, frameworks).

Consult:

- `reference/langfuse_integration_model.md`
- `reference/selfship_integrations.md`

Output a complete inventory for **every** agentic workflow, not a top-3. No `architecture` on this pass.

## Pass 2 — depth (optional)

Only after the user names which workflows to deepen (or MCP `workflows.list` shows selected keys they want deepened).

Follow **`reference/depth.md`** for each chosen `workflow_key`. One workflow at a time. Still read-only.

## After the inventory

Show the user the JSON. Stop.

If they later say they want it saved to Selfship, that is a **new** instruction: then (and only then) another agent turn may call `workflows.upsert` with `confirm=true`. This skill itself never makes that call.
