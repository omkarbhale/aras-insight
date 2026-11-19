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
| Audit queries | Let operators query audit events by user, time, conversation, job, event type, and outcome; export them for a customer's own review. |
| Progress publishing | Accept progress events, store them with a per-conversation sequence number, and fan them out to subscribed Chat App clients. |
| Progress replay | Serve events after a given sequence number so a reconnecting client catches up. |
| Usage metrics | Derive usage and quality metrics from audit events, turns, feedback, and model calls (5.4). |
| Retention | Purge events past their retention period. |

### Non-responsibilities

| Not done here | Done by |
|---|---|
| Deciding what to audit inside a component's own logic | Each component emits the events listed in 5.1 |
| Operational logs and traces (service health) | Deployment logging, see [deployment-and-operations.md](../architecture/deployment-and-operations.md) |
| A usage or ROI dashboard | Deferred; MVP provides the metrics as queryable data and an export |
| Storing Aras data values for history | Not done; the product keeps no history of Aras data |
| Model usage accounting detail | [Model Router](model-configuration.md) owns `model_calls`; this module only reads it for metrics |

## 3. Interfaces

### 3.1 Internal operations

| Operation | Caller | Inputs | Outputs | Errors |
|---|---|---|---|---|
| Record audit event | All components | event type, actor component, user id, optional conversation, turn, tool call, job ids, outcome, event-specific details (5.1) | audit event id | INVALID_INPUT (unknown type or missing required detail) |
| Publish progress | Main Agent, Analysis Worker (through `report_progress`), Background Jobs | conversation id, optional turn and job ids, source, text, optional kind | progress event id and sequence number | INVALID_INPUT (text over limit, unknown conversation) |
| Subscribe | Chat App API | conversation id, user id, last sequence seen | ordered event stream | NOT_FOUND (conversation not owned by user) |
| Replay | Chat App API | conversation id, user id, after sequence, limit | events | NOT_FOUND |
| Query audit | Operator interface | filters: time range, user, event types, conversation, job, outcome; paging cursor | audit events | INVALID_INPUT |
| Export audit | Operator interface | same filters, format | a downloadable file of matching events | INVALID_INPUT |
| Usage report | Operator interface | time range, grouping (day, week, user role where known) | metric values (5.4) | INVALID_INPUT |

Recording an audit event is synchronous with the action it records for Aras queries and knowledge changes: if the audit write fails, the action is not performed (5.2). Progress publishing is best-effort.

The operator interface is **Proposed** as a small set of service endpoints reachable only by deployment administrators, plus database access for the customer's own reporting tools. A graphical dashboard is deferred.

### 3.2 Progress event kinds

