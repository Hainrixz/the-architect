# Capability: AI / LLM Integration

> Everything between your application and a language model provider: one gateway, streaming, structured output, caching, cost control, guardrails, and the evals that let you change a prompt without fear.

Last verified: 2026-07-27

## Standing rule — never hand-maintain a model table

**Do not write a model id, a price, a context window, a rate limit, or an API parameter name into this repo or into a blueprint from memory.** Model lineups and pricing move faster than any file that describes them. This file will be stale; a self-updating skill will not.

Before writing any of those, invoke the bundled **`claude-api`** skill (auto-activates — no slash) and take the values from it. If you are targeting a non-Anthropic provider, fetch the provider's current model and pricing page with `WebFetch` and cite the retrieval date in the blueprint. If neither is available, write `VERIFY: model id and price` in the blueprint rather than guessing. A wrong model id fails loudly; a wrong price silently destroys your margin model.

The same rule applies to the built project: model ids live in **config or environment**, never inlined at a call site.

## When a project needs this

- Any feature described as "it summarizes / drafts / classifies / answers questions / extracts".
- The product streams text to a user and they watch it appear.
- The brief mentions embeddings, semantic search, auto-tagging, or first-pass moderation.
- Cost per request is variable and shows up in the P&L — see `knowledge/capabilities/credit-metering.md`.
- If the model also *acts* — calls tools in a loop, runs for minutes — you need `knowledge/capabilities/agent-loop.md` on top of this.
- The users are inside mainland China → `knowledge/capabilities/china-mainland.md`. Foreign model endpoints are not a production path from inside the border, a public generative feature carries a filing and a required disclosure surface, and input **and** output moderation stop being optional. The single-gateway rule below is what makes the provider swap a config change instead of a rewrite.

## Decision matrix

| Option | Best for | Pros | Cons |
|---|---|---|---|
| **Direct provider SDK** behind your own gateway module | One primary provider, most products | Newest features on day one, best streaming and caching support, no extra hop, no extra vendor | You write the abstraction; a second provider is your work |
| **Multi-provider SDK / router** (Vercel AI SDK, LiteLLM, OpenRouter, Portkey) | Shipping fast across providers, or reselling model choice | One interface, easy fallback, per-key spend caps | Lags provider features, lowest-common-denominator params, another failure domain and bill |
| **Cloud-hosted managed inference** (Bedrock, Vertex AI, Azure AI Foundry) | Enterprise procurement, data residency, existing cloud committed spend | Runs inside your compliance boundary, billing via the cloud contract | Model availability trails the provider, different auth and quota model, regional gaps |
| **Agent/RAG framework** (LangChain, LlamaIndex, and friends) | Prototypes and pipelines with many connectors | Fast to demo, batteries included | Heavy abstraction over an API that is already simple; debugging happens in someone else's control flow |
| **Self-hosted open weights** (vLLM, Ollama, TGI on your own GPUs) | Strict data isolation, extreme volume, offline or on-device | No per-token bill, full control, no vendor terms | You now operate a GPU fleet; quality gap on hard reasoning; capacity planning is your problem |

## Recommendation

**Use the provider's own SDK, imported in exactly one module — your gateway.** Everything else in the app calls the gateway, never the SDK. That module owns model selection, timeouts, retries, streaming, cancellation, token accounting, and typed errors.

This gets you provider features the day they ship, and it gets you provider portability anyway — because the seam you actually need is *your* interface, not a third party's. Swapping providers means rewriting one file, which is the same work a router would have done for you, minus a vendor.

Deviate when: procurement requires the model to run inside your cloud account (managed inference); you genuinely sell model choice to end users (router); or data may not leave your network at all (self-hosted).

**Do not adopt a framework for a chat call.** Reach for one only when you need its retrieval connectors and you have measured that they beat forty lines of your own code.

## Model selection by task tier

Pick a tier by the job, then get the current id for that tier from the `claude-api` skill. Route per call site — one model for the whole product is either overpaying or underperforming.

| Tier | Use for | Rule of thumb |
|---|---|---|
| **Small / fast** | Classification, routing, extraction, tagging, moderation pre-filter, short rewrites | Anything with a fixed output shape and a right answer. Latency and unit cost dominate; run these at high volume |
| **Mid / workhorse** | Production chat, summarization, drafting, most tool-calling, RAG answers | Your default. Ship on this tier, and only escalate a call site when an eval shows it failing |
| **Frontier** | Multi-step planning, hard reasoning, code generation, long-horizon agent runs, judging other models | Expensive per call but often cheaper per *solved task* — one correct answer beats four cheap wrong ones plus a retry |

Two rules that matter more than the choice: **start one tier up, then move down with an eval to prove it holds** — down-tiering blind is how quality regressions ship. And **batch/offline work belongs on whatever asynchronous discount the provider offers**, since nobody is waiting.

