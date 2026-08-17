---
name: selfship-instrument
description: >-
  Add or update Selfship observability in this repo: initial init + workflow
  tags, sync selected/deselected workflows, verify/fix gaps, or upgrade the
  Selfship SDK. Edits the local tree and stops. Does not commit, push, open a
  PR, or enqueue hosted first-PR.
disable-model-invocation: false
---

# Selfship instrument

Local instrumentation for the repo you have open. **Edit files, then stop.**

Do **not** `git commit`, `git push`, create a branch for the user, open a GitHub PR, or call any MCP tool that would enqueue hosted first-PR. If a PR is needed, the owner does that later themselves.

Load **`reference/body.md`** (modes, hard rules 1–19, checklist) and the mechanics file for this repo's language:

- Python → `reference/python-mechanics.md`
- TypeScript → `reference/typescript-mechanics.md`

Load **`reference/langfuse_best_practices.md`** for sessions, identity, tags, metadata, observation types, cost, feedback, releases.

Framework tables: `reference/selfship_integrations.md` and `reference/langfuse_integration_model.md`.

## Credentials (two different secrets)

| Where | Variables | Secret class |
| --- | --- | --- |
| **App runtime** (what you write into `.env.example`) | `SELFSHIP_ORG_ID` + `SELFSHIP_REPO_SECRET` | Ingest (`bps_…`) |
| **Your coding agent / MCP** | `SELFSHIP_ORG_ID` + `SELFSHIP_ORG_SECRET` | Agent API (`ssa_…`) |

Never put an `ssa_` secret in application code. Never use a `bps_` ingest secret as MCP auth.

## Modes

Pick one. State it before editing.

| Mode | Job |
| --- | --- |
| `initial` | Global `init()` + instrument selected workflows |
| `sync` | Add newly selected, remove deselected, verify already-instrumented |
| `verify` / `fix` | Audit `to_verify` + apply minimal fixes |
| `sdk_upgrade` | Bump the declared SDK dep to the contract floor and apply `migration-brief.json` call-site actions |

Contract floor (keep in sync with Selfship `SELFSHIP_SDK_MIN_VERSION`): **0.3.9** for both `selfship-ai` (PyPI) and `@selfship-ai/sdk` (npm). Range: `>=0.3.9,<0.4.0`.

## MCP tools this skill may call

When MCP is configured:

- `orgs.list` — confirm account
- `workflows.list` — which keys are selected / `instrumentation_status`
- `workflows.set_selected` — toggle yes/no (does **not** open a PR)
- `product_profile.get` — path-coverage context
- `action_items.list` — filter `source=sdk_dep_upgrade` or lever/title for upgrade work

Do **not** call `workflows.upsert`. Inventory publish is a different skill/act.

If MCP is not configured, read a local inventory the user provides, or instrument what they named.

## SDK upgrade checklist

1. Parse manifests the way Selfship does: `pyproject.toml` / `requirements*.txt` / `package.json` / lockfiles.
2. Compare the declared floor to **0.3.9**. If already in range and call sites match the brief, stop.
3. Load **`reference/migration-brief.json`** (copy of Selfship `sdk/migration-brief.json`). Apply each `call_site_actions[]` whose `detect` matches.
4. Do not invent migrations that are not in the brief.
5. Leave a short report: from→to, files touched, actions applied, actions skipped.

## Stop condition

After local edits (or a no-op verify), **end**. Do not offer to open a PR as a next agent step.
