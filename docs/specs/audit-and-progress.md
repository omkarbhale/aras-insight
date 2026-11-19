# Audit and progress specification

Status: MVP specification. Related: [Tool Gateway](tool-gateway.md), [Main Agent](main-agent.md), [Chat App](chat-app.md), [Background Jobs](background-jobs.md), [Knowledge and learning](knowledge-and-learning.md), [Model configuration](model-configuration.md), [Persistence](persistence.md), [Value and ROI](../business/value-and-roi.md), [Security and permissions](../architecture/security-and-permissions.md), [Deployment and operations](../architecture/deployment-and-operations.md).

## 1. Purpose

This module records two kinds of events.

- **Audit events** are the durable, append-only record of everything the product does on a user's behalf: every query sent to Aras, every file accessed, every job, every change to shared knowledge, and every sign-in. They answer "who asked what, what did the product read as them, and what did it change in its own knowledge". They also feed the usage metrics a buyer needs to evaluate value.
- **Progress events** are short-lived lines that show the user what is happening while an answer is being built, so long-running work is visible and the user can leave and return.

Both are written by many components and served by one module, so every component records events the same way.

## 2. Responsibilities and non-responsibilities

### Responsibilities

| Area | Responsibility |
|---|---|
| Audit recording | Accept audit events from all components, validate them against the event catalogue (5.1), and store them append-only. |
| Audit queries | Let operators read audit events by user, time, conversation, job, event type, and outcome through documented database views; export them for a customer's own review. |
| Progress publishing | Accept progress events, store them with a per-conversation sequence number, and fan them out to subscribed Chat App clients. |
| Progress replay | Serve events after a given sequence number so a reconnecting client catches up. |
| Usage metrics | Define the usage and quality metrics (5.4), derive them from audit events, turns, feedback, and model calls, and keep them in `usage_summaries` so they outlive their source rows. |
| Retention | Run the daily purge for every table that has a retention rule in [persistence.md](persistence.md#7-retention-summary) (5.5). |

### Non-responsibilities

| Not done here | Done by |
|---|---|
| Deciding what to audit inside a component's own logic | Each component emits the events listed in 5.1 |
| Operational logs and traces (service health) | Deployment logging, see [deployment-and-operations.md](../architecture/deployment-and-operations.md) |
| A usage or ROI dashboard | Deferred; MVP provides the metrics as database views and an export |
| Administrator endpoints or an administrator sign-in | None in the MVP; operators use database access and the export command (3.1) |
| Storing Aras data values for history | Not done; the product keeps no history of Aras data |
| Model usage accounting detail | [Model Router](model-configuration.md) owns `model_calls`; this module only reads it for metrics |

## 3. Interfaces

### 3.1 Internal operations

| Operation | Caller | Inputs | Outputs | Errors |
|---|---|---|---|---|
| Record audit event | All components | event type, actor component, user id, optional conversation, turn, tool call, job ids, outcome, event-specific details (5.1) | audit event id | INVALID_INPUT (unknown type or missing required detail) |
| Publish progress | Main Agent, Analysis Worker (through `report_progress`), Background Jobs, Throttle (through the Tool Gateway), the service itself | conversation id, optional turn and job ids, source, text, optional kind | progress event id and sequence number | INVALID_INPUT (text over limit, unknown conversation) |
| Subscribe | Chat App API | conversation id, user id, last sequence seen | ordered event stream | NOT_FOUND (conversation not owned by user) |
| Replay | Chat App API | conversation id, user id, after sequence, limit | events | NOT_FOUND |
| Query audit | Operator, through a database view | filters: time range, user, event types, conversation, job, outcome; paging cursor | audit events | none (a database read) |
| Export audit | Operator, through the export command | same filters, format | a file of matching events | INVALID_INPUT |
| Usage report | Operator, through database views over `usage_summaries` | time range, grouping (day, week, user) | metric values (5.4) | none (a database read) |

Recording an audit event is synchronous with the action it records for Aras queries and knowledge changes: if the audit write fails, the action is not performed (5.2). Progress publishing is best-effort.

Operators read audit events and usage figures through documented database views and an export command run by the operator; the service offers no endpoints for them and the product has no administrator sign-in ([ADR 0040](../decisions/0040-operator-reports-without-administrator-endpoints.md)). Operators are the customer's deployment administrators: the infrastructure staff who run the deployment and hold the database access it needs, not Aras users acting in the product. These reports are deployment tooling for them, not an administrator feature of the product, so they sit outside the MVP's deferral of features for Aras administrators and developers. A graphical dashboard is deferred.

### 3.2 Progress event kinds

| Kind | Source | Example text |
|---|---|---|
| `step` | Main Agent | "Looking up the change workflow definition" |
| `worker_step` | Analysis Worker via [`report_progress`](tool-catalog.md#report_progress) | "Fetched 1,240 change records; grouping by month" |
| `job_step` | Background Jobs | "Scanned 3,100 of 9,800 files" |
| `budget` | Throttle | "80 percent of this question's query budget used" |
| `state` | Main Agent | turn started, finished, waiting for you |

The allowed publishers are the Main Agent, Analysis Workers, Background Jobs, the Throttle (budget lines, published through the Tool Gateway), and the service itself (source `system`).

## 4. Data owned

| Table | Content | Schema |
|---|---|---|
| `audit_events` | Append-only audit record (5.1). | [persistence.md#audit_events](persistence.md#audit_events) |
| `progress_events` | Progress lines per conversation with sequence numbers. | [persistence.md#progress_events](persistence.md#progress_events) |
| `usage_summaries` | Aggregated usage figures by period, metric, and dimension: counts only, never Aras data values (5.4). | [persistence.md#usage_summaries](persistence.md#usage_summaries) |

Metrics read `turns`, `tool_calls`, `analysis_jobs`, `feedback`, `citations`, `background_jobs`, `learning_signals`, `knowledge_entries`, `knowledge_state_events`, and `model_calls` without owning them.

## 5. Behaviour

### 5.1 Audit event catalogue

Every audit event has common fields: id, time, event type, actor component, user id (the Aras user on whose behalf the action ran, or none for system actions such as retention purges), conversation, turn, tool call, and job ids where they apply, outcome (`ok`, `rejected`, `denied`, `failed`), and a details object whose required fields depend on the type.

| Event type | Emitted by | When | Required details |
|---|---|---|---|
| `auth.sign_in` | Chat App | Successful sign-in through the Aras authorization server | Aras login name |
| `auth.sign_in_failed` | Chat App | Rejected sign-in | reason code (never token material) |
| `auth.sign_out` | Chat App | Sign-out or session revocation | reason (user or expiry) |
| `credential.issued` / `credential.refreshed` / `credential.expired` / `credential.dropped` | Tool Gateway | Aras credential obtained, refreshed, lost, or deliberately discarded (job credentials are dropped when their job ends) | credential kind (session or job), job id for job credentials; never token material |
| `turn.started` / `turn.finished` | Main Agent | A turn begins, or ends in any final status | turn id, reply-to kind if any; on start: product version; on finish: status, outcome, failure reason, steps, workers started |
| `aras.query` | Tool Gateway | Every AML request sent to Aras, including those made for Discovery, workers, background jobs, and file access checks | calling tool (`aml_query`, `aml_fetch`, discovery, job, file access), AML text (as `audit.store_aml_text` allows, `redacted` by default; see section 9), query fingerprint, item types touched, analyzer cost class, analyzer diagnostics summary, rows returned, bytes, duration, Aras error code if any |
| `aras.query_rejected` | Tool Gateway | Query Analyzer rejected a query, or the Throttle refused it | rule ids or throttle reason, query fingerprint, AML text (as `audit.store_aml_text` allows) |
| `file.access` | File Service | Every file use: access check through Aras, fetch, extraction, or serving of stored text | file Aras id, file name, action (check, fetch, extract, serve), result (allowed, denied, unreadable, unsupported), bytes, characters served |
| `sandbox.write` | Tool Gateway | Data written into a Sandbox | sandbox id, path, source (AML result or file text), rows or characters |
| `sandbox.created` / `sandbox.destroyed` | Analysis Worker and Sandbox | Sandbox lifecycle | sandbox id, isolation (container or VM), reason for destruction (conversation expired, failure, operator) |
| `sandbox.exec` | Analysis Worker and Sandbox | Each `python_run` execution | sandbox id, analysis job id, duration, exit status, peak memory; never the code output |
| `analysis.started` / `analysis.finished` | Analysis Worker | Job lifecycle | analysis job id, status, steps, rows and files used; on start: product version |
| `job.proposed` / `job.confirmed` / `job.declined` / `job.expired` / `job.started` / `job.paused` / `job.resumed` / `job.finished` / `job.cancelled` | Background Jobs, Chat App | Background job lifecycle | job kind, scope summary, units done and total, status, reason for pause, failure, or expiry; on `job.started`: product version |
| `knowledge.created` / `knowledge.updated` / `knowledge.merged` | Knowledge Service | Shared entry written or changed | entry id, layer, kind, version, source (live learning, batched learning, discovery), conversation id for learned entries |
| `knowledge.state_changed` | Knowledge Service | State transition | entry id, from state, to state, trigger (the `knowledge_state_events` trigger: `config_renamed`, `config_deleted`, `config_changed_rederived`, `user_correction`, `analyzer_failure`, `user_contradiction`, `verified`, `merged`, or `clarified`; see [persistence.md](persistence.md#knowledge_state_events)). A confirmation changes the confirmation status, not the state, and is recorded as `knowledge.confirmed` |
| `knowledge.confirmed` / `knowledge.objected` | Knowledge Service | Independent confirmation or objection recorded | entry id, implicit or explicit |
| `knowledge.contradiction_recorded` / `knowledge.contradiction_resolved` | Knowledge Service | A disagreement between users is recorded on an entry, or resolved | entry id, contradiction id, resolution on resolve. Retirement of an entry is recorded as `knowledge.state_changed` |
| `preference.written` | Knowledge Service | A user preference entry is created or changed | entry id, owner user id, version |
| `preference.deleted` | Chat App | User deleted a preference | entry id |
| `config.reloaded` | Service | Deployment configuration reloaded | changed keys (never secret values) |
| `model.slot_health_changed` | Model Router | A model slot's endpoint becomes unhealthy, recovers, or fails over | slot, endpoint name, from and to health, reason category |
| `discovery.run` | Discovery | Each change check or regeneration | run id, items checked, changed, removed |
| `conversation.deleted` | Chat App | User deleted a conversation | conversation id |
| `feedback.submitted` | Chat App | Rating or correction | feedback id, rating, whether it flagged citations |
| `retention.purged` | This module | Retention run | table, rows removed, cutoff time |

The catalogue is closed: a component cannot write an event type not listed, which keeps the record reviewable. Adding a type is a product change to this specification.

Two things are deliberately not audit events. Model calls are recorded only in `model_calls` (slot, endpoint, tokens, duration; no prompt or completion content), owned by the Model Router. Citation opens are counted on the `citations` row (5.4), not logged one by one.

The start events (`turn.started`, `analysis.started`, `job.started`) carry the product release version. Prompts, skills, and the service are versioned and upgraded together as one release, so one version names the guidance behind an answer; per-skill versions are not recorded, and `messages.prompt_version` stays on the transcript. See [ADR 0039](../decisions/0039-audit-and-usage-records.md) for the audit and usage records.

### 5.2 Recording rules

1. Aras queries: the Tool Gateway writes the `aras.query` event after Aras returns and before returning the result to the caller. If the audit write fails, the result is discarded and the caller receives INTERNAL. Before sending, the gateway records nothing separately; a query that never returns (timeout) is recorded with outcome `failed` and the timeout. This guarantees that no data reaches a model without an audit record.
2. Knowledge changes: written in the same database transaction as the change itself.
3. File access: written by the File Service in the same way as Aras queries; a file's text is not served without a `file.access` record.
4. Sign-in and job lifecycle: written at the moment of the transition, in the same transaction as the state change where the state lives in the database.
5. Audit events are never updated or deleted except by the retention purge. Corrections are new events.

### 5.3 Progress events

1. A component publishes a progress line with a source and short text (at most `progress.max_text_chars`, 300 characters by default). The allowed publishers are listed in 3.2.
2. The module assigns the next sequence number for the conversation, stores the event, and pushes it to every live subscription for that conversation. Events for background jobs are stored under the originating conversation and also visible in the jobs view.
3. Rate limit: each publishing worker, job, or turn may publish at most one line per `progress.min_interval_ms` (one second by default); extra lines within the window are coalesced, keeping the latest. The limit is per worker, not shared by the whole source kind, so parallel workers do not swallow each other's lines. This keeps worker chatter readable.
4. Subscribers are checked for ownership: only the conversation's user receives its events.
5. A reconnecting client passes the last sequence it saw and receives everything after it, up to the replay limit configured in the [Chat App](chat-app.md#6-configuration).
6. Progress text is written by models and can contain anything a model produced. It is stored and delivered as plain text and rendered as text by the Chat App. It must not carry data values beyond what the user could see in the answer anyway; progress is visible only to the same user, so this is a readability rule, not a security boundary.

### 5.4 Usage metrics

Metrics are computed by a periodic aggregation (every `metrics.aggregation_interval_minutes`) into `usage_summaries`, which the operator reads through database views or exports; there is no live dashboard in the MVP. The summaries hold counts only, never Aras data values, and are kept longer than the records they are computed from, so figures survive the retention purge (see [persistence.md](persistence.md#7-retention-summary)). They exist so a customer can track adoption and value in a pilot ([pilot plan](../business/pilot-plan.md)) and so the business case in [value and ROI](../business/value-and-roi.md) can be measured. This section is the single definition of the metrics; the business case refers to it.

| Metric | Definition | Source |
|---|---|---|
| Active users | Distinct users with at least one turn in the period | `turns` |
| Repeat use | Share of the users active in a period who were also active in an earlier period | `turns` (derived) |
| Questions answered | Turns with outcome `answer` | `turns` |
| Answer mix | Share of direct lookups, analysis answers, background job answers | `tool_calls`, `analysis_jobs`, `background_jobs` |
| Time to answer | Median and 90th percentile from question to answer, per answer mix | `turns` |
| Analysis depth | Records examined and files read per answer, per answer mix | `analysis_jobs`, `audit_events` (`aras.query` rows returned, `file.access`) |
| Helpful rate | Helpful ratings divided by all ratings | `feedback` |
| Correction rate | Corrections per 100 answers: corrections submitted through feedback plus corrections the Learner detected as a learning signal, counted once per answer | `feedback`, `learning_signals` |
| Clarification rate | Turns with outcome `clarification` as a share of all turns | `turns` |
| Citation follow-through | Share of answers where the user opened at least one citation; citation opens per answer | `citations`: an opened count and first-opened time on each row, updated by the Chat App when a user opens a citation; no per-click log is kept |
| Coverage quality | Share of answers whose coverage reports unreadable files or budget limits | `turns` |
| Query load on Aras | Queries and rows per hour, by cost class; analyzer rejection rate; throttle refusals | `audit_events` |
| Knowledge growth and health | Entries by layer, kind, state, and confirmation status; state changes per period; contested entries open and the age of the oldest | `knowledge_entries`, `knowledge_state_events` |
| Model usage | Tokens (input plus output) and calls per slot, per question | `model_calls` |
| Cost per answer | Tokens per answer multiplied by the customer's own per-token rates, plus the customer's infrastructure share per answer | `model_calls`; the rates and the infrastructure share are the customer's own input, and the product stores no prices |
| Estimated time saved | Answers by answer mix times the minutes saved per answer that the customer configures for that mix (`metrics.time_saved_minutes`) | Derived; the minutes come from the customer's own baseline timing taken at the start of the pilot (the manual effort for a comparable question minus the typical time to answer), so the metric needs one customer input and no per-answer collection |

Metrics are reported per period, per user, and in total. They are not grouped by Aras identity group, because the product does not read or record a user's groups; a customer that wants a per-role report joins the exported per-user figures to its own Aras identity lists. Per-user figures are available only to operators.

### 5.5 Retention

1. A purge runs daily at `audit.purge_time`. It removes progress events older than `progress.retention_days` (7 days by default) and audit events older than `audit.retention_days` (365 days by default), and it applies the retention rules of the other tables in [persistence.md](persistence.md#7-retention-summary).
2. Each purge writes a `retention.purged` event.
3. Deleting a conversation does not delete its audit events; the details of those events never contain message text, only AML text (as `audit.store_aml_text` allows), identifiers, and counts. AML text stored in full can carry literal criteria values (`redacted`, the default, replaces them with placeholders), so audit events are one of the few places, besides transcripts and extracted file text, where Aras-derived values can be kept: for `audit.retention_days` (365 days by default), subject to `audit.store_aml_text`. Progress events of a deleted conversation are deleted with it; otherwise progress lines, which may summarise data, are kept for 7 days.
4. Customers with stricter rules can shorten retention; customers who must keep audit records longer can export them before purge.

## 6. Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `audit.retention_days` | integer | 365 | Audit event retention. |
| `audit.store_aml_text` | enum: `full`, `redacted`, `none` | `redacted` | How AML text is stored in audit details: `full`; `redacted`, which replaces literal criteria values with placeholders and keeps the query shape and its fingerprint; or `none`, which keeps the fingerprint only. Applies to the audit record only (section 9). |
| `audit.purge_time` | time of day | 02:00 local | When the daily purge runs. |
| `progress.retention_days` | integer | 7 | Progress event retention. |
| `progress.max_text_chars` | integer | 300 | Longest progress line. |
| `progress.min_interval_ms` | integer | 1,000 | Coalescing window per publishing worker, job, or turn. |
| `metrics.time_saved_minutes` | map from answer mix (`direct_lookup`, `analysis`, `background_job`) to minutes | none | Minutes saved per answer for each answer mix, set from the customer's own baseline timing; used for the time-saved metric. |
| `metrics.aggregation_interval_minutes` | integer | 60 | How often `usage_summaries` are refreshed. |

The retention keys of the other tables (`retention.conversation_days`, `retention.model_call_days`, `file_service.retention_days_unused`) are listed in [persistence.md](persistence.md#9-configuration) and applied by the same daily purge.

## 7. Limits

| Limit | Value |
|---|---|
| Progress text length | `progress.max_text_chars` |
| Progress lines per publishing worker, job, or turn | One per `progress.min_interval_ms` |
| Live subscriptions per user | 10 by default, one per open tab and conversation |
| Audit query page size | 1,000 events by default |
| Audit export size | 1,000,000 events per export by default; larger ranges are split |

## 8. Failure modes and recovery

| Failure | Detection | Effect on user | Recovery |
|---|---|---|---|
| Audit write fails for an Aras query | Database error | The tool call fails with INTERNAL; the model may retry | Database recovers; no data was released without audit. |
| Audit write fails for a knowledge change | Transaction fails | The change does not happen | Learner or Main Agent retries later. |
| Progress store or fan-out fails | Publish error | Missing progress lines; answer unaffected | Best-effort; later lines continue. |
| Subscriber falls behind | Buffer full | Client is disconnected and replays | Client replays from last sequence. |
| Purge fails | Missing `retention.purged` event, error log | None | Next day's purge catches up. |
| Metrics aggregation fails | Error log, stale summary time | Operator sees old figures with their time | Next run recomputes the affected periods from the source records still held. |

## 9. Security considerations

1. **Audit is a sensitive record.** It holds who asked what and the AML each user ran, which can reveal what someone was looking for and can contain literal values from criteria. Access is limited to operators with database access (the customer's deployment administrators); it is never shown to other end users. `audit.store_aml_text` lets a customer keep AML text in full, redacted (the default), or not at all. The setting governs the audit record only. The query text also lives in the private transcript (`tool_calls.input` and the assistant's messages), under conversation retention; a customer that wants less retention shortens `retention.conversation_days`.
2. Audit details never contain query results, file text, message text, passwords, or tokens.
3. Append-only: the service's database role used for normal operation can insert but not update or delete audit rows; only the purge job's role can delete, and only past retention. The customer provisions the two roles (see [Deployment and operations](../architecture/deployment-and-operations.md)).
4. Progress events are delivered only to the conversation's owner and may contain summaries of data the user is allowed to see; they are deleted after 7 days.
5. Usage metrics with per-user detail are for operators only. Aggregated metrics can be shared with business owners.
6. See [security-and-permissions.md](../architecture/security-and-permissions.md) for the overall model.

## 10. Observability

The module observes itself through:

| Signal | Content |
|---|---|
| Metrics | Audit write latency and failures; progress events per second; live subscriptions; replay requests; purge rows removed. |
| Alerts | Any audit write failure; purge not run for 2 days. |

## 11. MVP versus deferred

| In MVP | Deferred |
|---|---|
| Closed audit catalogue for every Aras query, file access, job, knowledge change, sign-in | ROI and usage metrics dashboard ([roadmap](../product/roadmap.md)) |
| Progress stream with replay | Notifications outside the app ([roadmap](../product/roadmap.md)) |
| Usage metrics as database views and an export | Scheduled checks that learned knowledge holds no data values |
| Retention purge | Forwarding audit events to a customer's own systems ([roadmap](../product/roadmap.md)); the MVP offers the export only |