## Streaming

Stream anything a human reads. Non-streamed responses feel broken above roughly two seconds.

- Server-sent events from your server to the browser; the provider stream is consumed server-side so keys never reach the client.
- **Persist partial output.** If the client disconnects mid-stream, save what was generated with a distinct status. Users reload and expect their half-answer.
- **Propagate cancellation** all the way to the provider request. An abandoned stream you keep consuming is pure burnt cost.
- **Disable proxy buffering and compression** on the streaming route, and send a periodic keepalive — CDNs and load balancers will otherwise buffer the whole response or drop an idle connection.
- Errors can arrive *after* a successful status and several chunks. Handle mid-stream failure as a first-class case in both server and UI.

## Structured output

- Use the provider's schema-constrained mode (tool/function calling or native JSON schema). Never ask for JSON in prose and parse the result.
- Keep **one source of truth** for each schema and derive both the provider schema and the runtime validator from it. Two hand-written copies drift within a week.
- Validate every response even in constrained mode. On validation failure, retry once with the validator error appended — then fail loudly.
- Keep schemas shallow and required fields few. Deep nesting and long enums measurably degrade quality; ask for a flat object and assemble it in code.

## Tool use

Definitions and loop mechanics live in `knowledge/capabilities/agent-loop.md`. At the integration layer, only three things matter: the tool schema and its validator come from one definition; a tool error is **returned to the model as a result**, never thrown through the loop; and every tool argument set is validated before execution, because tool args are model output and model output is untrusted input.

## Prompt caching

The single highest-leverage cost and latency optimization. Caching is **prefix-based**: the cached span must be byte-identical from the very start of the request.

- Order the prompt **most stable first**: system instructions, tool definitions, long reference documents, then conversation, then the current user message.
- One changed byte anywhere before a cache breakpoint invalidates everything after it. Timestamps, request ids, and shuffled tool order in the system prompt are the classic cache killers.
- Cache writes cost more than ordinary input; reads cost dramatically less. Caching wins when a prefix is reused across requests or turns, and loses on genuinely one-shot calls.
- Verify current pricing multipliers, minimum cacheable length, and TTL with the `claude-api` skill before designing around them.
- Instrument cache hit rate as a first-class metric. A silent drop to zero after a prompt refactor is a large invisible bill.

## Cost control

- **Never estimate cost by counting characters.** Persist provider-reported usage — input, output, cache-write, cache-read — per call. Estimates drift far enough to make margin analysis fiction.
- Cap max output tokens on every call. Runaway generation is a real bill.
- Enforce a per-user and per-org budget **before** the request, not after. See `knowledge/capabilities/credit-metering.md`.
- Bound conversation history with a truncation or summarization policy in the gateway. Cost and latency grow every turn until requests fail.
- Alert on cost per request, not just total spend — a prompt change that doubles unit cost is invisible in a monthly total until the bill arrives.

## Rate limits and retries

- Retry only on `429`, provider overload, and transport failures. Never retry a `400` — the request is wrong and will stay wrong.
- Exponential backoff **with jitter**, honoring the provider's retry-after header when present. Synchronized retries across your fleet are how a blip becomes an outage.
- **Do not silently retry a stream that already emitted tokens** into a fresh stream; the user sees the answer restart. Surface it.
- Cap concurrency at your own gateway and queue the overflow, so one tenant cannot consume the organization's whole rate limit.
- Track rate-limit headroom as a metric. Discovering your ceiling during a launch is the wrong time.

## Guardrails, safety, and PII

- **Every byte the model reads from outside the request is attacker-controlled**: retrieved documents, tool results, user uploads, scraped pages, email bodies. Instructions come from the system prompt only. Content never escalates permissions.
- Delimit untrusted content explicitly in the prompt and tell the model it is data, not instruction. This reduces prompt injection; it does not eliminate it. The real control is that destructive tools require approval — see `knowledge/capabilities/agent-loop.md`.
- Never place secrets, API keys, or full credential material in a prompt. They end up in logs, traces, and vendor retention.
- **PII**: decide what may leave your boundary before you build. Check the provider's zero-retention and no-training terms, and get them in the contract if a buyer will ask. Redact identifiers before sending when the task does not need them.
- **Your traces are a PII sink.** Prompt logs contain everything the user typed. They inherit the same retention, access control, region, and deletion obligations as your primary database — including deletion requests. See `knowledge/capabilities/enterprise-readiness.md`.
- Moderate inputs and outputs where user-generated content is public-facing; log the flag with the message id rather than blocking silently.

## Evals

Prompts are code. Without a test suite they calcify, and every regression is discovered by a customer.