| Kind | Source | Example text |
|---|---|---|
| `step` | Main Agent | "Looking up the change workflow definition" |
| `worker_step` | Analysis Worker via [`report_progress`](tool-catalog.md#report_progress) | "Fetched 1,240 change records; grouping by month" |
| `job_step` | Background Jobs | "Scanned 3,100 of 9,800 files" |
| `budget` | Throttle | "80 percent of this question's query budget used" |
| `state` | Main Agent | turn started, finished, waiting for you |

## 4. Data owned

| Table | Content | Schema |
|---|---|---|
| `audit_events` | Append-only audit record (5.1). | [persistence.md#audit_events](persistence.md#audit_events) |
| `progress_events` | Progress lines per conversation with sequence numbers. | [persistence.md#progress_events](persistence.md#progress_events) |

Metrics read `turns`, `tool_calls`, `feedback`, `citations`, `background_jobs`, `knowledge_state_events`, and `model_calls` without owning them.

## 5. Behaviour

### 5.1 Audit event catalogue

Every audit event has common fields: id, time, event type, actor component, user id (the Aras user on whose behalf the action ran, or none for system actions such as retention purges), conversation, turn, tool call, and job ids where they apply, outcome (`ok`, `rejected`, `denied`, `failed`), and a details object whose required fields depend on the type.

| Event type | Emitted by | When | Required details |
|---|---|---|---|
| `auth.sign_in` | Chat App | Successful sign-in | sign-in method, Aras login name |
| `auth.sign_in_failed` | Chat App | Rejected sign-in | sign-in method, reason code (no password or token material) |
| `auth.sign_out` | Chat App | Sign-out or session revocation | reason (user, expiry, administrator) |
| `credential.issued` / `credential.refreshed` / `credential.expired` / `credential.dropped` | Tool Gateway | Aras credential obtained, refreshed, lost, or deliberately discarded (job credentials are dropped when their job ends) | credential kind (session or job), job id for job credentials; never token material |
| `turn.started` / `turn.finished` | Main Agent | A turn begins, or ends in any final status | turn id, reply-to kind if any; on finish: status, outcome, failure reason, steps, workers started |
| `aras.query` | Tool Gateway | Every AML request sent to Aras, including those made for Discovery, workers, background jobs, and file access checks | calling tool (`aml_query`, `aml_fetch`, discovery, job, file access), AML text (**Proposed**: stored in full, see section 9), item types touched, analyzer cost class, analyzer diagnostics summary, rows returned, bytes, duration, Aras error code if any |
| `aras.query_rejected` | Tool Gateway | Query Analyzer rejected a query, or the Throttle refused it | rule ids or throttle reason, AML text |
| `file.access` | File Service | Every file use: access check through Aras, fetch, extraction, or serving of stored text | file Aras id, file name, action (check, fetch, extract, serve), result (allowed, denied, unreadable, unsupported), bytes, characters served |
| `sandbox.write` | Tool Gateway | Data written into a Sandbox | sandbox id, path, source (AML result or file text), rows or characters |
| `sandbox.created` / `sandbox.destroyed` | Analysis Worker and Sandbox | Sandbox lifecycle | sandbox id, isolation (container or VM), reason for destruction (conversation expired, failure, operator) |
| `sandbox.exec` | Analysis Worker and Sandbox | Each `python_run` execution | sandbox id, analysis job id, duration, exit status, peak memory; never the code output |
| `analysis.started` / `analysis.finished` | Analysis Worker | Job lifecycle | analysis job id, status, steps, rows and files used |
| `job.proposed` / `job.confirmed` / `job.declined` / `job.expired` / `job.started` / `job.paused` / `job.resumed` / `job.finished` / `job.cancelled` | Background Jobs, Chat App | Background job lifecycle | job kind, scope summary, units done and total, status, reason for pause, failure, or expiry |
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

### 5.2 Recording rules

1. Aras queries: the Tool Gateway writes the `aras.query` event after Aras returns and before returning the result to the caller. If the audit write fails, the result is discarded and the caller receives INTERNAL. Before sending, the gateway records nothing separately; a query that never returns (timeout) is recorded with outcome `failed` and the timeout. This guarantees that no data reaches a model without an audit record.
2. Knowledge changes: written in the same database transaction as the change itself.
3. File access: written by the File Service in the same way as Aras queries; a file's text is not served without a `file.access` record.
4. Sign-in and job lifecycle: written at the moment of the transition, in the same transaction as the state change where the state lives in the database.
5. Audit events are never updated or deleted except by the retention purge. Corrections are new events.

### 5.3 Progress events

1. A component publishes a progress line with a source and short text (at most 300 characters, **Proposed**).
2. The module assigns the next sequence number for the conversation, stores the event, and pushes it to every live subscription for that conversation. Events for background jobs are stored under the originating conversation and also visible in the jobs view.
3. Rate limit: a single source may publish at most one line per second (**Proposed**); extra lines within the second are coalesced, keeping the latest. This keeps worker chatter readable.
4. Subscribers are checked for ownership: only the conversation's user receives its events.
5. A reconnecting client passes the last sequence it saw and receives everything after it, up to the replay limit configured in the [Chat App](chat-app.md#6-configuration).
6. Progress text is written by models and can contain anything a model produced. It is stored and delivered as plain text and rendered as text by the Chat App. It must not carry data values beyond what the user could see in the answer anyway; progress is visible only to the same user, so this is a readability rule, not a security boundary.

### 5.4 Usage metrics

Metrics are computed from stored records on request (**Proposed**: by a periodic aggregation into a summary the operator can query or export; no live dashboard in MVP). They exist so a customer can track adoption and value in a pilot ([pilot plan](../business/pilot-plan.md)) and so the business case in [value and ROI](../business/value-and-roi.md) can be measured.

| Metric | Definition | Source |
|---|---|---|
| Active users | Distinct users with at least one turn in the period | `turns` |
| Questions answered | Turns with outcome `answer` | `turns` |
| Answer mix | Share of direct lookups, analysis answers, background job answers | `tool_calls`, `analysis_jobs`, `background_jobs` |
| Time to answer | Median and 90th percentile from question to answer, per answer mix | `turns` |
| Helpful rate | Helpful ratings divided by all ratings | `feedback` |
| Correction rate | Corrections per 100 answers | `feedback` |
| Citation follow-through | Share of answers where the user opened at least one citation; citation opens per answer | Citation-open events reported by the Chat App (**Proposed**, see open question 4) |
| Coverage quality | Share of answers whose coverage reports unreadable files or budget limits | `turns` |
| Query load on Aras | Queries and rows per hour, by cost class; analyzer rejection rate; throttle refusals | `audit_events` |
| Knowledge growth and health | Entries by layer, kind, and state; state changes per period; contested entries open | `knowledge_entries`, `knowledge_state_events` |
| Model usage | Tokens and calls per slot, per question | `model_calls` |
| Estimated time saved | Questions answered, by answer mix, times a per-mix time-saved estimate the customer configures | Derived; the estimate is the customer's own input |

Metrics are reported per period and, where the customer wants it, per Aras identity group the user belongs to (**Proposed**; requires recording the user's groups at sign-in, see open question 5). Per-user figures are available only to deployment administrators.

### 5.5 Retention

1. A purge runs daily (**Proposed**) and removes progress events older than 7 days and audit events older than 365 days (Proposed defaults, configurable).
2. Each purge writes a `retention.purged` event.
3. Deleting a conversation does not delete its audit events; the details of those events never contain message text, only AML text, identifiers, and counts. AML text can carry literal criteria values, so audit events are one of the few places, besides transcripts and extracted file text, where Aras-derived values are kept: for `audit.retention_days` (365 days by default), subject to `audit.store_aml_text`. Progress events of a deleted conversation are deleted with it; otherwise progress lines, which may summarise data, are kept for 7 days.
4. Customers with stricter rules can shorten retention; customers who must keep audit records longer can export them before purge.

## 6. Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `audit.retention_days` | integer | 365 (Proposed default) | Audit event retention. |
| `audit.store_aml_text` | enum: `full`, `redacted`, `none` | `full` (Proposed default) | Whether AML text is stored in audit details; `redacted` replaces literal criteria values with placeholders. |
| `audit.purge_time` | time of day | 02:00 local (Proposed default) | When the daily purge runs. |
| `progress.retention_days` | integer | 7 (Proposed default) | Progress event retention. |
| `progress.max_text_chars` | integer | 300 (Proposed default) | Longest progress line. |
| `progress.min_interval_ms` | integer | 1,000 (Proposed default) | Coalescing window per source. |
| `metrics.time_saved_minutes` | map from answer mix to minutes | none | Customer's own estimate used for the time-saved metric. |
| `metrics.aggregation_interval_minutes` | integer | 60 (Proposed default) | How often summaries are refreshed. |

## 7. Limits

| Limit | Value |
|---|---|
| Progress text length | `progress.max_text_chars` |
| Progress lines per source | One per `progress.min_interval_ms` |
| Live subscriptions per user | 10 (Proposed default), one per open tab and conversation |
| Audit query page size | 1,000 events (Proposed default) |
| Audit export size | 1,000,000 events per export (Proposed default); larger ranges are split |

## 8. Failure modes and recovery

| Failure | Detection | Effect on user | Recovery |
|---|---|---|---|
| Audit write fails for an Aras query | Database error | The tool call fails with INTERNAL; the model may retry | Database recovers; no data was released without audit. |
| Audit write fails for a knowledge change | Transaction fails | The change does not happen | Learner or Main Agent retries later. |
| Progress store or fan-out fails | Publish error | Missing progress lines; answer unaffected | Best-effort; later lines continue. |
| Subscriber falls behind | Buffer full | Client is disconnected and replays | Client replays from last sequence. |
| Purge fails | Missing `retention.purged` event, error log | None | Next day's purge catches up. |
| Metrics aggregation fails | Error log, stale summary time | Operator sees old figures with their time | Next run recomputes from source records. |

## 9. Security considerations

1. **Audit is a sensitive record.** It holds who asked what and the AML each user ran, which can reveal what someone was looking for and can contain literal values from criteria. Access is limited to deployment administrators; it is never shown to other end users. `audit.store_aml_text` lets a customer choose redacted or no AML text.
2. Audit details never contain query results, file text, message text, passwords, or tokens.
3. Append-only: the service's database role used for normal operation can insert but not update or delete audit rows (**Proposed**); only the purge job's role can delete, and only past retention.
4. Progress events are delivered only to the conversation's owner and may contain summaries of data the user is allowed to see; they are deleted after 7 days.
5. Usage metrics with per-user detail are administrator-only. Aggregated metrics can be shared with business owners.
6. See [security-and-permissions.md](../architecture/security-and-permissions.md) for the overall model.

## 10. Observability

The module observes itself through:

| Signal | Content |
|---|---|
| Metrics | Audit write latency and failures; progress events per second; live subscriptions; replay requests; purge rows removed. |
| Alerts (Proposed) | Any audit write failure; purge not run for 2 days. |

## 11. MVP versus deferred

| In MVP | Deferred |
|---|---|
| Closed audit catalogue for every Aras query, file access, job, knowledge change, sign-in | ROI and usage metrics dashboard ([roadmap](../product/roadmap.md)) |
| Progress stream with replay | Notifications outside the app |
| Queryable and exportable usage metrics | Scheduled checks that learned knowledge holds no data values |
| Retention purge | Per-customer audit forwarding integrations (Proposed future item) |

## 12. Open questions

1. **Proposed**: AML text stored in full by default, with `redacted` and `none` options. Some customers may prefer `redacted` as the default.
2. **Proposed** default retention periods (365 days audit, 7 days progress) and daily purge.
3. **Proposed**: an operator interface of service endpoints plus direct database access for reporting; no graphical dashboard in MVP.
4. **Proposed**: citation opens are counted to measure citation follow-through. This requires the Chat App to report each citation open, which adds an audit event type `citation.opened` not yet in the catalogue; to be confirmed.
5. **Proposed**: grouping metrics by the user's Aras identity groups needs those groups recorded at sign-in; open whether that is wanted.
6. **Proposed**: append-only enforcement through separate database roles.
7. **Proposed**: progress rate limit of one line per second per source and 300-character lines.
