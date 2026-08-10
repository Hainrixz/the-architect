# Shape: Agent App

> An application where the model *is* the product — chat, copilot, or autonomous worker — built by teams whose core loop is prompt → tool → trace → eval, not CRUD.

Last verified: 2026-07-27

## Is this your project?

**Yes if:**
- The value is "it answers", "it drafts", "it does the task for you" — remove the model and nothing is left.
- They talk about conversations, threads, or "the agent going off and doing things for a few minutes".
- They want it to call tools, hit APIs, or edit their data on their behalf, and pricing is per message, per run, or credits.

**No if:** — more briefs point *into* this shape than out of it, so be strict about the exits.

- The model is one feature inside a subscription product (a summarize button on a dashboard) → `knowledge/shapes/saas-webapp.md`.
- The output is an image, video, or audio asset and the hard parts are jobs, queues, and asset delivery → `knowledge/shapes/generative-media-app.md`.
- It lives entirely inside Slack/Discord/email with no product surface and no eval loop → `knowledge/shapes/automation-bot-integration.md`.
- It ships as an MCP server, SDK, or terminal tool for developers → `knowledge/shapes/cli-library-mcp.md`.
- The chat surface is incidental and the hard requirements are a phone's — push, offline, camera, background audio → `knowledge/shapes/mobile-app.md`.
- The model needs the user's filesystem, local processes, a tray item, or a global hotkey to be useful at all → `knowledge/shapes/desktop-app.md`.
- The deliverable is endpoints someone else's client calls, and the model call is one handler behind them → `knowledge/shapes/api-backend.md`.
- Nothing is built yet and the ask is really a waitlist page for an AI idea → `knowledge/shapes/marketing-site.md`.

**Then pick the execution mode.** Both share one data model and one gateway; they differ only in *who waits*. Ship synchronous first, add durable runs later — build the shared spine so that is not a rewrite.

| | **Synchronous** | **Durable** |
|---|---|---|
| Examples | chat, RAG Q&A, inline copilot | research agent, migration agent, ops runbook |
| Unit of work | a message, answered in seconds | a run, completed in minutes to hours |
| User waits? | yes, watching tokens stream; retry on failure | no, returns to a result; resume from last step |
| Needs | streaming, history, context strategy | queue, checkpoints, approvals, idempotency |
| Variant | realtime voice — `knowledge/capabilities/realtime-voice.md` | — |

## Default runtime track

**TypeScript / Node** — see `knowledge/runtime-tracks/ts-node.md`. Streaming to a browser, one language across gateway and UI, and first-class SDK support make it the fastest path from prompt to shipped product.

Alternatives: `knowledge/runtime-tracks/python.md` when the team is Python-native or retrieval and offline eval pipelines dominate; `knowledge/runtime-tracks/go.md` when the agent is mostly a high-throughput multi-tenant proxy.

## Core capabilities

| Capability | Why this shape needs it | File |
|---|---|---|
| LLM integration | Provider abstraction, streaming, caching, structured output | `knowledge/capabilities/ai-llm-integration.md` |
| Agent loop | Tool registry, step execution, retries, approvals | `knowledge/capabilities/agent-loop.md` |
| Observability | Traces per call are the debugger; without them you guess | `knowledge/capabilities/observability.md` |
| Database | Runs, steps, messages, traces are all durable state | `knowledge/capabilities/database.md` |
| Auth | Tools act on user data — every call needs an identity | `knowledge/capabilities/auth.md` |
| Credit metering | Marginal cost per request is real and must be capped | `knowledge/capabilities/credit-metering.md` |
| Testing | Prompts are code; evals are their test suite | `knowledge/capabilities/testing.md` |
| Deployment | Long responses and background workers break naive hosting | `knowledge/capabilities/deployment.md` |
| Payments, enterprise readiness *(later)* | Selling credits; buyers asking about retention and model training | `knowledge/capabilities/payments-rails.md`, `knowledge/capabilities/enterprise-readiness.md` |
| Mainland China operation | Foreign model endpoints are not a production path inside the border, and a public AI feature carries a filing and a disclosure surface | `knowledge/capabilities/china-mainland.md` |

