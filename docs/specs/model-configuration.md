# Model Router and model configuration specification

Status: MVP specification. Owner slice: module specifications. Related: [Main Agent](main-agent.md), [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md), [Knowledge Service and Learner](knowledge-and-learning.md), [Discovery](discovery.md), [Persistence](persistence.md), [Deployment and operations](../architecture/deployment-and-operations.md).

## 1. Purpose

Organisations that run their own AI infrastructure expect to point a product at their own model endpoints, switch models when better ones become available, and keep data inside their environment. The Model Router makes that a configuration matter rather than a code change:

- three **model slots**, each independently configured: **main** (the Main Agent), **worker** (Analysis Workers), and **background** (batched learning, discovery inference, and small classification steps);
- **OpenAI-compatible endpoints** as the common interface, plus **native clients** for providers where a native interface is useful;
- **no embedding model** in MVP, because knowledge is loaded by index and description and files are narrowed through the Aras graph and searched with code.

Every model call in the product goes through the Model Router. No other component holds model credentials.

## 2. Responsibilities and non-responsibilities

### Responsibilities

1. Hold the configuration of each slot: endpoint, client type, model name, generation parameters, limits, and fallbacks.
2. Translate the product's canonical message format (system instructions, user and assistant messages, tool definitions, tool calls, tool results) to and from each client type.
3. Stream responses where the caller asks for streaming (the Main Agent, for the live view).
4. Retry transient failures and fail over to configured fallback endpoints (section 5.4).
5. Check at startup and on configuration reload that each slot's endpoint satisfies the capability requirements (section 5.2).
6. Enforce per-slot concurrency limits and the per-turn token budget.
7. Record a usage row for every call ([model_calls](persistence.md#model_calls)).
8. Keep model credentials out of the database, the logs, and the Sandbox.

### Non-responsibilities

- It does not run an agent loop; the Main Agent and Analysis Worker run their own thin loops (see [main-agent.md](main-agent.md) and [analysis-worker-and-sandbox.md](analysis-worker-and-sandbox.md)). No agent framework is used.
- It does not decide prompts, tools, or context compaction; callers do. It reports token counts so callers can manage context.
- It does not store prompts or completions; transcripts are stored by the Main Agent ([messages](persistence.md#messages)).
- It does not provide embeddings, fine-tuning (deferred), or hosting of models.
- It does not cache responses (caching is not in MVP).

## 3. Interfaces

The Model Router is internal; it is not an agent tool.

| Operation | Caller | Inputs | Outputs | Errors |
|---|---|---|---|---|
| Complete | Main Agent (slot main), Analysis Worker loop (slot worker), Learner and Discovery (slot background) | slot; canonical messages; tool definitions (optional); options: streaming, maximum output tokens, structured output requested; attribution: conversation id, turn id, analysis job id or background job id | assistant message (text and tool calls), finish reason, token usage (input, output), endpoint and model used, model call id | see error categories below |
| Count tokens | Main Agent, Analysis Worker loop, Learner | slot; canonical messages | estimated input tokens for that slot's model | none (falls back to an estimate) |
| Slot status | operations, Chat App (degraded banner) | none | per slot: healthy, degraded (on fallback), unavailable; last check time; capability check results | none |
| Reload configuration | operations | none | new configuration applied to new calls | invalid configuration rejected, previous kept |

Error categories returned to callers (internal to the router, not agent tool error codes):

| Category | Meaning | Retried by the router | Caller's expected handling |
|---|---|---|---|
| transient | Network error, timeout, server error from the endpoint | Yes, then fallback | If still failing, end the turn with a clear message; the conversation is saved and can be retried |
| rate_limited | Endpoint returned a rate limit | Yes, honouring any retry hint, then fallback | As transient |
| context_overflow | Input exceeds the model's context | No | Compact context (Main Agent) or split work, then call again |
| invalid_request | Malformed request or unsupported parameter | No | Defect; logged and the turn fails |
| auth_failed | Endpoint rejected the credential | No (fallback attempted) | Operations alert; turn fails |
| refused | The model declined or returned a filtered response | No | Caller shows a neutral message and records it |
| budget_exhausted | The per-turn token budget is used up | No | Main Agent ends the turn with partial results and coverage |

## 4. Data owned

- [model_calls](persistence.md#model_calls): one row per call attempt. Fields: id (`mdl_`); slot; endpoint name; client type; model name; conversation id, turn id, analysis job id, background job id (as applicable); attempt number and whether it was a fallback; streaming flag; input tokens; output tokens; latency in milliseconds; finish reason; status (succeeded or error category); started_at. No prompt or completion content.

Slot configuration is not stored in the database; it lives in the deployment configuration (section 6). Credentials are referenced by name and resolved from the deployment's secret source (Proposed: environment variables or a secret file mounted at runtime).

## 5. Behaviour

### 5.1 Slots and what uses them

| Slot | Used by | Character of the work | Typical requirement |
|---|---|---|---|
| main | Main Agent | Conversation, tool use, AML writing, answer synthesis with citations | Strong reasoning and tool use; streaming; large context |
| worker | Analysis Workers | Multi-step analysis: fetching data to files, writing and running Python, reporting with citations | Strong tool use and code writing; large context; streaming not required |
| background | Learner (batched learning, assumption evaluation), Discovery (model-inferred entries, re-derivation) | Non-interactive, cost-sensitive, latency-tolerant | Good instruction following and structured output; may be a smaller model |

Slots may point at the same endpoint and model. The product assumes a reasonably competent model in the main and worker slots; quality of answers depends on it.

### 5.2 Capability requirements and checks

Each slot's model must support (Proposed requirements):

| Capability | main | worker | background |
|---|---|---|---|
| Tool (function) calling with structured arguments | required | required | not required |
| Several tool calls in one response | recommended (parallel workers) | recommended | not required |
| Structured output (valid JSON object on request) | required | required | required |
| Streaming | required | optional | optional |
| Context window | at least 128,000 tokens (Proposed) | at least 128,000 tokens (Proposed) | at least 32,000 tokens (Proposed) |

At startup and on reload, the router runs a capability check per slot: a short request that requires a tool call with a structured argument and, for main, a streamed response. A failing check marks the slot unavailable (or degraded if a fallback passes), is logged, and is shown on the operations status. Configured context window and output limits come from configuration because endpoints do not reliably report them.

### 5.3 Canonical format and client types

1. Callers always use the canonical format. The router maps it to the client type of the endpoint: an OpenAI-compatible chat interface, or a native client for a provider where the native interface offers something material (for example better tool calling or reasoning controls).
2. Tool call identifiers are preserved across the mapping so that tool results match their calls on any endpoint.
3. Because transcripts are stored in the canonical format, a conversation can continue on a different model (after a configuration change or failover) without conversion. Provider-specific content that cannot be carried over (for example opaque reasoning content) is dropped when the model changes (Proposed).
4. Which native clients ship in MVP is an implementation choice; the OpenAI-compatible client is mandatory.

### 5.4 Retry and failover

1. On a transient or rate-limited error, the router retries the same endpoint with exponential backoff and jitter (Proposed: 3 attempts, starting at 1 second, honouring a retry hint from the endpoint up to 30 seconds).
2. If attempts are exhausted, or on auth_failed, the router tries the slot's fallback endpoints in order with the same policy.
3. A streamed response that fails after tokens were already delivered is not silently replayed on another endpoint; the router returns transient and the Main Agent restarts that model step (the live view shows that the step is being retried).
4. A slot running on a fallback is reported as degraded.
5. Every attempt, including failed ones, writes a model_calls row.

### 5.5 Budgets and concurrency

1. Each slot has a concurrency limit; calls beyond it wait in a queue (Proposed: with a timeout of 60 seconds, after which transient is returned).
2. A per-turn token budget covers the Main Agent and every Analysis Worker of the turn (Proposed). When it is exhausted the router returns budget_exhausted and the Main Agent finishes the answer with what it has, reporting reduced coverage.
3. Background calls never take concurrency from the main slot when the slots share an endpoint: limits are per slot.

### 5.6 Changing models

Changing a slot's model or endpoint is a configuration change followed by reload. New calls use the new configuration; calls in flight finish on the old one. No code change and no data migration is needed, since no embeddings or model-specific stores exist.

## 6. Configuration

Per slot (keys shown for `main`; `worker` and `background` have the same keys):

| Key | Type | Default | Meaning |
|---|---|---|---|
| models.main.client | enumeration: openai_compatible, or a native client name | openai_compatible | Client type for the primary endpoint. |
| models.main.base_url | URL | none (required) | Endpoint address, typically inside the customer's network. |
| models.main.model | text | none (required) | Model name as the endpoint expects it. |
| models.main.credential_ref | text | none | Name of the secret holding the API key; never the key itself. |
| models.main.context_window_tokens | integer | none (required) | Context size of the model, used for budgeting and compaction. |
| models.main.max_output_tokens | integer | 8,000 (Proposed default) | Output limit per call. |
| models.main.temperature | number | endpoint default | Sampling temperature, if the model accepts it. |
| models.main.extra_parameters | map | empty | Provider-specific parameters passed through (for example reasoning effort). |
| models.main.timeout_seconds | integer | 300 (Proposed default) | Per-call timeout. |
| models.main.max_concurrent_calls | integer | 16 for main and worker, 2 for background (Proposed defaults) | Concurrency limit of the slot. |
| models.main.retry_attempts | integer | 3 (Proposed default) | Attempts per endpoint on transient errors. |
| models.main.fallbacks | ordered list of endpoint definitions (client, base_url, model, credential_ref, context_window_tokens) | empty | Fallback endpoints. |
| models.turn_token_budget | integer | 3,000,000 (Proposed default) | Total tokens per user turn across main and worker calls. |
| models.capability_check_on_start | boolean | true | Run capability checks at startup and reload. |

## 7. Limits

- Per-slot concurrency and the per-turn token budget as configured.
- Output length per call as configured; when a response ends because of the limit, the finish reason says so and the caller decides whether to continue.
- No limit on the number of endpoints, but each slot has one primary endpoint at a time (no load balancing across endpoints in MVP, Proposed).

## 8. Failure modes and recovery

| Failure | Detection | Effect on user | Recovery |
|---|---|---|---|
| Primary endpoint down | Transient errors after retries | Slower answer while failing over; none if a fallback works | Automatic failover; slot shown degraded |
| All endpoints of the main slot down | Failover exhausted | Turn ends with "the language model is unavailable"; question saved for retry | Operations restore the endpoint; user retries |
| Worker slot down | Failover exhausted | Main Agent answers what it can directly and reports that analysis could not run | As above |
| Background slot down | Failover exhausted | None visible; learning and inference delayed; uncertain entries remain verified before use | Background Jobs retry later |
| Model lacks a required capability | Capability check | Slot unavailable from startup | Operations choose another model |
| Context overflow | Endpoint error or pre-check | None visible if compaction succeeds | Caller compacts and retries |
| Credential revoked | auth_failed | As endpoint down | Operations update the secret and reload |
| Token budget exhausted | Router | Answer delivered with partial results and stated coverage | User may ask a narrower question |

## 9. Security considerations

- **Data stays where the customer sends it.** The product sends prompts only to the endpoints configured by the customer. No telemetry or prompt content goes anywhere else.
- **Credentials.** Model credentials are read from the secret source at startup, held in memory by the router, never written to the database, logs, transcripts, or the Sandbox. The Analysis Worker loop runs in the service, so the Sandbox never needs model access.
- **What models see.** Models see Aras data the signed-in user could see, as returned by the Tool Gateway. Customers choosing an external endpoint accept that this data leaves their network; the deployment guide should state this plainly (see [deployment and operations](../architecture/deployment-and-operations.md)).
- **Logs.** Router logs carry identifiers, token counts, latency, and error categories, never message content.
- **Background prompts.** Discovery inference receives configuration only; batched learning receives transcripts, which are private conversation records already stored on the customer's server.

## 10. Observability

- model_calls rows provide per-slot, per-model, per-conversation, and per-job usage, which also supports the business metrics in [value and ROI](../business/value-and-roi.md).
- Metrics (Proposed): calls, tokens, latency percentiles, and errors by slot, endpoint, and category; failover count; slot health; queue wait times; budget exhaustion count.
- Audit events `config.reloaded` (who, when, which slots changed) and `model.slot_health_changed` (slot, old and new status) (see [Audit and progress](audit-and-progress.md#51-audit-event-catalogue)).

## 11. MVP versus deferred

In MVP: three slots, OpenAI-compatible client plus native clients where useful, retry and failover, capability checks, per-slot concurrency, per-turn token budget, usage records.

Not in MVP (see [roadmap](../product/roadmap.md)):

- embedding models (none needed; any future vector index for files or knowledge would add an embedding slot);
- model fine-tuning on customer data (not supported);
- response caching (deferred with query result caching);
- usage and ROI dashboard on top of model_calls (deferred).

## 12. Open questions

1. Proposed: capability requirements and minimum context sizes in section 5.2.
2. Proposed: retry policy (3 attempts, exponential backoff, retry hint honoured up to 30 seconds) and fallback order.
3. Proposed: a per-turn token budget of 3,000,000 tokens across main and worker calls; should there also be per-user daily budgets?
4. Proposed: per-slot concurrency defaults (16, 16, 2) and a 60-second queue timeout.
5. Proposed: provider-specific content that cannot be carried across models is dropped when a conversation moves to another model.
6. Proposed: one primary endpoint per slot, no load balancing in MVP.
7. Which native clients ship in MVP besides the mandatory OpenAI-compatible client?
8. Proposed: credentials resolved from environment variables or a mounted secret file.
