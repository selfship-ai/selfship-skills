# SelfShip ↔ Langfuse Integration Parity

> Companion to [`langfuse_integration_model.md`](./langfuse_integration_model.md). That doc catalogs how
> all 42 framework integrations on [langfuse.com/integrations](https://langfuse.com/integrations) actually
> work under the hood. This doc maps each one onto **what SelfShip can support today**, given our actual
> architecture (`internal-docs/archi.md`) and SDKs (`sdk/python`, `sdk/typescript`).

---

## 0. Why parity is (mostly) automatic

Per `archi.md` §7.1, SelfShip is deliberately a **byte-identical, thin wrapper**:

```text
public_key = org_id
secret_key = repo_secret
host       = https://otel.selfship.ai
```

The gateway (`otel.selfship.ai`) accepts the exact same wire formats a self-hosted Langfuse instance
does — `Authorization: Basic base64(pk:sk)` on `/api/public/ingestion` and
`/api/public/otel/v1/traces` (+ `/v1/traces` alias) — and swaps to the real Langfuse project keys
server-side. **Any client that only needs a configurable `host`/`baseUrl` and two credential strings
works against SelfShip with zero code changes beyond the endpoint.** That is true whether the client
is our own SDK, the raw `langfuse` package, or a generic OpenTelemetry exporter.

This means SelfShip's integration surface is **not limited to what our two SDKs (`selfship-ai` /
`@selfship-ai/sdk`) implement** — it's the union of:

1. Everything reachable through our SDKs directly (`get_client()` / `getClient()` passthrough to the
   full Langfuse client).
2. Everything reachable by pointing the **raw Langfuse SDK** at our host with SelfShip credentials
   (documented today in `docs/python/integrations.mdx`).
3. Everything reachable via **raw OTLP/HTTP** against the gateway with no SDK at all
   (`docs/otel/getting-started.mdx`), which is how the large majority of the 42 Langfuse framework
   integrations actually work anyway (see classification key below).

For customer code using a SelfShip SDK, configure coexistence only through the public API:
`provider_mode="standalone"` / `providerMode: "standalone"` by default, or explicit shared mode with
the application-owned provider and `should_export_span` / `shouldExportSpan`. Use `workflow()` for
per-workflow tracing; do not construct raw Langfuse clients or span processors in customer code.

---

## 1. Confidence legend

| Parity                  | Meaning                                                                                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| ✅ **Full — verified**    | Mechanism matches something already documented/tested in our own docs (`docs/**`) or is a direct, unmodified pass-through of the pinned Langfuse SDK. |
| ✅ **Full — high confidence** | Generic OTLP/HTTP export with configurable endpoint + headers/basic-auth. Same mechanism our own `otel/*` docs already demonstrate for other frameworks; no framework-specific blocker expected. |
| ⚠️ **Likely — unverified** | Depends on a third-party/custom package accepting a fully custom `host`/`baseUrl` and non-`pk-lf-`/`sk-lf-`-shaped keys. Plausible but **not tested against SelfShip**. |
| ❌ **Gap**                | A known platform-level limitation (see §4) blocks or degrades the integration.                                                                    |
| n/a **No SDK**           | No Java/Kotlin/Rust/C++ SelfShip SDK exists. Raw-OTLP still applies if the framework's own OTel plumbing allows a custom endpoint — SelfShip's gateway is language-agnostic. |

---

## 2. Full comparison — all 42 frameworks

