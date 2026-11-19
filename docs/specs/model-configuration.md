# Model Router and model configuration specification

Status: MVP specification. Related: [Main Agent](main-agent.md), [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md), [Knowledge Service and Learner](knowledge-and-learning.md), [Discovery](discovery.md), [Persistence](persistence.md), [Deployment and operations](../architecture/deployment-and-operations.md).

## 1. Purpose

Organisations that run their own AI infrastructure expect to point a product at their own model endpoints, switch models when better ones become available, and keep data inside their environment. The Model Router makes that a configuration matter rather than a code change:

- three **model slots**, each independently configured: **main** (the Main Agent), **worker** (Analysis Workers), and **background** (the Learner, discovery inference, and by default the final answer check);
- **OpenAI-compatible endpoints** as the common interface. The MVP ships the OpenAI-compatible client only; a **native client** for a provider is added when a customer's provider needs one;
- **no embedding model** in MVP, because knowledge is loaded by index and description and files are narrowed through the Aras graph and searched with code.

Every model call in the product goes through the Model Router. No other component holds model credentials.

## 2. Responsibilities and non-responsibilities

### Responsibilities

1. Hold the configuration of each slot: endpoint, client type, model name, generation parameters, limits, and fallbacks.
2. Translate the product's canonical message format (system instructions, user and assistant messages, tool definitions, tool calls, tool results) to and from each client type.
3. Stream responses where the caller asks for streaming (the Main Agent, for the live view).
4. Retry transient failures and fail over to configured fallback endpoints (section 5.4).
5. Check at startup and on configuration reload that each slot's endpoint satisfies the capability requirements (section 5.2).
6. Enforce per-slot concurrency limits and the per-turn token budget, and serve live calls ahead of batch calls within a slot (section 5.5).
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
| Complete | Main Agent (slot main; its final answer check on the slot configured for it, background by default), Analysis Worker loop (slot worker), Learner and Discovery (slot background) | slot; canonical messages; tool definitions (optional); options: streaming, maximum output tokens, structured output requested, priority (`live` or `batch`, default `live`); attribution: conversation id, turn id, analysis job id or background job id | assistant message (text and tool calls), finish reason, token usage (input, output), endpoint and model used, model call id | see error categories below |
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

`model_calls` is the only record of model usage. A model call is not an audit event, and the row holds no prompt or completion content.

Slot configuration is not stored in the database; it lives in the deployment configuration (section 6). Credentials are referenced by name and resolved at runtime from environment variables or a secret file mounted into the service; an integration with a customer's secret manager is added on customer request, which the reference keys allow without a change to the configuration format ([ADR 0034](../decisions/0034-credentials-encrypted-at-rest.md)).

## 5. Behaviour

### 5.1 Slots and what uses them

