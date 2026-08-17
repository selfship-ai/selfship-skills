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

Load **`reference/frameworks.md`** when the repo already emits AI spans, uses LangSmith, LiveKit, Vercel AI, or OpenInference — OTel-first routing, LangSmith OTel recipe, and framework gotchas.

Framework tables: `reference/selfship_integrations.md` and `reference/langfuse_integration_model.md`.

Inspect existing telemetry **before** `init()` / `workflow()` / `langchain_callbacks()`. If a real turn already emits `gen_ai` / `ai` / `llm` / `lk` / LangSmith spans, point OTLP at `otel.selfship.ai` and stop — do not attach a second mechanism. After edits, validate one real turn with `traces.get` / `sessions.get` (inventory tags are not enough).

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
| `initial` | Inspect existing telemetry first. Preserve a usable OTel/LangSmith path; otherwise global `init()` + instrument selected workflows |
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
- `traces.list` / `traces.get` — confirm a real turn landed (full I/O on get). Pass `environment` matching the app; MCP defaults to production, SDK defaults to development
- `sessions.list` / `sessions.get` — confirm stable `conversation_id` / `user_id`. `sessions.get` needs `user_key` plus `session_id` or `trace_ids`

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