| # | Framework | Langfuse mechanism | SelfShip parity | Notes |
|---|---|---|---|---|
| 1 | LangChain & LangGraph (Python) | `selfship.langchain_callbacks()` (wraps `langfuse.langchain.CallbackHandler`) | ✅ Full — verified | Prefer the SDK helper over constructing `CallbackHandler` directly (`docs/python/integrations.mdx`, `docs/otel/langchain.mdx`). Invoke inside `workflow()`. Cross-boundary chains with no LangChain parent take `langchain_callbacks(trace_context=wf.trace_context())`. Nested invokes inside a running node must reuse `config["callbacks"]` (the live `CallbackManager`) — a fresh handler list, even with `trace_context=`, stamps `is_langchain_root` and overwrites the trace's I/O. |
| 1b | LangChain & LangGraph (JS) | Nest `chain.invoke` inside `workflow()` / `resumeTrace()`; reuse `config.callbacks` inside a running node | ⚠️ Likely — unverified | There is no `langchainCallbacks` helper in `@selfship-ai/sdk`. The TS SDK pins `@langfuse/tracing` / `@langfuse/otel` **v5**, not the monolithic `langfuse` v3 package. Do not construct `CallbackHandler` with `{ traceId, parentSpanId }` from `handle.traceContext()` — SelfShip spells the parent `observationId`, Langfuse expects `parentSpanId`. See §4.2. |
| 2 | Vercel AI SDK | AI SDK telemetry → `@langfuse/otel`/`@langfuse/vercel-ai-sdk` | ✅ Full — verified | No Langfuse package needed at all — generic `@vercel/otel` + `OTEL_EXPORTER_OTLP_*` env vars. Documented in `docs/otel/vercel-ai.mdx`. |
| 3 | Google ADK | OpenInference `GoogleADKInstrumentor` | ✅ Full — high confidence | Standard OTLP/HTTP export; point at gateway. |
| 4 | Pydantic AI | `Agent.instrument_all()` (native OTel) | ✅ Full — high confidence | Pure OTel env-var redirection. |
| 5 | OpenAI Agents SDK | OpenInference `OpenAIAgentsInstrumentor` | ✅ Full — high confidence | Same generic OTLP path. |
| 6 | Agno | OpenLit instrumentation | ✅ Full — high confidence | Same generic OTLP path. |
| 7 | AI SDK C++ | Built-in `ai::langfuse` CMake target/Tracer | ⚠️ Likely — unverified | n/a SDK (C++). Depends on whether the tracer exposes a configurable host and accepts arbitrary key strings. |
| 8 | Amazon Bedrock AgentCore | ADOT disabled, OTLP env vars | ✅ Full — high confidence | Pure env-var OTel; language of the workload doesn't matter. |
| 9 | AutoGen | OpenLit instrumentation | ✅ Full — high confidence | Same generic OTLP path. |
| 10 | BeeAI | OpenInference `BeeAIInstrumentor` | ✅ Full — high confidence | Same generic OTLP path. |
| 11 | Claude Agent SDK (Python) | OpenInference `ClaudeAgentSDKInstrumentor` | ✅ Full — high confidence | Same generic OTLP path. |
| 12 | Claude Agent SDK (JS/TS) | OpenInference JS + `@langfuse/otel` | ✅ Full — high confidence | Skip `@langfuse/otel`; use a generic Node OTel SDK `OTLPTraceExporter` pointed at the gateway. |
| 13 | CrewAI | OpenInference `CrewAIInstrumentor` | ✅ Full — high confidence | Same generic OTLP path. |
| 14 | DSPy | OpenInference `DSPyInstrumentor` | ✅ Full — high confidence | Same generic OTLP path. |
| 15 | Embabel | Native OTel starter + 3rd-party Maven exporter | ⚠️ Likely — unverified | n/a SDK (Java). The Maven exporter (`opentelemetry-exporter-langfuse`, not Langfuse-official) needs to allow a custom endpoint; unverified. |
| 16 | Eve | Native OTel tracer + `@langfuse/otel` | ✅ Full — high confidence | Skip `@langfuse/otel`; generic OTLP/HTTP exporter works. |
| 17 | Flue | `@flue/opentelemetry` + `@langfuse/otel` | ✅ Full — high confidence | Same — generic OTLP adapter works without the Langfuse-specific package. |
| 18 | Haystack | OpenInference `HaystackInstrumentor` | ✅ Full — high confidence | Same generic OTLP path. |
| 19 | Instructor | `instructor.patch()` wraps `langfuse.openai.OpenAI` | ✅ Full — verified | Same mechanism as our documented OpenAI integration (`docs/python/integrations.mdx`); Instructor just patches the same client. |
| 20 | Koog | Native OTel feature + `addLangfuseExporter()` | ⚠️ Likely — unverified | n/a SDK (Kotlin). `addLangfuseExporter()` may assume Langfuse Cloud host conventions; needs verification. |
| 21 | LangChain DeepAgents | `selfship.langchain_callbacks()` | ✅ Full — verified | Identical to #1, including nested `CallbackManager` reuse. |
| 22 | Langserve | `langchain_callbacks()` via `RunnableConfig` | ✅ Full — verified | Identical to #1. Reuse the parent `config["callbacks"]` rather than building a fresh list per request. |
| 23 | LiteLLM SDK | `litellm.callbacks = ["langfuse_otel"]` | ✅ Full — high confidence | LiteLLM's built-in callback reads `LANGFUSE_HOST` / `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` directly — set these to the gateway host + `org_id` + `repo_secret`. Bypasses our SDK entirely (LiteLLM manages its own client), but works. |
| 24 | LiveKit | Native OTel + `tracer_provider` passed into `Langfuse()` | ✅ Full — verified | Python path uses the same `Langfuse()` constructor our SDK wraps; wire the SelfShip client's underlying provider directly. |
| 25 | LlamaIndex | OpenInference `LlamaIndexInstrumentor` (current) / `langfuse.llama_index.LlamaIndexCallbackHandler` (legacy, still in pinned v3 package) | ✅ Full — verified | Both paths documented already (`docs/python/integrations.mdx` shows the legacy callback handler); OpenInference route also works via generic OTLP. |
| 26 | LlamaIndex Workflows | Same OpenInference `LlamaIndexInstrumentor` | ✅ Full — high confidence | Same generic OTLP path. |
| 27 | Mastra | `@mastra/langfuse` → `LangfuseExporter` | ⚠️ Likely — unverified | `LangfuseExporter` typically accepts `publicKey`/`secretKey`/`baseUrl`. Pointing `baseUrl` at the gateway with `org_id`/`repo_secret` should work (same wire format) but is untested against SelfShip. |
| 28 | Microsoft Agent Framework | `configure_otel_providers()` (native OTel) | ✅ Full — high confidence, ⚠️ caveat | Tracing itself is pure OTel env-var redirection. The framework's `langfuse` package usage is limited to `auth_check()` — verify that call hits one of our gateway's allow-listed read routes, or that it's optional. |
| 29 | Mirascope | `mirascope.integrations.langfuse.with_langfuse()` → calls Langfuse `@observe()` internally | ✅ Full — verified | Ultimately routes through the same `langfuse` package our Python SDK wraps; works once `selfship.init()` has run. |
| 30 | Pipecat | `enable_tracing=True` + OTLP exporter (native OTel) | ✅ Full — high confidence | Pure OTel env-var redirection. |
| 31 | Quarkus LangChain4j | Native OTel (Quarkus extension) | ✅ Full — high confidence | n/a SDK (Java), but OTel is language-agnostic — gateway doesn't care. |
| 32 | Ragas | `create_score()` / scores API | ✅ Full — verified | Writes go through `/api/public/ingestion`; reads needed for scoring are covered by the gateway's allow-listed routes (`/api/public/traces*`, `/observations*`, `/sessions*`, `/scores*` — see `archi.md` §4.3). |
| 33 | Restate | OpenInference wrapped by `RestateTracer` | ✅ Full — high confidence | Generic OTLP wrapper; assumed configurable like all OpenInference-based integrations. |
| 34 | Semantic Kernel | OpenLit instrumentation | ✅ Full — high confidence | Same generic OTLP path. |
| 35 | SmolAgents | OpenInference `SmolagentsInstrumentor` | ✅ Full — high confidence | Same generic OTLP path. |
| 36 | Spring AI | Native OTel (Actuator/Micrometer bridge) | ✅ Full — high confidence | n/a SDK (Java); OTel is language-agnostic. |
| 37 | Strands Agents | `strands-agents[otel]` extra (native OTel) | ✅ Full — high confidence, ⚠️ caveat | Same `auth_check()` caveat as #28. |
| 38 | Swiftide | Rust `langfuse` Cargo feature, `LangfuseLayer` | ⚠️ Likely — unverified | n/a SDK (Rust). Depends on whether `LangfuseLayer` accepts a custom base URL. |
| 39 | TanStack AI | OpenInference middleware + `@langfuse/otel` | ✅ Full — high confidence | Skip `@langfuse/otel`; generic OTLP/HTTP exporter works. |
| 40 | Temporal | OpenInference + Temporal `OpenAIAgentsPlugin` | ✅ Full — high confidence | Assumes the plugin's OTel wiring is itself generic (standard for Temporal plugins). |
| 41 | VoltAgent | `@voltagent/langfuse-exporter` | ⚠️ Likely — unverified | Same caveat as Mastra (#27) — needs a configurable `baseUrl`. |
| 42 | Watsonx Orchestrate ADK | `orchestrate settings observability langfuse configure` (CLI) | ⚠️ Likely — unverified | CLI/config-only, no SDK. Unclear whether the CLI accepts a fully custom host + arbitrary key format, or assumes Langfuse Cloud conventions. |

---

## 3. Summary counts

| Parity | Count | Frameworks |
|---|---|---|
| ✅ Full — verified | 10 | LangChain & LangGraph (Py), Vercel AI SDK, Instructor, LangChain DeepAgents, Langserve, LiveKit, LlamaIndex, Ragas, Mirascope, (Python OpenAI via `langfuse.openai`, implicit) |
| ✅ Full — high confidence (generic OTLP) | 24 | Google ADK, Pydantic AI, OpenAI Agents SDK, Agno, Amazon Bedrock AgentCore, AutoGen, BeeAI, Claude Agent SDK (Py+JS), CrewAI, DSPy, Eve, Flue, Haystack, LiteLLM, LlamaIndex Workflows, Microsoft Agent Framework, Pipecat, Quarkus LangChain4j, Restate, Semantic Kernel, SmolAgents, Spring AI, Strands Agents, TanStack AI, Temporal |
| ⚠️ Likely — unverified (custom package, host-override untested) | 7 | AI SDK C++, Embabel, Koog, Mastra, Swiftide, VoltAgent, Watsonx Orchestrate ADK |
| ⚠️ Partial (JS v4 package gap) | 1 | LangChain & LangGraph (JS) |

**Net: ~34/42 (81%) already have full or high-confidence parity today, entirely because the gateway
speaks Langfuse's own wire protocol.** The remaining 8 are custom third-party packages/CLIs we haven't
tested, not architectural blockers — closing them is a verification exercise, not new engineering (see
§5).

---

## 4. Known platform-level gaps

These apply regardless of which of the 42 frameworks a customer uses:

### 4.1 Media / multimodal uploads — disabled

Per `archi.md` §4.3 & §9.2, `/api/public/media` and `/api/public/media/*` are **blocked (404)** at the
gateway, and the SDK sets `media_upload_thread_count=0`. Any integration that relies on Langfuse's
presigned-S3 media-upload flow for images/audio/file attachments will silently drop that data.
Text/JSON-only trace payloads are unaffected.

### 4.2 JS/TS SDK uses Langfuse OTel packages (v5+)

Our `@selfship-ai/sdk` depends on `@langfuse/tracing`, `@langfuse/otel`, and OpenTelemetry.
Customer integrations must use public SelfShip APIs (`init` / `workflow` / `flush`) — not raw
`LangfuseSpanProcessor` construction. Frameworks that are OTel-native can still export via the
gateway OTLP path without any Langfuse package (see §2). Optional framework packages such as
`@langfuse/langchain` remain untested when pointed directly at SelfShip host/keys.

### 4.3 OTLP protocol support — HTTP/protobuf only

The gateway's documented and tested path is `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf` against
`https://otel.selfship.ai` (`docs/otel/getting-started.mdx`, `docs/api-reference/otel.mdx`). There is
no gRPC listener (port 4317-style) fronting the gateway. Frameworks/exporters that default to gRPC
OTLP must be explicitly configured for HTTP/protobuf, or they will fail to connect.

### 4.4 No Java / Kotlin / Rust / C++ SelfShip SDK

We only ship Python and TypeScript SDKs. For ecosystems without a SelfShip SDK (Java: Embabel, Spring
AI, Quarkus LangChain4j; Kotlin: Koog; Rust: Swiftide; C++: AI SDK C++), the raw-OTLP gateway path is
the only route — and only works if that framework's own tracer/exporter exposes a configurable
endpoint and doesn't hardcode Langfuse Cloud/key-format assumptions.

### 4.5 EE-only Langfuse features are moot (not gaps)

Langfuse's Instance Management API (org/project/key provisioning) is EE-licensed and irrelevant here —
SelfShip provisions tenants itself via direct SQL (`archi.md` §5). This isn't a customer-facing
integration gap, just a reason our provisioning differs from a vanilla Langfuse Cloud signup.

### 4.6 `auth_check()` calls in framework OTel bootstrapping

A few native-OTel frameworks (Microsoft Agent Framework, Strands Agents) call `langfuse`'s
`auth_check()` purely to validate credentials before starting OTel export — not to send traces through
it. Whether that specific call resolves against our allow-listed read routes needs a one-time check per
framework; tracing itself is unaffected either way.

---

## 5. Closing the remaining gaps — recommended next steps

1. **Verify the 7 "likely — unverified" custom packages** (Mastra, VoltAgent, AI SDK C++, Embabel, Koog,
   Swiftide, Watsonx Orchestrate ADK) against a real SelfShip org + repo secret. For each, confirm: (a)
   host/baseUrl is overridable, (b) it doesn't reject non-`pk-lf-`/`sk-lf-`-prefixed keys, (c) it doesn't
   assume gRPC OTLP.
2. **Decide on JS v4 parity**: either officially document "install `@langfuse/otel` /
   `@langfuse/langchain` directly against SelfShip host + keys" as a supported power-user path (Layer 1
   passthrough, same spirit as our Python `get_client()`), or explicitly scope it out.
3. **Add framework guide pages** for the ✅ high-confidence OTLP frameworks we don't yet have docs for
   (currently only Vercel AI SDK and LangChain have dedicated `docs/otel/*.mdx` pages) — mechanically
   the same env-var swap, so this is low-effort content work, not engineering.
4. **Re-evaluate media uploads** if a customer integration specifically depends on it (e.g. multimodal
   agent frameworks attaching images to traces) — currently a deliberate platform decision, not a
   technical limitation of the gateway design.
5. **Confirm gRPC is truly unsupported** (or add a gRPC listener) if adoption data shows frameworks
   defaulting to gRPC OTLP in practice (common default for some Java/Go OTel SDKs).