## Data model

| Entity | Holds | Belongs to |
|---|---|---|
| `conversations` | thread metadata, title, owner | user / org |
| `messages` | role, content, attachments, status | conversation |
| `runs` | one unit of agent work: input, mode, status, result | conversation, or standalone |
| `steps` | ordered checkpoints: state, attempt, output | run |
| `tool_calls` | tool name, validated args, result, error, idempotency key | step |
| `traces` | latency, tokens in/out/cached, cost, model id | run + step |
| `approvals` | gated call, requester, decision, decided_at | tool_call |
| `documents` / `chunks` | source text, embeddings, provenance — optional, see Pitfalls | org |
| `eval_cases` | input, expected behavior, tags, baseline score | standalone golden set |
| `usage_events` | billable units per user per period, derived from traces | user |

`runs.status` drives everything: `queued → running → awaiting_approval → succeeded | failed | cancelled`. Never infer status from the presence of rows.

## Directory structure

```
src/
  gateway/      # ONLY place a provider SDK is imported: model ids from config,
                #   streaming, cancellation, partial persistence, typed errors
  agent/
    tools/      # one file per tool: schema, handler, timeout, approval flag
    loop.*      # step executor: plan -> call -> persist -> repeat
    runner.*    # durable-mode worker entrypoint
  retrieval/    # optional -- only if the eval justified it
  evals/        # golden cases (version-controlled) + scorer vs committed baseline
  api/  db/  ui/ # streaming + run routes; migrations; chat view, run timeline, approvals
```

## Build order

**Shared spine**

1. **Model gateway** — one module wrapping the provider: config-driven model ids, timeouts, typed errors, streaming. If a provider SDK is imported in two places, swapping models becomes a refactor. · *Done when:* a test asserts a simulated provider rate-limit surfaces as a typed retryable error, and a mocked stream yields at least two incremental deltas.
2. **Data spine** — migrations for conversations, messages, runs, steps, tool_calls, traces, usage_events. · *Done when:* migrations apply to a clean database and roll back cleanly; a seed inserts one run with three steps and a query returns them in order.
3. **Tracing before features** — every gateway call writes a trace with model id, latency, input/output/cached tokens, cost. · *Done when:* after a single request, `traces` has a row with non-null token counts joined to its run.

**Synchronous mode**

4. **Streaming endpoint** — accept a message, stream the response, persist the turn on close. · *Done when:* an event-stream request emits incremental chunks, and killing the client mid-stream still persists a message with status `aborted` plus its partial content.
5. **Conversation UI** — thread list, streaming render, stop button, error states. · *Done when:* WHEN the user presses stop THE SYSTEM SHALL halt rendering within one second and persist the partial turn; a reload restores full history.
6. **Context strategy** — decide what goes into the prompt (full history, summarized tail, or retrieval) by measuring, not assuming. · *Done when:* the eval suite reports accuracy and cost per strategy over at least twenty questions, and the winner is committed with its numbers.

**Durable mode**

7. **Tool layer** — typed registry: schema, handler, per-tool timeout, approval flag, idempotency key. · *Done when:* a tool invoked with invalid arguments returns a structured validation error back to the model instead of throwing, covered by a test.
8. **Run orchestrator** — persist each step before executing it; a worker picks up queued runs. · *Done when:* killing the worker mid-run and restarting resumes from the last committed step, with a test asserting exactly one side-effect row.
9. **Human-in-the-loop approvals** — gated tools pause the run and notify. · *Done when:* WHEN a run reaches an approval-gated tool THE SYSTEM SHALL set status `awaiting_approval`, notify, and resume on approve or terminate on reject — surviving a process restart.
10. **Run timeline UI** — steps, tool calls, errors, cost, plus an approval inbox. · *Done when:* an E2E test loads a seeded run by id and asserts the rendered timeline contains exactly one row per persisted step, each carrying its arguments, result, error, and token cost, and that no request captured during that page load queries the database directly.