- **Golden set in version control**: real inputs, expected behavior, tags. Twenty well-chosen cases beat a thousand synthetic ones. Grow it from production failures — every bug becomes a case.
- **Scorers, cheapest first**: exact match and structural checks where possible; a model-as-judge with an explicit rubric only where they are not. Calibrate any judge against human labels once, or you are measuring the judge.
- **Commit a baseline** and gate CI on it: the eval command exits non-zero when the pass rate drops. This is the whole point.
- Track **cost and latency alongside accuracy** in every eval run. A change that is two points better and three times more expensive is usually a bad change.
- Sample production traffic for online scoring. Offline evals never cover what users actually send.

## Data model additions

| Entity | Holds |
|---|---|
| `llm_calls` | model id, purpose, latency, input/output/cache-read/cache-write tokens, cost, finish reason, error |
| `prompts` | named prompt, version, template body, active flag — so a trace can name the exact prompt that produced it |
| `usage_events` | billable units per user per period, derived from `llm_calls`, never re-estimated |
| `eval_cases` | input, expected behavior, tags, baseline score |
| `eval_runs` | commit sha, prompt version, model id, pass rate, cost, latency |
| `moderation_flags` | message id, category, score, action taken |

Join `llm_calls` to whatever your shape's unit of work is — a message, a run, a job. Every model call must be attributable to a user and a feature or you cannot do unit economics.

## Build steps this adds

1. **Gateway module** — one file wraps the provider SDK: model ids from config, per-call timeout, typed errors, streaming, cancellation. · *Done when:* a test asserts a simulated rate limit surfaces as a typed retryable error and a simulated `400` as non-retryable; a grep proves the SDK is imported in exactly one file.
2. **Usage persistence** — every call writes an `llm_calls` row from provider-reported usage. · *Done when:* after one request the row exists with non-null input, output, and cache token counts, attributable to a user.
3. **Streaming path** — SSE endpoint with keepalive, partial persistence, and abort propagation. · *Done when:* a client that disconnects mid-stream leaves a persisted partial with status `aborted`, and the provider request is observably cancelled.
4. **Structured output** — one schema definition generating both provider schema and validator, with one repair retry. · *Done when:* a forced malformed response is repaired on retry, and a second failure raises a typed error covered by a test.
5. **Prompt caching** — reorder the prompt stable-first, set breakpoints, expose hit rate. · *Done when:* the second identical request reports non-zero cache-read tokens and measurably lower latency, both asserted in a test.
6. **Budgets and limits** — pre-call quota check, max output tokens, gateway concurrency cap, backoff with jitter. · *Done when:* WHEN a user exceeds their quota THE SYSTEM SHALL reject the request before any provider call, with zero spend recorded.
7. **Guardrails** — untrusted content delimited, secrets excluded from prompts, trace redaction. · *Done when:* a regression test proves an instruction embedded in a retrieved document does not change tool behavior, and no test fixture secret appears in a persisted trace.
8. **Eval harness** — golden set, scorers, committed baseline, CI gate. · *Done when:* the eval command scores every case, writes a report with pass rate, cost, and p95 latency, and exits non-zero below baseline.

## Pitfalls

- **Hardcoding a model id at a call site.** Config only. Otherwise the next model release is a refactor across the codebase.
- **Importing the provider SDK in more than one place.** The gateway stops being a seam the moment there is a second import.
- **Chunk-and-embed RAG as the reflex first move.** When the relevant corpus fits in context, long context plus prompt caching is usually cheaper *and* more accurate, and it removes chunking, embedding drift, and a vector store from your ops surface. Choose with an eval. Retrieval wins when the corpus is large, the needed slice is unpredictable per query, or answers must cite exact sources — and then it wants filtering plus hybrid keyword-and-vector search with a reranker, not naive top-k.
- **Temperature as a quality knob.** For extraction and classification, lower it and fix the prompt. Sampling settings do not repair an ambiguous instruction.
- **Retrying on validation failure forever.** One repair attempt, then fail. Retry loops turn a bad prompt into a bad bill.
- **Trusting token counts you computed yourself.** Tokenizers differ by model and change. Use provider-reported usage.
- **Shipping the demo prompt.** The prompt that impressed in a notebook has no golden set, no version, and no owner. Version it before it reaches production.
- **Letting traces become an unmanaged PII lake.** Retention and access control from day one, not after the first security questionnaire.

## See also

- `knowledge/capabilities/agent-loop.md` — when the model calls tools in a loop and runs must survive a restart
- `knowledge/capabilities/credit-metering.md` — turning `llm_calls` into quotas, credits, and a bill
- `knowledge/capabilities/observability.md` — traces, spans, and the dashboards that make model behavior debuggable
- `knowledge/capabilities/testing.md` — where the eval harness sits relative to unit and E2E tests
- `knowledge/shapes/agent-app.md` — the shape where the model *is* the product
- `knowledge/runtime-tracks/ts-node.md`, `knowledge/runtime-tracks/python.md` — where provider SDKs get pinned
