# Langfuse Best Practices (SelfShip)

> How Langfuse organizes observability data, what Langfuse recommends, and how SelfShip maps
> onto it. Companion to [`archi.md`](./archi.md) (tenant provisioning & gateway) and
> [`first_pr.md`](./first_pr.md) / [`pr-runner/prompts/AGENTS.md`](../pr-runner/prompts/AGENTS.md)
> (what we tell the integration agent to instrument).
>
> For the **design spec** on enabling environments, filling every first-class trace attribute, and
> handling threaded-chat-with-forks vs scheduled/batch executions, see
> [`tracing_enrichment.md`](./tracing_enrichment.md).
>
> Langfuse docs referenced throughout:
> [Environments](https://langfuse.com/docs/observability/features/environments),
> [Metadata](https://langfuse.com/docs/observability/features/metadata),
> [Tags](https://langfuse.com/docs/observability/features/tags),
> [Data model](https://langfuse.com/docs/observability/data-model),
> [Observation types](https://langfuse.com/docs/observability/features/observation-types),
> [Sessions](https://langfuse.com/docs/observability/features/sessions),
> [Token & cost tracking](https://langfuse.com/docs/observability/features/token-and-cost-tracking),
> [Managing environments FAQ](https://langfuse.com/faq/all/managing-different-environments).

___

## 1\. Langfuse hierarchy

Langfuse has three layers customers (and we) reason about:

| Layer | What it is | Who manages it in SelfShip |
|---|---|---|
| **Organization** | Top-level tenant: members, billing, one or more projects | **Provisioner** — one Langfuse org per SelfShip org (`name` = SelfShip `org_id`) |
| **Project** | Where traces land; has API keys, prompts, datasets | **Provisioner** — **one Langfuse project per connected repo** (name = repo full name, e.g. `acme/widget-api`) |
| **Environment** | Label on every trace/observation (`production`, `staging`, `development`) | **Customer** — set via SDK env var or OTel resource attribute; we do not provision it |

```text
Langfuse instance (self-hosted, internal)
└── Organization  (1 per SelfShip customer)
    ├── Project "acme/api"       (1 per connected repo)
    │   └── Traces tagged with environment  (customer's deployment stage)
    ├── Project "acme/web"
    │   └── Traces tagged with environment
    └── …
```

Customers never choose org or project in the Langfuse UI — SelfShip provisions both. They authenticate
with **per-repository** credentials (`org_id` + `SELFSHIP_REPO_SECRET`); the gateway resolves the
matched `org_secrets.repo_id` and swaps to that repo's Langfuse project `pk:sk`. **Mixing multiple
repos into one Langfuse project is invalid** — each repo must have its own project. **Environment**
is the knob for separating prod vs staging *within* a repo.

___

## 2\. Organizations & projects — Langfuse recommendations

### 2\.1 When to use one project vs many

__Langfuse's default recommendation__ (for a single application): use __one project__ and separate
deployment stages with __environments__, not separate projects per stage.

SelfShip maps **one Langfuse project per connected repo** — a customer org with three repos has
three Langfuse projects under one Langfuse org. That is intentional: repos are distinct codebases with
their own evaluators, prompts, and trace volume. **Funneling every repo in an org into a single
Langfuse project is invalid.**

| Approach | Use when | Tradeoffs |
|---|---|---|
| **One project + environments** (Langfuse default for one app) | Same codebase, same team; separate prod/staging/dev | Simplest stage filter; shared prompts/datasets |
| **Separate projects per repo** (SelfShip rule) | Multiple repos in one customer org | Hard isolation per codebase; evaluators and costs don't cross-contaminate |
| **Separate projects per stage** | Different RBAC per stage (e.g. prod visible to fewer people) | Prompts/configs don't auto-sync — need GitHub integration or Prompt Management API |
| **Separate orgs** | True multi-tenant isolation, different companies | Full data wall; what SelfShip does **between customers** |

Within each repo project, use **environments** (not extra projects) to separate prod vs staging.

### 2\.2 What SelfShip does (and doesn't)

__We do:__

- **One Langfuse org per SelfShip org** — hard tenant isolation; Langfuse org name = SelfShip
  `org_id` for easy cross-reference in the internal Langfuse UI.
- **One Langfuse project per connected repo** — provisioned when the repo is linked (name =
  `owner/repo`, e.g. `acme/widget-api`); each project gets its own API key pair in Langfuse.
- **Gateway credential swap per repo** — customer sends `org_id:bps_repo_secret`; gateway verifies
  the secret, reads `repo_id` from the matched `org_secrets` row, and swaps to that repo's Langfuse
  `pk:sk`. Customers never see Langfuse keys. `SELFSHIP_REPO` / `metadata.repo` are optional enrichment only.
- **Enforce repo on ingest** — secrets with `repo_id IS NULL` (legacy org-wide) are rejected after
  migration; unprovisioned repos return `403 repo not ready`.

__We don't:__

- Use a single catch-all `"default"` project for an entire org — **invalid** when the org has
  multiple repos.
- Treat `metadata.repo` as a substitute for project isolation — metadata supports dashboard
  filtering; the Langfuse **project** is the hard boundary for storage, evaluators, and billing roll-ups.
- Expose Langfuse UI to customers.

__Guidance for customers:__

- Use **one SelfShip org** for all repos and all deployment stages.
- Set `SELFSHIP_REPO_SECRET` (per connected repo) so ingest lands in that repo's Langfuse project.
  Optionally set `SELFSHIP_REPO=owner/repo` for `metadata.repo` enrichment (dashboard filters) —
  it does **not** select the project.
- Set `LANGFUSE_TRACING_ENVIRONMENT=production` (or `staging`, `development`) per deployment to
  separate stages **within** that repo's project.
- Do **not** create separate SelfShip orgs just to separate environments or repos — that fragments
  billing, keys, and dashboards.

___

## 3\. Environments

### 3\.1 How to set

__Python / JS SDK (recommended):__

```bash
LANGFUSE_TRACING_ENVIRONMENT=production
```

Or at client init: `Langfuse(environment="staging")`. Init parameter wins over the env var.

__OpenTelemetry (no SDK):__

```bash
OTEL_RESOURCE_ATTRIBUTES=langfuse.environment=staging
```

__Rules (Langfuse-enforced):__

- Lowercase alphanumeric + `-` / `_` only
- Max 40 characters
- Cannot start with `langfuse`
- Default is `default` if unset
- Environments are created on first ingest and are persistent (no UI rename/delete today)

### 3\.2 Best practices

1. **Consistent names** across services — `production`, `staging`, `development` (not `prod` in
   one service and `production` in another).
2. **Filter in the Langfuse UI** — environment is a global filter across traces, sessions, scores.
3. **Use for stage comparison** — same repo project, different environment → compare latency/cost/errors
   without duplicating prompts or datasets across repos.
4. **Document in `.env.example`** — first-PR integrations should add
   `LANGFUSE_TRACING_ENVIRONMENT=production` (or leave a comment that staging deployments should
   override it).

___

## 4\. Trace data model

Langfuse organizes data into three core layers: **sessions → traces → observations**.

- A **trace** is one end-to-end unit of work (typically one chat turn or one HTTP request). It
  contains a **tree of observations**. Trace attributes (`user_id`, `session_id`, `tags`,
  `metadata`) are propagated down to all observations within the trace.
- An **observation** is one step inside a trace. Observations can be nested.
- A **session** groups multiple traces that are part of the same user interaction (a chat thread).
  See [§4.2](#42-sessions).
- A **user** links traces to an end-user via `user_id`. See [§4.4](#44-user-tracking).
- **Scores** attach evaluations (incl. user feedback) to a trace/observation/session. See
  [§4.5](#45-user-feedback).
- **Releases & versions** tag which build / component version produced a trace. See
  [§4.6](#46-releases--versioning).

See [Langfuse data model](https://langfuse.com/docs/observability/data-model).

### 4\.1 Observation types

Langfuse supports a set of LLM-application-specific observation types. Picking the right type gives
you better filtering, icons, and automatic cost/usage roll-ups in the UI.

| Type | Typical use | Tracks token usage & cost? |
|---|---|---|
| `event` | Point-in-time marker (basic building block) | No |
| `span` | Duration of a unit of work (orchestration, parsing) | No |
| `generation` | LLM/model call (prompt, completion, tokens, cost) | **Yes** |
| `embedding` | Call to an embedding model | **Yes** |
| `agent` | LLM-driven control flow that orchestrates tools | No |
| `tool` | A tool / function / external API call (e.g. weather API) | No |
| `chain` | Link between steps (e.g. retriever → LLM) | No |
| `retriever` | Data retrieval (vector store, DB, search) | No |
| `evaluator` | Assesses relevance / correctness / helpfulness of output | No |
| `guardrail` | Protects against malicious content or jailbreaks | No |

__Only `generation` and `embedding` carry `usage_details` / `cost_details`__ — see
[§4.3](#43-token--cost-tracking). Every other type is a plain span for structure and filtering.

__Version requirements:__ the semantic types beyond `event` / `span` / `generation` require Python
SDK `>=3.3.1` and TypeScript SDK `>=4.0.0`. On older SDKs they degrade to plain spans.

__How to set the type:__

- **Automatic** — agent-framework integrations set the type for you (e.g. a LangChain method marked
  `@tool` becomes a Langfuse `tool` observation; native `langfuse.openai` / `langfuse.langchain`
  create `generation` observations with token usage).
- **Manual (Python)** — `@observe(as_type="tool")` or
  `start_as_current_observation(as_type="tool", ...)`:

  ```python
  from langfuse import observe

  @observe(as_type="tool")
  def call_weather_api(location: str):
      return weather_service.get_weather(location)
  ```

- **Manual (TypeScript)** — `startActiveObservation(name, fn, { asType: "tool" })`,
  `startObservation(name, data, { asType: "tool" })`, or `observe(fn, { asType: "tool" })`.

__SelfShip mapping:__ `begin()` / `track()` open plain __spans__. For tool calls, use
`@observe(as_type="tool")` (Python) or the `asType` option on the TypeScript SDK. See
[Observation types](https://langfuse.com/docs/observability/features/observation-types).

### 4\.2 Sessions

A **session** groups multiple traces that belong to the same multi-turn interaction (e.g. a chat
thread) so you can view a single **session replay** of the whole conversation. The relationship is
`1 session : n traces`, keyed by `session_id`.

__Rules (Langfuse-enforced):__

- `session_id` is any US-ASCII string **under 200 characters**; longer values are dropped.
- All traces sharing a `session_id` are grouped, including their child observations.
- Sessions are created implicitly on first ingest — there is nothing to provision.

__How to set (Python SDK v4):__ propagate `session_id` so every trace and child observation in the
conversation inherits it:

```python
from langfuse import propagate_attributes

with propagate_attributes(session_id="chat-session-123", user_id=user_id):
    # every trace/observation created in this block joins the session
    ...
```

__TypeScript:__ wrap work in `propagateAttributes({ sessionId: "chat-session-123" }, async () => { … })`.

__SelfShip mapping:__ pass your `conversation_id` — SelfShip maps it to Langfuse `session_id`. Reuse
the **same value across every turn** of a chat so the turns land in one session. Do not fabricate a
value when there is no real conversation grouping (a one-shot request needs no session).

__Session-level scores:__ you can attach `scores` to a whole session (not just a trace) — via the
Langfuse UI (human annotation) or programmatically via the SDK/API — for conversation-level QA,
user-feedback, or moderation signals. Sessions can also be bookmarked and shared as public links.

See [Sessions](https://langfuse.com/docs/observability/features/sessions).

### 4\.3 Token & cost tracking

Usage and cost are tracked **only on `generation` and `embedding` observations**, broken down by
__usage type__ (`input`, `output`, `cached_tokens`, `audio_tokens`, …) and mirrored by __cost
details** in USD. There are two ways values arrive:

1. **Ingested** (preferred, most accurate) — pass `usage_details` and optionally `cost_details`
   from the provider response.
2. **Inferred** — if usage or cost is missing, Langfuse infers it **at ingestion time** from the
   `model` name using its built-in tokenizers and model price list.

__Ingested values always win over inferred values.__ If usage is present but cost is not, Langfuse
computes cost from the model's per-usage-type prices.

__Ingesting (Python SDK v4):__

```python
generation.update(
    usage_details={
        "input": response.usage.input_tokens,
        "output": response.usage.output_tokens,
        "cache_read_input_tokens": response.usage.cache_read_input_tokens,
        # "total": int,  # optional; derived as the sum of all buckets if omitted
    },
    cost_details={           # optional — omit to let Langfuse infer from the model price
        "input": 1,
        "output": 1,
        "cache_read_input_tokens": 0.5,
    },
)
```

TypeScript uses the same shape with camelCase keys: `usageDetails` / `costDetails`.

__Usage types are mutually-exclusive buckets__ — this is the most common footgun:

- Each token must be counted in exactly **one** key. `input` must **exclude** anything already in an
  `input_*` bucket (e.g. `input_cached_tokens`); `output` must exclude `output_*` buckets (e.g.
  `output_reasoning_tokens`). `total` is the one exception — it spans all buckets and equals their sum.
- The UI sums every key containing `input` as input usage and every key containing `output` as
  output usage; if buckets overlap, tokens are **counted twice** and inferred cost is overstated.
- Many providers report **inclusive** counts. OpenAI's `prompt_tokens` / `input_tokens` *include*
  cached tokens; Anthropic's `input_tokens` already *excludes* cache reads/writes. If you write your
  own flat-key instrumentation, subtract the detail counts from the top-level count first.
- **When Langfuse normalizes for you:** native integrations/SDK wrappers, OpenTelemetry
  `gen_ai.usage.*` / `llm.token_count.*` attributes, and the nested OpenAI usage schema are all
  converted to exclusive buckets on ingest. **Langfuse-style flat keys** (`usage_details` /
  `langfuse.observation.usage_details`) are stored **verbatim** — you must pre-subtract.

__Inference details:__

- Built-in tokenizers cover OpenAI (`o200k_base` / `cl100k_base` via `tiktoken`) and Anthropic
  (`@anthropic-ai/tokenizer`).
- **Reasoning models (o1 family) cannot have cost inferred** without ingested token counts —
  Langfuse can't see hidden reasoning tokens. Always ingest usage for reasoning models (native
  wrappers do this for you).
- **Custom / self-hosted models:** add a model definition (UI under *Project Settings → Models*, or
  the `/api/public/models` API) with a `match_pattern` regex and per-usage-type prices. User-defined
  models take priority over Langfuse-maintained ones. Model definitions also support **pricing
  tiers** (e.g. higher price above 200K input tokens). Changing a definition only affects
  __new__ generations.

__SelfShip note:__ because SelfShip is a thin wrapper, native Langfuse integrations
(`langfuse.openai`, `langfuse.langchain`, `langfuse.llama_index`) capture usage and cost
automatically — prefer them over hand-rolled instrumentation so buckets and normalization are
correct. Retrieve aggregated usage/cost for billing or analytics via the Langfuse Metrics API.

See [Model usage & cost tracking](https://langfuse.com/docs/observability/features/token-and-cost-tracking).

### 4\.4 User tracking

Setting `user_id` links traces to an end-user and powers the Langfuse **Users** view — an overview
of all users plus a per-user drill-down that aggregates **token usage, cost, trace counts, and
feedback** for a single identity.

__How to set:__

- `user_id` can be a username, email, or any unique identifier — but keep it **≤ 200 characters**
  (same limit as other first-class fields) and avoid PII you don't want in your telemetry store.
- Propagate it so every trace and child observation in the unit of work inherits it:

  ```python
  from langfuse import propagate_attributes

  with propagate_attributes(user_id="user_12345", session_id=conversation_id):
      # every trace/observation created here is attributed to the user
      ...
  ```

  TypeScript: `propagateAttributes({ userId: "user-123" }, async () => { … })`.
- Native integrations pick it up from the same `propagate_attributes` / `propagateAttributes` block
  (OpenAI wrapper, LangChain `CallbackHandler`, etc.).

__Best practices:__

- Set identity **once at the root** and let it propagate — don't set `user_id` on individual child
  spans.
- **Never fabricate** a `user_id`; omit it when identity is unknown (e.g. anonymous/unauthenticated
  requests). It is optional.
- Don't duplicate it into `metadata.user_id` — `user_id` is first-class and filterable on its own.
- Per-user metrics (cost, tokens, trace counts) are queryable programmatically via the Langfuse
  __Metrics API__ and can be charted in custom dashboards. The individual user view deep-links as
  `https://{host}/project/{projectId}/users/{userId}`.

__SelfShip mapping:__ pass `user_id` on `begin` / `track` / `@observe` (see [§5](#5-trace-attributes--what-goes-where)
and [§7.1](#71-identity)); wire it from your auth middleware / session, not a hard-coded value.

See [User tracking](https://langfuse.com/docs/observability/features/users).

### 4\.5 User feedback

User feedback tells you whether the AI actually helped. In Langfuse it is captured as a **score**
linked to a trace (or observation / session). **Many customer apps already have a feedback UI**
(thumbs up/down, star ratings, "was this helpful?", copy/retry buttons) — the goal is to **leverage
that existing signal** by forwarding it into Langfuse as a score, not to build new UI.

__Two flavors, both stored as scores:__

- **Explicit** — the user directly rates a response (thumbs up/down, stars, comment). Clear signal,
  but low volume and biased toward unhappy users.
- **Implicit** — derived from behavior (copied the output, accepted a suggestion, retried, closed a
  support ticket). High volume, no user effort, but noisier to interpret.

__Server-side only (required):__ SelfShip instrumentation and scoring must stay on the **server**.
Never ship SelfShip credentials (including `SELFSHIP_REPO_SECRET` / `SELFSHIP_ORG_ID`) to the
browser, and never use `NEXT_PUBLIC_*` for SelfShip. Return the trace id to the frontend if needed
for UX, then submit the score from a backend route / worker when the feedback arrives:

```python
from langfuse import get_client
langfuse = get_client()

langfuse.create_score(
    trace_id=trace_id,
    name="user-feedback",
    value=1,                       # 1 = 👍, 0 = 👎
    comment="Closed after first response",
)
```

(or the TypeScript SDK equivalent via `getClient()` on a server route).

__Also for implicit / delayed feedback:__ record a score from the backend when you learn the outcome
(survey result, ticket closed, task completed). You need the trace id you captured at request time:

```python
from langfuse import get_client
langfuse = get_client()

langfuse.create_score(
    trace_id=trace_id,
    name="ticket-resolution",
    value=1,                       # or 0 on escalation
    comment="Closed after first response",
)
```

__Notes:__

- Feedback scores can target a whole **session** (conversation-level feedback), not just one trace —
  see [§4.2](#42-sessions).
- Filter by score in the UI (e.g. `user-feedback` less than 1) to surface low-rated responses, feed
  annotation queues, or build eval datasets from real outcomes.
- For automatic, high-volume feedback, layer LLM-as-a-Judge evaluators on top of the same score
  mechanism.
- **SelfShip caveat:** scoring hits the Langfuse `/api/public/scores` endpoint (not the OTel trace
  path). Confirm the gateway routes scores for your tenant before promising feedback wiring;
  server-side scoring via `get_client()` already works wherever the SDK is initialized.

See [User feedback](https://langfuse.com/docs/observability/features/user-feedback).

### 4\.6 Releases & versioning

Two related knobs let you attribute metric changes (cost, latency, quality) to code or component
changes — the basis for **A/B tests in production** and answering "why did latency/cost move?".

| Attribute | Scope | What it tracks | Typical value |
|---|---|---|---|
| `release` | Whole app / SDK client | The overall build that produced the trace | Semver or git commit SHA |
| `version` | A single observation (by `name`) | The version of one component (prompt, chain, tool) | `"1.0"`, `"1.1"`, … |

__`release` — set once per deployment.__ The SDK resolves it in this order:

1. SDK init parameter (`Langfuse(release="v2.1.24")`).
2. `LANGFUSE_RELEASE` env var — best for CI/CD; set it to the git SHA or release tag.
3. Auto-detected on Vercel, Heroku, Netlify (falls back to their built-in release env vars).

__`version` — per-observation.__ Supported on __all__ observation types. Propagate it to a whole
subtree, or set it on one observation:

```python
from langfuse import propagate_attributes

with propagate_attributes(version="1.1"):
    # every observation in this block is tagged version="1.1"
    ...

# or on a single observation:
with langfuse.start_as_current_observation(as_type="generation", name="guess-countries", version="1.1"):
    ...
```

TypeScript: `propagateAttributes({ version: "1.1" }, async () => { … })`, or
`observation.update({ version: "1.1" })`. LangChain: `CallbackHandler(version="1.1")`.

__SelfShip guidance:__ set `LANGFUSE_RELEASE` in the deployment env (alongside
`LANGFUSE_TRACING_ENVIRONMENT`) so every trace carries the build — cheap to wire in CI and the most
useful of the two for regression triage. Reserve `version` for components you iterate on
deliberately (a specific prompt or chain) so you can compare metrics across its versions. First-PR
`.env.example` should include a commented `LANGFUSE_RELEASE=` line.

See [Releases & versioning](https://langfuse.com/docs/observability/features/releases-and-versioning).

___

## 5\. Trace attributes — what goes where

Langfuse has **first-class attributes**. Use the right one; don't stuff everything into metadata.

| Attribute | Purpose | Filterable in UI | SelfShip API |
|---|---|---|---|
| `user_id` | End-user identity | Yes (Users view) | `user_id` on `begin` / `track` / `@observe` |
| `session_id` | Multi-turn conversation grouping | Yes (Sessions view) | `conversation_id` |
| `environment` | Deployment stage | Yes (global filter) | `LANGFUSE_TRACING_ENVIRONMENT` |
| `tags` | Categorical labels | Yes | `tags=` / `tags:` on `workflow()` (preferred) |
| `metadata` | Arbitrary key-value context | Yes (on root trace) | `metadata=` / `metadata:` on `workflow()` (preferred); include `release` here and/or set `LANGFUSE_RELEASE` |
| `name` | Trace/span display name | Yes | `agent_name` on `begin` / `track`; `@observe(name=...)` |
| `input` / `output` | Request/response payloads | Yes | `begin(input=...)`, `end(output=...)`, generations |
| `release` | Deploy / build attribution | Yes | **Not** a `workflow()` kwarg — `metadata["release"]` and/or `LANGFUSE_RELEASE` |

### 5\.1 Tags vs metadata

| Use **tags** for | Use **metadata** for |
|---|---|
| Feature names (`support-bot`, `rag`) | Model name, retriever id, token counts |
| API surface (`endpoint:chat`, `v2-prompt`) | Request id, tenant slug, experiment variant |
| Workflow labels you filter on often | Structured context that varies per observation |

- **Tags:** strings, max 200 chars each; multiple per observation; aggregated up to the trace.
- **Metadata:** key-value; keys alphanumeric; **values must be strings ≤ 200 chars** (longer values
  are dropped).

These size limits apply to **metadata and tags only** — never to root `input` / `output` (see §7.4).
Truncate or hash large values client-side for metadata/tags; do **not** truncate the user-visible
deliverable recorded as root output.

Both are **immutable after ingest** — cannot edit in the UI later.

### 5\.2 Metadata best practices

1. **Attach filterable metadata to the root trace.** The Langfuse UI filters on root-trace
   attributes. In Langfuse Python v4 (OTel-based), metadata on deep child spans may not appear in trace-level
   filters unless propagated.

2. **Use `propagate_attributes()` for nested paths (Python SDK v4).**

   ```python
   from langfuse import propagate_attributes

   with propagate_attributes(
       metadata={"model": "gpt-4o", "request_id": "req_abc"},
       tags=["support-bot"],
   ):
       # every child observation inherits metadata + tags
       ...
   ```

   Set identity in the same block when possible:

   ```python
   with propagate_attributes(user_id=user_id, session_id=conversation_id):
       ...
   ```

   `propagate_attributes` applies **forward only** — observations created before the block do not
   get retroactive metadata.

   **Thread / process boundaries (Langfuse):** OTel context and Python `contextvars` are
   thread-local. Attributes and active spans do **not** automatically cross worker threads
   (`ThreadPoolExecutor`, `to_thread`, custom thread pools) or processes. Langfuse documents this
   limitation for the `@observe` decorator and the same constraint applies to
   `propagate_attributes`. Remediation: run LLM work in the same context that opened the root,
   enable OpenTelemetry `ThreadingInstrumentor`, or explicitly copy context
   (`contextvars.copy_context()` + `context.run(...)`). Cross-**service** linking uses
   `as_baggage=True` / distributed trace IDs — that is a different problem than in-process threads.

3. **Or set explicitly on the current observation:**

   ```python
   with propagate_attributes(metadata={"feature": "search"}):
       ...
   span.update(metadata={"stage": "parsing"})  # observation-only
   ```

   Do **not** call `update_current_trace()` — that method was removed in Langfuse Python v4.
   Correlating attributes belong on `propagate_attributes()`. Trace-level I/O is handled
   inside `workflow()`; do not reimplement it via `get_client()`.

4. **Don't duplicate first-class fields.** Use `user_id` / `session_id` directly, not
   `metadata.user_id`.

5. **Keep values short.** Truncate or hash large ids client-side if they exceed 200 characters.

6. **LangChain / LangGraph caveat (SDK v4):** metadata in `RunnableConfig` may land on child
   observations only. Use `RunnablePassthrough` at the chain root or `propagate_attributes` so
   filterable keys reach the root trace. See
   [Langfuse #7897](https://github.com/langfuse/langfuse/issues/7897).

### 5\.3 Tags best practices

1. Prefer `propagate_attributes(tags=[...])` for trees (same propagation rules as metadata).
2. Use a small, consistent vocabulary — e.g. `["rag", "production"]`, not one-off sentence tags.
3. For OpenAI integration without a parent span: `metadata={"langfuse_tags": ["tag-1"]}`.

___

## 6\. SelfShip SDK mapping

SelfShip is a thin wrapper over `langfuse>=4,<5` (Python) and Langfuse JS **v5** (TypeScript). Full
Langfuse API is available via `get_client()` / `getClient()` except media uploads (disabled). Prefer
`workflow()` for identity/tags/metadata enrichment; use `get_client()` for scores and advanced
client ops only — do not invent `release`/`session_id` kwargs on SelfShip `workflow`/`init`.

| SelfShip API | Langfuse concept | Notes |
|---|---|---|
| `init()` | `Langfuse(public_key=org_id, secret_key=repo_secret, host=otel.selfship.ai)` | Credentials from `SELFSHIP_ORG_ID` / `SELFSHIP_REPO_SECRET`; gateway routes via secret→repo_id |
| `SELFSHIP_REPO` | Optional `metadata.repo` enrichment | Repo full name (`owner/repo`); **not** used for project selection |
| `@observe()` | Re-exported `langfuse.observe` | Python: OTel context → automatic nesting |
| `begin()` / `end()` | Root observation + `propagate_attributes` | Flat/single-step interactions |
| `track()` | Single-shot span + immediate end | Fire-and-forget |
| `conversation_id` | `session_id` | Reuse same value across turns in a chat |
| `user_id` | `user_id` | Omit if unknown — do not fabricate |
| `agent_name` | Trace/span name | |
| `set_property` / `set_properties` | Span `metadata` | Per-interaction detail |
| `identify(user_id, traits)` | `metadata.user` on subsequent traces | In-process cache only; not a Langfuse user object |
| `shutdown()` | `flush()` + client shutdown | Required in short-lived processes |

### 6\.1 Python (SDK v4, OTel context)

- Nesting is **automatic** via OpenTelemetry context propagation.
- Prefer `@observe()` or `start_as_current_observation` for multi-step paths.
- Use `begin`/`track` only for flat, single-step interactions.
- Native integrations (`langfuse.openai`, `langfuse.langchain`, `langfuse.llama_index`) auto-create
  child **generations** with token usage.
- Tool/function calls use **`as_type="tool"`** on `@observe` or `start_as_current_observation`
  (LangChain `CallbackHandler` sets tool type automatically).

### 6\.2 TypeScript (Langfuse JS v5 / `@langfuse/tracing`)

- Prefer `workflow({ name, userId, conversationId, tags, metadata }, fn)` — it opens the root
  observation and propagates identity. `getClient()` returns the SelfShip runtime, **not** a
  Langfuse trace factory: there is no `client.trace()`.
- `observe(fn, { name? })` is function-first (returns a wrapper). Never call it options-first
  like `workflow({ name }, fn)`. Nesting uses the active OTel context; for a new request root
  use `workflow()`.
- Build trees with `workflow()` (or `startActiveObservation` from `@langfuse/tracing` after
  `init()`), not the removed classic-SDK `client.trace()` / `trace.span()` APIs.

### 6\.3 Raw OpenTelemetry (no SDK)

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.selfship.ai
OTEL_EXPORTER_OTLP_HEADERS="X-SelfShip-Org-ID=<org_id>,X-SelfShip-Repo-Secret=<repo_secret>"
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

Recognized span attributes:

| Attribute | Purpose |
|---|---|
| `langfuse.user.id` | End-user |
| `langfuse.session.id` | Conversation / session |
| `langfuse.trace.name` | Trace display name |
| `langfuse.environment` | Deployment stage (or `OTEL_RESOURCE_ATTRIBUTES`) |

See [`docs/otel/getting-started.mdx`](../docs/otel/getting-started.mdx).

### 6\.4 Framework integrations (where to look)

Langfuse ships native/OTel integrations for most agent frameworks — each captures tool calls, model
calls, and token usage automatically. Point the underlying exporter at the SelfShip endpoint (§6.3)
and add your `user_id` / `session_id` / `version` via `propagate_attributes` (Python) or
`propagateAttributes` (TS) as shown in each guide's "Interoperability with the SDK" section.

| Framework | How it hooks in | Guide |
|---|---|---|
| **LangChain / LangGraph** | Native `CallbackHandler` via `selfship.langchain_callbacks()` (Python) or nest-inside-`workflow()`/`resumeTrace()` (TS) | [langchain](https://langfuse.com/integrations/frameworks/langchain) |
| **LangChain DeepAgents** | Same as LangChain / LangGraph | [langchain-deepagents](https://langfuse.com/integrations/frameworks/langchain-deepagents) |
| **Vercel AI SDK** | OTel telemetry (AI SDK 7 integration / v6 `experimental_telemetry`) | [vercel-ai-sdk](https://langfuse.com/integrations/frameworks/vercel-ai-sdk) |
| **OpenAI Agents SDK** | OpenInference instrumentation → OTel | [openai-agents](https://langfuse.com/integrations/frameworks/openai-agents) |
| **Claude Agent SDK (Python)** | OpenInference instrumentation → OTel | [claude-agent-sdk](https://langfuse.com/integrations/frameworks/claude-agent-sdk) |
| **Claude Agent SDK (JS/TS)** | OpenInference instrumentation → OTel | [claude-agent-sdk-js](https://langfuse.com/integrations/frameworks/claude-agent-sdk-js) |
| **AutoGen** | OpenLit instrumentation → OTel | [autogen](https://langfuse.com/integrations/frameworks/autogen) |

__SelfShip note:__ OTel-based integrations (OpenInference, OpenLit, Vercel) export wherever
`OTEL_EXPORTER_OTLP_ENDPOINT` points — set it to the SelfShip ingest host with the org headers
(§6.3). For LangChain, prefer `selfship.langchain_callbacks()` after `selfship.init()` over
constructing `CallbackHandler` directly. Invoke the chain **inside** `workflow()`. A chain started
from a copied context with no LangChain parent takes `langchain_callbacks(trace_context=...)`.
A nested invoke inside a running LangGraph node must reuse `config["callbacks"]` (the live
`CallbackManager`) — a fresh handler list, even with `trace_context=`, stamps `is_langchain_root`
and overwrites the trace's I/O. TypeScript has no `langchainCallbacks` helper: nest `chain.invoke`
inside `workflow()` / `resumeTrace()`, and reuse `config.callbacks` inside a running node.
Third-party OTel instrumentation can emit **extra spans** (HTTP, DB) that
still count as billable units — filter them with a `should_export_span` / `shouldExportSpan`
predicate as shown in the AutoGen and Claude Agent SDK (JS) guides.

___

## 7\. Recommended conventions for customer integrations

These are the defaults we want first-PR and docs to steer toward.

### 7\.1 Identity

| Source in customer code | Set as |
|---|---|
| Auth middleware / session `user_id`, `account_id` | `user_id` |
| `conversation_id`, `chat_id`, `thread_id`, `session_id` | `conversation_id` → Langfuse `session_id` |
| Anything else useful (model, tenant, request id) | `metadata` or `tags` |

- Set identity **once at the root** and propagate to children.
- **Never fabricate** identity — omit if unavailable.
- Values must be strings ≤ 200 characters.

### 7\.2 Environment & repo

Add to `.env.example` (one file per repo — values differ per deployment and per codebase):

```bash
SELFSHIP_ORG_ID=org_…
SELFSHIP_REPO_SECRET=bps_…                  # per connected repo; routes ingest via secret→repo_id
# SELFSHIP_REPO=owner/repo                  # optional metadata.repo enrichment (not routing)
LANGFUSE_TRACING_ENVIRONMENT=production   # override to staging/development per deploy (within this repo's project)
# LANGFUSE_RELEASE=                       # git SHA / release tag; set in CI (see §4.6)
```

Without a valid per-repo secret, ingest cannot target a Langfuse project. `SELFSHIP_REPO` /
`GITHUB_REPOSITORY` only enrich metadata.

### 7\.3 Metadata / tags vocabulary (suggested)

| Key / tag | Example | Where |
|---|---|---|
| `repo` | `"acme/widget-api"` | optional via `SELFSHIP_REPO` — enrichment only; project is selected by the ingest secret |
| `model` | `"gpt-4o"` | metadata via `set_property` or `propagate_attributes` |
| `request_id` | `"req_abc123"` | metadata (root) |
| `tenant` / `org` | `"acme-corp"` | metadata if multi-tenant app |
| Feature tag | `"support-bot"` | tags |
| Endpoint tag | `"endpoint:chat"` | tags |

### 7\.4 Trace shape

For each instrumented path:

1. **One root** per unit of work (request / chat turn).
2. **Full path coverage** — model calls, tools, retrieval as nested children; no "dark segments."
3. **Same identity** on root and all descendants.
4. **Output completeness** — root `set_output` / `end(output=...)` carries the full, verbatim
   user-visible deliverable (the same value, post-sanitization / post-fallback, that is
   sent/persisted for the user). Never a slice, preview, length, or status receipt; status fields
   go in metadata. Compute the deliverable once and pass the same variable to the send path and
   `set_output`; scope the root span to cover output finalization. On streamed paths, set the root
   output to the fully drained answer. The metadata/tags ≤ 200-char truncate guidance (§5.1) does
   **not** apply to root output.

___

## 8\. First PR — what we instrument today

The automated first-PR agent (`pr-runner/prompts/AGENTS.md`) is aligned with the practices above:

- Add `selfship-ai` / `@selfship-ai/sdk`; `init()` at the real entry point.
- Set `SELFSHIP_REPO` to the connected repo full name — traces must land in that repo's Langfuse
  project, not a shared org-wide project.
- Instrument **1–3 core LLM paths**, end-to-end with nested observations.
- Wire real `user_id` / `conversation_id` from existing code.
- Prefer native Langfuse integrations for model calls.
- OTel-only path when the repo already exports traces.
- Emit structured JSON including `trace_identity`, `path_coverage`, `integration_points`.

__Not yet enforced by first PR:__

- Prescribed metadata key vocabulary beyond required keys (`workflow_key`, `job_id`, `run_id`) — suggested in §7.3, not hard-coded in prompts.
- Dogfooding — tracing `pr-runner` itself with SelfShip (planned; see `first_pr.md` §5).

__Enforced by Integration (SDK 0.3.3+):__

- **SDK invariants:** null-safe identity normalization, trait-key consistency, metadata stringification/truncation, fail-open behavior.
- **Deterministic blockers:** minimum SDK dependency `>=0.3.9` (both languages; 0.3.9 guarantees a non-empty status message on ERROR spans; 0.3.8 pins the trace I/O mirror to the workflow span), normalization bypasses on direct Langfuse paths, workflow attribution (`workflow:<key>`), scheduled tag + run/job context, metadata literal violations on bypass paths, forbidden direct/private APIs.
- **Semantic reviewer blockers:** real-vs-fabricated identity, threaded-session correctness, high-cardinality tags, trace-per-item fan-out, root/child placement, behavior preservation.
- `SELFSHIP_ENV` in generated `.env.example` (preferred; `LANGFUSE_TRACING_ENVIRONMENT` remains a supported alias resolved by the SDK).

___

## 9\. Operational notes

### 9\.1 Flushing

Langfuse batches traces asynchronously in a background thread. Long-lived processes (web servers,
daemons) flush periodically — nothing to do. **Short-lived processes** lose the last batch if they
exit before the background flush runs. The correct call depends on process lifetime
(`depth_detail.process_lifetime`):

| Lifetime | Call | Why |
|---|---|---|
| `long_lived` / `request` | nothing (periodic flush) | Server keeps running |
| `serverless` (Lambda/Vercel/Cloudflare) | **`flush()` / `forceFlush()` per invocation** — or `waitUntil` / Vercel `after` | Container **freezes** between invocations; `atexit` does **not** fire in Lambda. Do **not** `shutdown()` — it tears down the client so warm invocations stop tracing |
| `batch` / CLI / one-shot | `selfship.shutdown()` (flush + terminate) | Process exits for good |

See [Serverless functions](https://langfuse.com/faq/all/aws-lambda-and-serverless-functions) and
[Instrumentation → client lifecycle](https://langfuse.com/docs/observability/sdk/instrumentation).

### 9\.2 Sampling

To cap trace volume on high-traffic apps, sample **client-side** — the decision is made at the
__trace level__, so if a trace is dropped, all of its observations *and* scores are dropped with it
(no orphaned/partial traces). Rate is a float in `[0, 1]`; default `1` = collect everything, `0.2` =
keep 20%.

- **Python SDK:** `LANGFUSE_SAMPLE_RATE=0.2` env var, or `Langfuse(sample_rate=0.2)` at init.
- **TypeScript / OTel:** Langfuse honors the OpenTelemetry sampler — set a
  `TraceIdRatioBasedSampler(0.2)` on your `NodeSDK` / `registerOTel` alongside
  `LangfuseSpanProcessor` (applies to the OpenAI, LangChain, and Vercel AI SDK integrations too).

__SelfShip guidance:__ keep the default (`1.0`) unless a customer has genuinely high volume — under-
sampling hides the tail (errors, slow turns) that tracing exists to catch. When needed, prefer
env-var config (`LANGFUSE_SAMPLE_RATE`) so it's per-deployment and CI-tunable. Sampling is a volume
knob, **not** a cost/PII redaction tool. See
[Sampling](https://langfuse.com/docs/observability/features/sampling).

### 9\.3 SelfShip-specific limits

- **Media uploads disabled** — gateway blocks `/api/public/media`; SDK sets
  `media_upload_thread_count=0`.
- **Langfuse UI** — internal only (ClusterIP); customers use the SelfShip dashboard.

### 9\.4 Langfuse upgrades

Provisioner writes directly to Langfuse Postgres (`pkg/langfuse/provision.go`). Pin the Helm chart
version and re-validate `organizations` / `projects` / `api_keys` schema on every Langfuse upgrade.

### 9\.5 Coexisting with an existing OpenTelemetry setup

OpenTelemetry uses a **single global `TracerProvider` per process**. Langfuse's OTel-based
Python and JS SDKs (Python v3+/v4, JS v4+/v5) attach a span processor to that global provider
by default.
When a complex app already runs OTel-based tooling (Sentry, Datadog, Logfire, OTel
auto-instrumentation), two things go wrong:

1. **Another tool already owns the global provider.** Sentry/Datadog "claim" it, so initializing
   Langfuse afterward attaches nothing and **no LLM traces are exported** — silently.
2. **We attach to an auto-instrumented provider.** Every processor sees every span, so HTTP,
   database, and framework spans flow to Langfuse as **billable units** and clutter trace trees.

Our SDK floor (`langfuse>=4,<5` → Python v4 / JS v5) has **no default span filter** — only Python
v4+ / JS v5+ auto-drop non-LLM spans. So on our floor this must be handled explicitly.

| Option | When | How |
|---|---|---|
| **Standalone provider mode** (recommended for AI apps) | Another tool owns the global provider, or auto-instrumentation would flood non-LLM spans | Customer code uses `selfship.init(provider_mode="standalone")` / `init({ providerMode: "standalone" })`; do not register or replace a global provider. Use `workflow()` in the LLM execution context. |
| **Shared provider mode + filter** | You want one distributed trace across both tools | Customer code uses `selfship.init(provider_mode="shared", tracer_provider=app_provider, should_export_span=filter)` (TS camelCase equivalents). The app-owned provider must be explicit; keep only LLM/GenAI scopes. |
| **Default global attach** | No existing telemetry (`clean`) | Plain `init()` |

References: [existing OTel setup](https://langfuse.com/faq/all/existing-otel-setup),
[existing Sentry setup](https://langfuse.com/faq/all/existing-sentry-setup),
[unwanted HTTP/DB spans](https://langfuse.com/faq/all/unwanted-http-database-spans).

### 9\.6 Streaming responses

For streaming LLM output (SSE, `StreamingResponse`, async generators, `stream=True`), the
**generation observation must end after the stream is fully consumed** — not when the handler
returns. Ending early drops `output` and token usage, so the generation looks empty/dark. Consume
the stream **inside** the traced scope, or end the observation in the generator's `finally`. Native
wrappers (`langfuse.openai`, `langfuse.langchain`) capture streamed output only when the stream is
drained within the traced scope. Flush **after** the stream completes in serverless (§9.1).

### 9\.7 Avoid double instrumentation & span leaks

- **One integration per call site.** Do not stack `langfuse.openai` + a LangChain `CallbackHandler`
  + an OpenInference instrumentor on the same model call — each emits its own `generation`, so
  tokens and cost are **double-counted**. Pick the single most specific mechanism per call.
- **Close spans on every path.** Wrap observations in context managers / `try…finally` so they end
  on exceptions, and record the error (`level="ERROR"` + a **non-empty** status message) rather than
  leaking an open span. Never mark ERROR without a reason. Soft failures (timeout, empty completion,
  retry) must call `handle.set_error(...)` / `setError(...)` or go through SDK `observe` /
  `langchain_callbacks` (0.3.9+) so the failed attempt keeps a message. Do not delete or downgrade
  the failed span after a retry — Structure stays strict. asyncio cancellation and timeouts
  otherwise leave generations open forever.
- **Async background work.** `asyncio.create_task`, `gather`, and framework background tasks (e.g.
  FastAPI `BackgroundTasks`) capture context at **creation** time; if the root ends before the task
  runs, its spans orphan. Open the root in the same task, or bridge context explicitly — the same
  constraint as threads (§5.2 point 2), one level up.

___

## 10\. Quick decision guide

```text
Need to separate customers?          → Separate SelfShip orgs (we provision separate Langfuse orgs)
Need to separate repos in one org?   → One Langfuse project per repo; use that repo's SELFSHIP_REPO_SECRET (§1, §2.2)
Need prod vs staging for one repo?   → LANGFUSE_TRACING_ENVIRONMENT (same repo project)
Need different RBAC per stage?       → Separate Langfuse projects per stage (Langfuse-native; not our default)
Need to filter traces by user?       → user_id (first-class) (§4.4)
Need to group multi-turn chats?      → conversation_id / session_id (§4.2)
Need to mark a tool/retriever call?  → observation type via as_type / asType (§4.1)
Need token counts / USD cost?        → ingest usage_details/cost_details on a generation (§4.3)
Need to capture 👍/👎 or ratings?     → send a score linked to the trace id (§4.5)
Need to A/B test / attribute a regression? → release (build) + version (component) (§4.6)
Need to cut trace volume at high traffic? → LANGFUSE_SAMPLE_RATE / OTel sampler (§9.2)
App already runs OTel/Sentry/Datadog?  → SelfShip standalone or explicitly shared provider_mode (§9.5)
Serverless (Lambda/Vercel/CF)?         → flush()/forceFlush() per invocation, NOT shutdown() (§9.1)
Streaming LLM output?                  → end the generation after the stream drains (§9.6)
Multiple integrations on one call?     → pick one — stacking double-counts cost (§9.7)
Need to filter by feature/endpoint?  → tags
Need arbitrary context?              → metadata on root trace (+ propagate_attributes for trees)
```
