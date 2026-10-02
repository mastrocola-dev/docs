# ADR-007: Agent runtime on Functions Flex Consumption with queue-based jobs

**Status:** Proposed
**Date:** 2026-10-01

## Context

The agent runs only in a pipeline (ADR-005). The public site adds the third execution moment: a visitor asks about the architecture and the `ask` instance answers. That exposes a paid API to anonymous traffic, crosses five repositories, and needs a runtime that costs nothing when idle. The agent architecture already fixes that HTTP access is asynchronous (job and polling), never request/response.

Options considered for compute:

- **Container Apps**, as ADR-003 and ADR-004 expected. Rejected as default: an image, a registry and a build pipeline per service for workloads that are a single function each. It remains the fallback if the Functions host cannot load `.ts` without a build.
- **Functions Flex Consumption** — scale to zero, managed identity for every connection, access restrictions, code deployed as a zip of sources.

## Decision

```
browser ─► Cloudflare (rate limit) ─► api ─► queue jobs ─► worker ─► mcp-docs
                                       ▲                      │  └──► Anthropic API
                                       └──── queue events ◄───┘
```

- **Three function apps on Flex Consumption:** `api` (new repository `service-api`), `worker` (a `function.ts` adapter over `run()` in `service-agent`) and `mcp-docs` (a streamable HTTP adapter over `createServer`, redeployed on every push to `docs` main; the stdio adapter stays for CI). The `always ready` option is rejected for cost.
- **Jobs travel through Service Bus Basic.** `api` sends to `jobs`; `worker` is triggered by it and reports through `events`, which triggers `api`. Peek-lock, dead-letter queue, `maxDeliveryCount: 2`: a redelivered job is failed, never run again, so a crash cannot be billed twice.
- **The worker stores nothing and `api` owns all state.** `api` holds no agent code. The contract below is the only coupling: each side validates it with its own zod schema as a tolerant reader, and no package is shared (ADR-004).
- **State in Table Storage** behind `JobStore` and `QuotaStore` ports: jobs (kept 24 h), per-IP quotas and daily cost, partitioned by day and purged by a timer. Cosmos DB replaces it only for vector search, queries beyond point reads or fragile cleanup; its free tier is reserved for RAG.
- **LLM through the direct Anthropic API**, workspace `runtime`, whose spend limit is the hard ceiling.
- **Abuse controls, outermost first:** Cloudflare rate limit; origin reachable only from Cloudflare addresses (list as code) over a Cloudflare origin certificate in Full (strict) mode; CORS restricted to the site; Turnstile verified on the server; questions of 3–500 characters; 10 questions per IP per day (hashed `CF-Connecting-IP`); bounded concurrency (429); US$ 1 per day computed with `cost()`, checked before a job is accepted (503) and added on completion with ETag concurrency.
- **Prompt injection is contained, not prevented.** `ask` runs Haiku 4.5 with `maxSteps: 6`, `maxRunTokens: 30000`, `runTimeoutMs: 60000` and the three read-only tools. The question is delimited in `<question>`; the output is `{ outOfScope, answer ≤ 1500, sources ≤ 5 }`. `outOfScope` shows a fixed message; otherwise `sources` is required and checked against existing documents. The site renders with `textContent` and builds the links itself.
- **Identity (ADR-006):** one user-assigned identity per function app, created with its roles in bootstrap. `worker` calls `mcp-docs` with an Entra token; `api` reads `turnstile-secret-key` and `worker` reads `anthropic-api-key-runtime` from Key Vault.
- **Observability:** an OpenTelemetry exporter replaces `fileTracer` at runtime, metadata only. Application Insights alerts (daily cost at 50/80/100%, dead-lettered message, failure rate) reach e-mail through an Action Group; an Azure budget covers infrastructure. No mail provider in code.
- **Infrastructure** in `infra/agent/`; `api.mastrocola.dev` with DNS as code. `docs/adr-index.yml` pins `service-agent` and `mcp-docs` by tag.

### Message contract (v1)

Every message carries `v` and `jobId`. Readers ignore unknown fields and unknown `type` values; a breaking change increments `v`.

| Queue | Message |
|---|---|
| `jobs` | `{ v, jobId, instance: 'ask', input: { question } }` |
| `events` | `{ v, jobId, seq, type: 'started' }` |
| `events` | `{ v, jobId, seq, type: 'step', kind: 'model' \| 'tool', name? }` |
| `events` | `{ v, jobId, seq, type: 'completed', output, costUsd }` |
| `events` | `{ v, jobId, seq, type: 'failed', reason: 'timeout' \| 'budget' \| 'redelivered' \| 'error', costUsd }` |

Basic queues have no ordering guarantee: `api` applies an event only when its `seq` is greater than the stored one, and a terminal state is final. Job states are `queued → running → done | failed`.

## Consequences

Positive:

- Idle cost is close to zero; the dominant cost is the LLM, capped twice (daily budget and workspace limit)
- No secret is added outside Key Vault and no connection string exists: queues, tables and the vault are reached by identity
- `service-agent` gains a transport without changing `run()`; storage or queue can be swapped behind ports
- A failed job is visible: dead-letter alert and a `failed` state, never a silent retry

Negative, accepted:

- Cold starts on both hops; the interface shows `queued` and `running` instead of hiding the wait
- Six moving parts for one question box; accepted as the reference shape for later agents
- Polling instead of streaming; quotas per IP are unfair behind shared addresses
- A crashed run is not retried: the visitor asks again
- This ADR supersedes ADR-003's planned Foundry re-evaluation and ADR-004's "each server runs as its own Container App"; their decisions otherwise stand

## Pending before acceptance

Confirmed by documentation: Flex Consumption supports Node 24, inbound access restrictions, identity-based connections for host storage, deployment and Service Bus, and site-scoped certificates for custom domains. To be confirmed by a disposable deployment:

1. The Functions host loads a `.ts` entry point with no build (Node 24)
2. A Cloudflare origin certificate binds to the custom domain as a site-scoped certificate imported from Key Vault
3. Entra authentication between function apps with a managed identity as caller
4. Cold start of the HTTP and queue paths, which decides whether the site warms the runtime on focus