**Hardening**

11. **Eval harness** — golden set with a committed baseline, wired into CI. · *Done when:* the eval command scores every case, writes a report, and exits non-zero when the pass rate falls below the baseline.
12. **Metering, limits, guardrails** — usage_events derived from provider-reported tokens, quota enforced before the call, all tool and retrieval output treated as untrusted. · *Done when:* summed usage_events match provider-reported totals within one percent, an over-quota user is rejected with zero provider spend, and a regression test proves an instruction embedded in a retrieved document does not trigger a gated tool.
13. **Deploy** — streaming path plus background worker, both under load. · *Done when:* a streamed response arrives incrementally through the production edge with no buffering, and a durable run completes end to end in staging.

## Pitfalls

- **Reaching for chunk-and-embed RAG by default.** It is now the wrong first move for most corpora. Long context plus prompt caching is cheaper and far more accurate when the relevant set fits, and it deletes chunking, embedding drift, and a vector store from your ops surface. Decide with the eval in step 6, not with an opinion.

  | Situation | Use |
  |---|---|
  | Corpus fits in context, changes slowly, reused across requests | Long context + caching |
  | Millions of documents, unpredictable slice per query, or answers must cite exact sources | Filter first; then hybrid keyword + vector with a reranker |

- **Building durable runs on the HTTP request lifetime.** Serverless timeouts and load-balancer idle limits will kill a ten-minute run. Durable work belongs in a queue-backed worker with persisted steps from day one.
- **Tool calls that are not idempotent.** Retries are guaranteed. A tool that sends an email or charges a card needs an idempotency key stored on the row *before* it executes.
- **Estimating cost by counting characters.** Persist provider-reported usage per call, including cached tokens; estimates drift far enough to make margin analysis fiction.
- **Shipping without evals.** With no golden set nobody can safely change the prompt, so it calcifies and every regression is found by a customer instead.
- **Trusting retrieved or tool-returned text.** Anything the model reads from outside the request is attacker-controlled. Gate destructive tools behind approvals; never let fetched content escalate permissions.
- **Unbounded conversation history.** Cost and latency grow every turn until requests fail. Enforce a truncation or summarization policy in the gateway.

## Skills for the build phase

From `knowledge/skills-registry.md`. All optional — if one is absent, fall back to the knowledge base or built-in `WebSearch` / `WebFetch`, note it in one line, and keep going. Use `find-skills` to discover skills for the specific tools this agent will call.

| Skill | Use for |
|---|---|
| `claude-api` | Auto-activates. Invoke before writing any model id, price, or API parameter — never hand-maintain a model table. |
| `/last30days` | Current opinion on agent frameworks, eval tooling, and retrieval before committing. |
| `ui-ux-pro-max` | Visual system for the conversation view and run timeline. |
| `emil-design-eng` | Streaming, typing, and stop-button motion — what makes an agent feel responsive. |
| `frontend-design` | Production UI, build phase only. |
| `playwright-cli` | E2E coverage of streaming, cancellation, and the approval flow. |

## See also

- `knowledge/capabilities/ai-llm-integration.md` — provider abstraction, streaming, caching, structured output
- `knowledge/capabilities/agent-loop.md` — step executor, tool contracts, retries, approvals, observability hooks
- `knowledge/runtime-tracks/ts-node.md` — the default track, with pinned versions
- `knowledge/shapes/saas-webapp.md` — when the model is a feature, not the product
- `knowledge/shapes/mobile-app.md` — when the phone's capabilities, not the model, are the hard part
- `knowledge/shapes/desktop-app.md` — when the agent needs the user's machine to do its job