| Slot | Used by | Character of the work | Typical requirement |
|---|---|---|---|
| main | Main Agent | Conversation, tool use, AML writing, answer synthesis with citations | Strong reasoning and tool use; streaming; large context |
| worker | Analysis Workers | Multi-step analysis: fetching data to files, writing and running Python, reporting with citations | Strong tool use and code writing; large context; streaming not required |
| background | Learner (every step: live merges and splits inside `record_learning`, assumption evaluation, which judges the user's next message for confirmation or objection, and batched learning), Discovery (model-inferred entries and their re-derivation), the Main Agent's model-based final answer check, which runs on every answer unless switched off (default slot; the Main Agent configuration can move it to the main slot) | Cost-sensitive; mostly non-interactive and latency-tolerant, plus short live steps | Good instruction following and structured output; may be a smaller model |

Slots may point at the same endpoint and model. The product assumes a reasonably competent model in the main and worker slots; quality of answers depends on it.

Discovery is the single caller of the model for discovered entries: it runs inference and re-derivation of model-inferred entries, and the Knowledge Service applies the results. Live steps on the background slot (the final answer check, assumption evaluation, and the Learner's live calls) are served ahead of batch steps (batched learning, Discovery inference), so they never wait behind batch work (section 5.5; see also [ADR 0047](../decisions/0047-final-answer-check.md)).

### 5.2 Capability requirements and checks

Each slot's model must support the following ([ADR 0051](../decisions/0051-model-slot-requirements-and-budgets.md)):

| Capability | main | worker | background |
|---|---|---|---|
| Tool (function) calling with structured arguments | required | required | not required |
| Several tool calls in one response | recommended (parallel workers) | recommended | not required |
| Structured output (valid JSON object on request) | required | required | required |
| Streaming | required | optional | optional |
| Context window | at least 128,000 tokens | at least 128,000 tokens | at least 32,000 tokens |

At startup and on reload, the router runs a capability check per slot that tests exactly what that slot is required to do: for main and worker, a short request that requires a tool call with a structured argument, and a request for a valid JSON object; for main, also a streamed response; for background, a request for a valid JSON object. The router also compares the slot's configured context window with the minimum above. A failing check, or a configured window below the minimum, marks the slot unavailable (or degraded if a fallback passes), is logged, and is shown on the operations status. Configured context window and output limits come from configuration because endpoints do not reliably report them.

The Learner sizes each batched-learning chunk from the background slot's configured context window: at most half of it, and never above `learner.batch_max_transcript_tokens` (see [Knowledge and learning](knowledge-and-learning.md)). A model that meets the minimum can therefore process every chunk.

### 5.3 Canonical format and client types

1. Callers always use the canonical format. The router maps it to the client type of the endpoint: the OpenAI-compatible chat interface, or a native client where one has been added for a provider whose native interface offers something material (for example better tool calling or reasoning controls).
2. Tool call identifiers are preserved across the mapping so that tool results match their calls on any endpoint.
3. Because transcripts are stored in the canonical format, a conversation can continue on a different model (after a configuration change or failover) without conversion. Provider-specific content that cannot be carried over (for example opaque reasoning content) is dropped when the model changes; the visible conversation and the tool results are what later steps need.
4. The MVP ships the OpenAI-compatible client only. A native client is added when a customer's provider needs one; the canonical format keeps that a bounded task.

### 5.4 Retry and failover

1. On a transient or rate-limited error, the router retries the same endpoint with exponential backoff and jitter (by default 3 attempts, starting at 1 second, honouring a retry hint from the endpoint up to 30 seconds; see `models.main.retry_attempts`, `retry_initial_delay_seconds` and `retry_max_hint_seconds` in section 6).
2. If attempts are exhausted, or on auth_failed, the router tries the slot's fallback endpoints in order with the same policy.
3. A streamed response that fails after tokens were already delivered is not silently replayed on another endpoint; the router returns transient and the Main Agent restarts that model step (the live view shows that the step is being retried).
4. A slot running on a fallback is reported as degraded.
5. Every attempt, including failed ones, writes a model_calls row.

### 5.5 Budgets and concurrency

1. Each slot has a concurrency limit; calls beyond it wait in a queue for at most `models.main.queue_timeout_seconds` (default 60 seconds), after which transient is returned.
2. A per-turn token budget covers the Main Agent and every Analysis Worker of the turn. Tokens are input plus output tokens, as recorded in `model_calls`. Calls on the background slot (the Learner, Discovery, and by default the answer check) are not counted against it. When the budget is exhausted the router returns budget_exhausted and the Main Agent finishes the answer with what it has, reporting reduced coverage.
3. Background calls never take concurrency from the main slot when the slots share an endpoint: limits are per slot.
4. Within a slot, calls waiting in the queue are served by priority: `live` calls (the final answer check, assumption evaluation, the Learner's live calls) before `batch` calls (batched learning, Discovery inference). A batch call already running is not interrupted.
5. Discovery caps its model-inferred entry generation per run (200 inference calls by default; the remainder is queued for the next run and those entries are absent until generated; see [Discovery](discovery.md)). The cap bounds the batch workload on the background slot.

### 5.6 Changing models

Changing a slot's model or endpoint is a configuration change followed by reload. New calls use the new configuration; calls in flight finish on the old one. No code change and no data migration is needed, since no embeddings or model-specific stores exist.

## 6. Configuration

Per slot (keys shown for `main`; `worker` and `background` have the same keys):

| Key | Type | Default | Meaning |
|---|---|---|---|
| models.main.client | enumeration: `openai_compatible` (the only client in the MVP; the name of each native client is added with that client) | openai_compatible | Client type for the primary endpoint. |
| models.main.base_url | URL | none (required) | Endpoint address, typically inside the customer's network. |
| models.main.model | text | none (required) | Model name as the endpoint expects it. |
| models.main.credential_ref | text | none | Name of the secret holding the API key; never the key itself. |
| models.main.context_window_tokens | integer | none (required) | Context size of the model, used for budgeting and compaction and checked against the slot's minimum (section 5.2). |
| models.main.max_output_tokens | integer | 8,000 | Output limit per call. |
| models.main.temperature | number | endpoint default | Sampling temperature, if the model accepts it. |
| models.main.extra_parameters | map | empty | Provider-specific parameters passed through (for example reasoning effort). |
| models.main.timeout_seconds | integer | 300 | Per-call timeout. |
| models.main.max_concurrent_calls | integer | 16 for main and worker, 2 for background | Concurrency limit of the slot. |
| models.main.queue_timeout_seconds | integer | 60 | Longest wait in the slot queue before transient is returned. |
| models.main.retry_attempts | integer | 3 | Attempts per endpoint on transient errors. |
| models.main.retry_initial_delay_seconds | integer | 1 | Delay before the first retry; later retries back off exponentially with jitter. |
| models.main.retry_max_hint_seconds | integer | 30 | Longest retry hint from the endpoint that is honoured. |
| models.main.fallbacks | ordered list of endpoint definitions (client, base_url, model, credential_ref, context_window_tokens) | empty | Fallback endpoints. |
| models.turn_token_budget | integer | 3,000,000 | Total input plus output tokens per user turn across main and worker calls. |
| models.capability_check_on_start | boolean | true | Run capability checks at startup and reload. |

## 7. Limits

- Per-slot concurrency and the per-turn token budget as configured.
- Output length per call as configured; when a response ends because of the limit, the finish reason says so and the caller decides whether to continue.
- No limit on the number of endpoints, but each slot has one primary endpoint at a time, with fallbacks used only on failure (no load balancing across endpoints in the MVP; a customer who needs balancing puts it in front of the endpoint).

## 8. Failure modes and recovery

| Failure | Detection | Effect on user | Recovery |
|---|---|---|---|
| Primary endpoint down | Transient errors after retries | Slower answer while failing over; none if a fallback works | Automatic failover; slot shown degraded |
| All endpoints of the main slot down | Failover exhausted | Turn ends with "the language model is unavailable"; question saved for retry | Operations restore the endpoint; user retries |
| Worker slot down | Failover exhausted | Main Agent answers what it can directly and reports that analysis could not run | As above |
| Background slot down | Failover exhausted | None visible for learning and inference, which are delayed; uncertain entries remain verified before use. The model-based answer check cannot run while it is set to this slot (see [Main Agent](main-agent.md)) | Background Jobs retry later |
| Model lacks a required capability, or its configured context window is below the slot's minimum | Capability check | Slot unavailable from startup | Operations choose another model |
| Context overflow | Endpoint error or pre-check | None visible if compaction succeeds | Caller compacts and retries |
| Credential revoked | auth_failed | As endpoint down | Operations update the secret and reload |
| Token budget exhausted | Router | Answer delivered with partial results and stated coverage | User may ask a narrower question |

## 9. Security considerations

- **Data stays where the customer sends it.** The product sends prompts only to the endpoints configured by the customer. The product sends no automatic telemetry to its makers, and no prompt content goes anywhere else ([ADR 0041](../decisions/0041-no-telemetry-to-the-makers.md)).
- **Credentials.** Model credentials are read at startup from the environment variables or the mounted secret file that `credential_ref` names, held in memory by the router, never written to the database, logs, transcripts, or the Sandbox. The Analysis Worker loop runs in the service, so the Sandbox never needs model access.
- **What models see.** Models see Aras data the signed-in user could see, as returned by the Tool Gateway. Customers choosing an external endpoint accept that this data leaves their network; the deployment guide should state this plainly (see [deployment and operations](../architecture/deployment-and-operations.md)).
- **Logs.** Router logs carry identifiers, token counts, latency, and error categories, never message content.
- **Background prompts.** Discovery inference receives configuration only; batched learning receives transcripts, which are private conversation records already stored on the customer's server.

## 10. Observability

- model_calls rows provide per-slot, per-model, per-conversation, and per-job usage, which also supports the business metrics in [value and ROI](../business/value-and-roi.md). No separate audit event is written per model call.
- Metrics: calls, tokens, latency percentiles, and errors by slot, endpoint, and category; failover count; slot health; queue wait times; budget exhaustion count.
- Audit events `config.reloaded` (who, when, which slots changed) and `model.slot_health_changed` (slot, old and new status) (see [Audit and progress](audit-and-progress.md#51-audit-event-catalogue)).

## 11. MVP versus deferred

In MVP: three slots, the OpenAI-compatible client, retry and failover, slot-specific capability checks, per-slot concurrency with live calls ahead of batch calls, per-turn token budget, usage records.

Not in MVP (see [roadmap](../product/roadmap.md)):

- native model clients (none ships; one is added when a customer's provider needs it);
- per-user daily token budgets (the per-turn budget is the MVP limit; a daily budget is added if the pilot shows runaway use);
- embedding models (none needed; any future vector index for files or knowledge would add an embedding slot);
- model fine-tuning on customer data (not supported);
- response caching (deferred with query result caching);
- usage and ROI dashboard on top of model_calls (deferred).
