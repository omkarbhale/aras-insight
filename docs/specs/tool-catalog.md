# Tool catalog specification

Status: MVP specification. Related: [Main Agent](main-agent.md), [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md), [Tool Gateway](tool-gateway.md), [Query Analyzer](query-analyzer.md), [File Service](file-service.md), [Knowledge and learning](knowledge-and-learning.md), [Background Jobs](background-jobs.md), [Prompts](../prompts/README.md), [Skills](../skills/README.md).

This document is the single definition of every tool the agents can call. Prompts and skills refer to these tools by the exact names used here. Every other specification that mentions a tool defers to this document for its inputs, outputs, and errors.

Aras item type, relationship, and property names used in the examples below (for example Express ECO, Affected Item, Part, `item_number`) are typical of standard Aras releases and must be verified against the target version; the names of a real installation come from discovery.

## 1. Purpose

The tools are the only way the Main Agent and the Analysis Workers act on the world. They give the model:

- the discovered shape of the customer's Aras installation (`schema_describe`);
- the product's and the organisation's accumulated knowledge (`knowledge_index`, `knowledge_read`);
- read-only access to Aras data and files, always as the signed-in user (`aml_query`, `file_read`, `aml_fetch`, `file_fetch`);
- a sealed place to compute over that data (`python_run`, `sandbox_read`);
- ways to hand work off (`start_analysis`, `start_background_job`), to learn (`record_learning`), and to report (`report_progress`, `submit_report`).

The catalog fixes, for every tool, which agent may call it, what it accepts, what it returns, how it fails, what limits apply, and how its results become citations. A tool definition sent to the model is generated from this catalog; the descriptions the model sees are in [the prompts](../prompts/README.md).

## 2. Responsibilities and non-responsibilities

The catalog is responsible for:

- the exact tool names, argument fields, result fields, and error codes;
- the shared result envelope and diagnostic shape;
- the rule that a tool result that contains Aras data, file text, or discovered or learned knowledge always carries citation ids issued by the Citation Registry.

The catalog is not responsible for:

- how a tool is implemented (see the owning module specification named in each tool entry);
- query validity and cost rules (see [Query Analyzer](query-analyzer.md));
- the throttle algorithm (see [Tool Gateway](tool-gateway.md));
- the wording of the tool descriptions shown to the model (see [Prompts](../prompts/README.md)).

## 3. Interfaces

### 3.1 Tool availability by agent

| Tool | Main Agent | Analysis Worker | Owning module | Touches Aras |
|---|---|---|---|---|
| `schema_describe` | yes | no | [Knowledge Service](knowledge-and-learning.md), from [Discovery](discovery.md)'s stored configuration | no (reads the discovered layer) |
| `knowledge_index` | yes | no | [Knowledge Service](knowledge-and-learning.md) | no |
| `knowledge_read` | yes | no | [Knowledge Service](knowledge-and-learning.md) | no |
| `aml_query` | yes | no | [Tool Gateway](tool-gateway.md) | yes, as the user |
| `file_read` | yes | no | [File Service](file-service.md) | yes, access re-check as the user |
| `start_analysis` | yes | no | [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md) | indirectly |
| `start_background_job` | yes | no | [Background Jobs](background-jobs.md) | indirectly, after user confirmation |
| `record_learning` | yes | no | [Learner](knowledge-and-learning.md) | no |
| `aml_fetch` | no | yes | [Tool Gateway](tool-gateway.md) | yes, as the user |
| `file_fetch` | no | yes | [File Service](file-service.md) | yes, access re-check as the user |
| `python_run` | no | yes | [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md) | no (the Sandbox has no network) |
| `sandbox_read` | no | yes | [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md) | no |
| `report_progress` | no | yes | [Audit and progress](audit-and-progress.md) | no |
| `submit_report` | no | yes | [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md) | no |

**Proposed:** Analysis Workers do not receive `schema_describe`, `knowledge_index`, or `knowledge_read`. The Main Agent places the schema excerpts and knowledge entries a job needs in the job brief, together with their citation ids. This keeps a worker's scope bounded and makes its context reproducible from the brief. See open question 1.

All tools run with the identity of the user who owns the conversation. No tool accepts a user identifier, credential, or Aras URL from the model; those come from the session context the service attaches to every call.

### 3.2 Shared result envelope

Every tool returns one envelope. On success:

| Field | Type | Always present | Meaning |
|---|---|---|---|
| `ok` | boolean | yes | `true`. |
| `result` | object | yes | Tool-specific fields defined in each entry below. |
| `citations` | list of citation summaries | yes (may be empty) | Every citable source this call produced, already registered in the Citation Registry. |
| `warnings` | list of diagnostics | yes (may be empty) | Non-blocking diagnostics, typically Query Analyzer warnings. |
| `usage` | object | yes | `rows` (integer), `bytes` (integer), `duration_ms` (integer), `budget_remaining` (object with `queries`, `rows`, `bytes`, `wall_seconds`). |

On failure:

| Field | Type | Always present | Meaning |
|---|---|---|---|
| `ok` | boolean | yes | `false`. |
| `error.code` | string | yes | One of the shared error codes in section 3.4. |
| `error.message` | string | yes | A short, model-readable explanation, written to help the model decide what to do next. |
| `error.retryable` | boolean | yes | `true` when the same call, possibly corrected, can reasonably succeed. |
| `error.retry_after_ms` | integer | no | Present for `THROTTLED`: the earliest time a retry can be admitted. |
| `error.diagnostics` | list of diagnostics | yes (may be empty) | Structured findings, typically Query Analyzer rejections. |

A tool never returns raw stack traces, internal host names, credentials, or Aras server error pages to the model. Aras faults are reduced to their fault code and fault string (see [Tool Gateway](tool-gateway.md) section 5.4).

### 3.3 Diagnostic shape

| Field | Type | Meaning |
|---|---|---|
| `rule` | string | Rule identifier, for example `QA-V04`. Rules are defined only in [Query Analyzer](query-analyzer.md). Non-analyzer diagnostics use the prefix `GW-` (gateway), `FS-` (File Service), or `LRN-` (Learner, for example `LRN-DATA-VALUE`). |
| `severity` | `reject` or `warn` | `reject` means the call was not executed. `warn` means it was executed and the result may be incomplete or misleading. |
| `path` | string | Location in the submitted input, for example `Item[@type='Part']/Relationships/Item[1]/@select`. Empty when the finding concerns the whole input. |
| `message` | string | What was found, in one or two sentences. |
| `suggestions` | list of strings | Concrete alternatives, for example the closest existing property names or valid list values. |
| `docs` | string | A short hint naming the product skill that explains the topic. |

### 3.4 Shared error codes

| Code | Retryable | Meaning | Typical model reaction |
|---|---|---|---|
| `INVALID_INPUT` | yes | Tool arguments missing or of the wrong type. | Fix the arguments. |
| `ANALYZER_REJECTED` | yes | The Query Analyzer rejected the AML; see diagnostics. | Rewrite the query using the suggestions. |
| `NOT_READ_ONLY` | no | The AML requested something other than an allowed read action. | Do not retry; the product is read-only. Explain to the user if they asked for a change. |
| `ARAS_ERROR` | sometimes | Aras returned a fault for a well-formed query. | Read the fault string; correct and retry once, otherwise report. |
| `ARAS_UNAVAILABLE` | yes | Aras did not respond or returned a server error. | Retry later; tell the user if it persists. |
| `ARAS_AUTH_EXPIRED` | no (for the model) | The user's Aras session expired and could not be renewed. | Stop; the turn ends `failed` with reason `auth_expired`, keeping its work in the transcript. After the user signs in again the Chat App offers "Continue", which resumes from the saved transcript in a new turn. |
| `PERMISSION_DENIED` | no | Aras refused access, or an item or file is not visible to this user. | Do not try to work around it; record it in coverage. |
| `NOT_FOUND` | no | The referenced id (item, file, entry, job, Sandbox path) does not exist or is not visible. | Check the id; record in coverage if relevant. |
| `THROTTLED` | yes | A concurrency or rate limit was reached. | Wait at least `retry_after_ms`; prefer fewer, more selective queries. |
| `BUDGET_EXHAUSTED` | no | The question budget for this turn is used up. | Stop fetching; answer with what is available and report partial coverage. |
| `TIMEOUT` | yes | The call exceeded its time limit (an Aras query, or a `python_run` execution). | Narrow the query (filters, smaller page) or make the code cheaper, and retry. |
| `RESULT_TOO_LARGE` | yes | The result exceeds what the tool may return. | Narrow, page, or hand the work to `start_analysis`. |
| `FILE_UNREADABLE` | no | The file could not be read (for example a scanned PDF without a text layer, or a corrupt file). | Record in coverage as unreadable. |
| `FILE_UNSUPPORTED_FORMAT` | no | The file format is outside the MVP formats. | Record in coverage as unsupported. |
| `SANDBOX_UNAVAILABLE` | yes | The Sandbox could not be created or reached. | Retry once; otherwise submit a report stating the failure. |
| `SANDBOX_EXEC_FAILED` | yes | Python ran but failed (exception, or killed for memory or disk). | Read stderr, fix the code, retry. |
| `PATH_INVALID` | yes | A Sandbox path is outside the Sandbox's `data/` or `work/<job id>/` areas, is malformed, or names an `aml_fetch` file that already exists. | Use a path relative to the Sandbox root, inside `data/` (read only) or the job's `work/<job id>/` area. |
| `JOB_NOT_CONFIRMED` | no | A background job proposal was declined, expired unconfirmed, or already decided; the message says which. | Do not propose it again in this turn unless the user asks. |
| `KNOWLEDGE_CONFLICT` | yes | The same knowledge entry was updated concurrently while the proposal was being written (see [Knowledge and learning](knowledge-and-learning.md)). The Learner retries once before returning it. | Retry the call once; disagreement with existing knowledge is not an error but an outcome of `record_learning`. |
| `INTERNAL` | yes | Unexpected product failure. | Retry once; otherwise tell the user something went wrong. |

### 3.5 Citation summaries

Every `citations` list contains summaries with these fields. The full record lives in the Citation Registry table [citations](persistence.md#citations).

| Field | Type | Meaning |
|---|---|---|
| `id` | string | Citation id issued by the Citation Registry, prefix `cit_`. |
| `kind` | `record`, `file`, or `knowledge` | Citation kind. |
| `label` | string | Human-readable label: keyed name and item type for records; file name and locator for files; entry title for knowledge. |
| `item_type` | string | Records only: the Aras item type name. |
| `aras_id` | string | Records and files: the Aras id. |
| `locator` | string | Files only: page, sheet and cell range, slide, section heading, or character range. |
| `entry_id` | string | Knowledge only: the knowledge entry id, prefix `kn_`. |
| `state` | string | Knowledge only: `confirmed`, `uncertain`, `contested`, or `retired` at the time of citing. |
| `confirmation` | string | Learned knowledge only: `unconfirmed` or `independently_confirmed`. |
| `confidence` | string | Knowledge only: `high`, `medium`, or `low`, computed at the time of citing from the entry's layer, kind, derivation method, state, and confirmation, as defined in [Knowledge and learning](knowledge-and-learning.md#41-knowledge-entry-schema). |

**Knowledge citations.** `knowledge_read` and `schema_describe` both issue knowledge citations. A citation is registered for discovered and learned entries; none is registered for product skills or for the user's own preferences. The same entry is cited once for a given version, state, and confirmation; when any of them has changed since it was last cited in the conversation, a new citation id is issued.

**Citation sets.** A computed number over many records (for example "412 change records closed late") cannot reasonably list every record. Each `aml_fetch` call therefore also registers one citation set, id prefix `cst_`, that groups the record citations of every row it wrote; the Analysis Worker may register a narrower set over the rows its code selected through `submit_report`. A set is cited exactly like a citation, `[cite:cst_Wd81pT4z]`, and the Chat App opens it as a list of its member records, each opening in Aras. A set is a grouping of `record` citations, not a fourth citation kind. Ids are opaque; the examples in this document are illustrative.

**How the model uses citations.** In its final answer the model writes a marker `[cite:<citation id>]` directly after each statement that depends on a source, for example `[cite:cit_Xk29fQm3]`. Only ids that appear in a tool result in the same conversation are valid. The Main Agent validates every marker before the answer is shown (see [Main Agent](main-agent.md)); the Chat App turns valid markers into links that open the record or file in Aras, or the knowledge entry in the product (see [Chat App](chat-app.md)).

## 4. Tool definitions

Each entry lists purpose, caller, inputs, outputs, tool-specific errors, limits, and citation behaviour. Shared envelope fields from section 3.2 are not repeated.

### `schema_describe`

**Purpose.** Return the discovered schema for item types and the configuration around them, so the model writes AML against what actually exists in this installation rather than against assumptions. Serves item types, properties, relationship types, lists, workflow maps, and lifecycle maps through the Knowledge Service, from the configuration that [Discovery](discovery.md) has stored.

**Used by.** Main Agent.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `targets` | list of strings | yes (1 to 20) | Names to describe. Each is an item type name, relationship type name, list name, workflow map name, or lifecycle map name. Matching is case-insensitive on name and label. |
| `target_kind` | `item_type`, `relationship_type`, `list`, `workflow_map`, `lifecycle_map`, or `any` | no (default `any`) | Restricts how `targets` are interpreted. |
| `detail` | `summary` or `full` | no (default `summary`) | `summary`: names, labels, data types, and relationship targets. `full`: adds list values, state and transition names, activity names, and property flags such as required, computed, or federated. |
| `include_relationships` | boolean | no (default `true`) | For item types, include relationship types whose source is the item type. |
| `search` | string | no | Free text matched against names, labels, and descriptions, used when the model does not know the exact name. When present, `targets` may be empty and up to 20 best matches are returned at `summary` detail. |

**Outputs (`result`).**

| Field | Type | Meaning |
|---|---|---|
| `items` | list | One element per resolved target. Each has `kind`, `name`, `label`, `description`, `aras_id`, `modified_on` (configuration stamp), and kind-specific detail: for item types, `properties` (name, label, data type, data source for item properties, list name for list properties, flags), `relationships` (relationship type name, related item type), `lifecycle_maps`, `workflow_maps`, `versionable`; for lists, `values` (value and label); for lifecycle maps, `states` and `transitions`; for workflow maps, `activities` and `paths`. |
| `unresolved` | list of objects | Targets that matched nothing, each with `target` and `suggestions` (closest names). |
| `discovery_stamp` | timestamp | When the discovered layer was last confirmed current. |
| `stale` | boolean | `true` when the last change check is older than the configured interval and a check is in progress. The data is still usable. |

**Tool-specific errors.** `INVALID_INPUT` (no targets and no search, or more than 20 targets). `NOT_FOUND` is never returned; unresolved names go in `unresolved`.

**Limits.** Up to 20 targets per call. A `full` result is capped at 60,000 characters; when exceeded, the largest list-value sets are truncated with a `truncated` flag and the count of omitted values. **Proposed.**

**Citations.** One `knowledge` citation per described configuration element, pointing at the discovered-layer entry for that element (layer `instance`, kind `discovered`, state as stored). The model cites these when an answer depends on how the installation is configured, for example "ECOs pass through four review activities".

### `knowledge_index`

**Purpose.** List the knowledge entries relevant to a topic so the model can decide which to load in full. Covers all three layers: product skills, instance knowledge (discovered and learned), and the user's own preferences. Loading is by index and description; there is no vector search in MVP.

**Used by.** Main Agent.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `topic` | string | yes | Words describing what the model needs, for example "ECO delay definition" or "how to read sign-off comments". |
| `layers` | list of `product_skill`, `instance`, `user_preference` | no (default all) | Layers to include. |
| `kinds` | list of `discovered`, `learned` | no (default both) | For the instance layer only. |
| `include_states` | list of states | no (default `confirmed`, `uncertain`, `contested`) | `retired` entries are excluded unless asked for explicitly. |
| `config_elements` | list of strings | no | Configuration element names (item type, property, list, state, activity); restricts to entries that depend on or refer to any of them. |
| `limit` | integer | no (default and maximum `knowledge.index_max_lines`, Proposed 50) | Maximum entries returned. |

**Outputs (`result`).**

| Field | Type | Meaning |
|---|---|---|
| `entries` | list | Each element: `id`, `layer`, `kind` (instance only), `title`, `description` (one line), `state`, `confidence` (`high`, `medium`, or `low`, computed as defined in [Knowledge and learning](knowledge-and-learning.md#41-knowledge-entry-schema)), `confirmation` (learned only), `config_elements` (names it depends on or refers to), `updated_at`. |
| `total_matches` | integer | Matches before `limit`. |
| `match_method` | string | How matching was done, for example "title and description terms, configuration-element filter". Informational. |

**Tool-specific errors.** `INVALID_INPUT`.

**Limits.** At most `knowledge.index_max_lines` entries per call (Proposed 50). Descriptions longer than 200 characters are truncated in the index.

**Citations.** None. An index line is not a source; the model must load an entry with `knowledge_read` before citing it.

### `knowledge_read`

**Purpose.** Load the full text of knowledge entries so the model can apply them, and obtain citation ids for the discovered and learned ones.

**Used by.** Main Agent.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `ids` | list of strings | yes (1 to `knowledge.read_max_entries`, Proposed 10) | Knowledge entry ids (prefix `kn_`) from `knowledge_index`. Product skills are entries too: a skill's name is its entry `title`, and its `kn_` id is what is passed here. |

**Outputs (`result`).**

| Field | Type | Meaning |
|---|---|---|
| `entries` | list | Each element: `id`, `layer`, `kind`, `title`, `description`, `body`, `state`, `confidence` (`high`, `medium`, or `low`, as for `knowledge_index`), `confirmation`, `uncertain_reason` (when `state` is `uncertain`: why), `merged_into` (when `state` is `retired` and the entry was merged into another: the id of that entry), `usage_note`, `provenance` (summary: taught in which conversation and when, or derived from which configuration items; teacher identity is shown as a display name only when the reader is allowed to see it, see [Knowledge and learning](knowledge-and-learning.md)), `config_dependencies` (element names with their recorded modification stamps), `open_contradictions` (count), `updated_at`. |
| `missing` | list of strings | Ids that do not exist or belong to another user's preferences. Retired entries are not missing: they are returned with `state` `retired` and, where one exists, `merged_into`. |

`usage_note` tells the model how it must use the entry:

| Situation | `usage_note` text intent |
|---|---|
| `confirmed`, independently confirmed or discovered | Use normally and cite. |
| `confirmed` but `unconfirmed` (only the teacher has vouched for it) | Use, cite, and state it to the user as an explicit assumption. |
| `uncertain` | Usable, but verify first: re-check against the schema, or ask the user to confirm. |
| `contested` | Do not pick a meaning; ask the user which meaning they intend, listing the competing meanings in the body. |
| `retired` | Do not apply it; if `merged_into` names an entry, read that entry instead. |

**Tool-specific errors.** `INVALID_INPUT`. Missing ids are reported in `missing`, not as an error.

**Limits.** Up to `knowledge.read_max_entries` entries per call (Proposed 10); total body text capped at `tools.max_result_chars` (60,000 characters), with later entries omitted and listed in `missing` with reason `size`. **Proposed.**

**Citations.** One `knowledge` citation per returned discovered or learned entry, recording the state, confirmation, and confidence at the time of reading; no citation is registered for product skills or for the user's own preferences. When an answer cites a learned entry and the user then corrects the answer, the Main Agent reports the correction and the Knowledge Service marks that entry uncertain; when a query built from a cited learned entry fails static analysis, the Tool Gateway reports it and the Knowledge Service marks the entry uncertain. All state transitions are performed by the Knowledge Service (see [Knowledge and learning](knowledge-and-learning.md)).

### `aml_query`

**Purpose.** Run one read-only AML query as the signed-in user and return rows directly to the model. Intended for simple lookups whose result is small enough to read, for example "who owns this part" or "list the open ECOs assigned to me".

**Used by.** Main Agent.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `aml` | string | yes | One AML request. It must use only allowed read actions (see [Query Analyzer](query-analyzer.md) rule QA-V02). |
| `purpose` | string | yes | One sentence on what the query is for. Stored in the audit record and shown in live progress. |
| `max_rows` | integer | no (default and maximum `main_agent.direct_result_max_rows`, Proposed 100) | Rows to return to the model. The AML itself must still carry a row limit or paging (rule QA-C02). |
| `fields` | list of strings | no | Property names to keep in the returned rows when the AML selects more than the model needs. |

**Outputs (`result`).**

| Field | Type | Meaning |
|---|---|---|
| `rows` | list of objects | One object per returned item, flattened: `item_type`, `aras_id`, `keyed_name`, selected properties, and nested relationship rows under `relationships`. Each row carries `_cite` (its citation id). Item-valued properties are shown as keyed name plus id. |
| `row_count` | integer | Rows returned to the model. |
| `total_rows_reported` | integer or null | The total Aras reported for the query when paging was used; null when unknown. |
| `truncated` | boolean | `true` when more rows existed than were returned. |
| `truncation_hint` | string | Present when truncated: suggests a narrower filter or `start_analysis`. |
| `analyzer` | object | `cost_class` (`light`, `moderate`, `heavy`) and the list of warnings (also in envelope `warnings`). |

**Tool-specific errors.** `ANALYZER_REJECTED` (with diagnostics), `NOT_READ_ONLY`, `ARAS_ERROR`, `ARAS_UNAVAILABLE`, `ARAS_AUTH_EXPIRED`, `PERMISSION_DENIED`, `THROTTLED`, `BUDGET_EXHAUSTED`, `TIMEOUT`, `RESULT_TOO_LARGE` (returned instead of rows when the result text alone exceeds 60,000 characters even at the row cap).

**Limits.** At most `main_agent.direct_result_max_rows` rows (Proposed 100) and `tools.max_result_chars` characters (Proposed 60,000) returned to the model; page size in the AML at most `gateway.max_page_size` (Proposed default 500); query timeout `throttle.query_timeout_seconds` (Proposed default 60); counts against the question budget.

**Citations.** Every returned item, including nested related items, is registered as a `record` citation with item type, Aras id, keyed name, a link that opens the item in Aras, and the tool call id. Items Aras returned but that were cut by `max_rows` are not registered.

### `file_read`

**Purpose.** Return the extracted text of one file stored in the Aras vault, or a range of it, after re-checking through Aras that the user can access the file. Intended for reading a few specific documents; many files go to an Analysis Worker.

**Used by.** Main Agent.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `file_id` | string | yes | Aras id of the File item. |
| `via_item` | object | no | The item through which the file was found: `item_type` and `aras_id`. Recorded in the citation so the user can open the parent item. |
| `range` | object | no | One of: `pages` (first and last page), `sheet` (name, optional cell range), `slides` (first and last), `section` (heading text), or `chars` (start and end offsets). Defaults to the beginning. |
| `find` | string | no | A plain phrase; when present, the result returns the passages containing it (up to 20, each with surrounding context) instead of a range. |

**Outputs (`result`).**

| Field | Type | Meaning |
|---|---|---|
| `file` | object | `aras_id`, `file_name`, `format`, `size_bytes`, `page_count` or equivalent, `char_count`. |
| `text` | string | The requested text, segment boundaries marked with their locator. |
| `passages` | list | When `find` was used: each passage with `locator`, `text`, and `_cite`. |
| `next_range` | object or null | The range to request next when the file continues. |
| `extraction` | object | `status` (`ok`, `partial`), `reason` when partial (`no_text_layer` or `truncated`), `notes` (for example "3 pages had no text layer"). |

**Tool-specific errors.** `PERMISSION_DENIED` (the access re-check failed), `NOT_FOUND`, `FILE_UNREADABLE` (with reason `no_text_layer`, `encrypted`, `corrupt`, `extraction_timeout`, or `extraction_failed`), `FILE_UNSUPPORTED_FORMAT`, `RESULT_TOO_LARGE` (reason `file_too_large`: file over `file_service.max_file_bytes`), `ARAS_UNAVAILABLE`, `ARAS_AUTH_EXPIRED`, `THROTTLED`, `BUDGET_EXHAUSTED`, `TIMEOUT`.

**Limits.** Up to `file_service.read_max_chars` characters of text per call (Proposed default 20,000). Files above `file_service.max_file_bytes` (Proposed default 100 MB) are not fetched. Counts one query and the fetched bytes against the question budget; a cached extraction counts only the access re-check.

**Citations.** One `file` citation per returned segment (page, sheet, slide, section, or range) or `find` passage, with file id, file name, locator, the parent item from `via_item` when given, and a link to open the file or its parent in Aras.

### `start_analysis`

**Purpose.** Hand a bounded piece of heavy work (aggregation, trends, process questions, many records, many files) to an Analysis Worker, which works in the conversation's Sandbox, and receive its report. Several calls in the same model step run in parallel for multi-part questions.

**Used by.** Main Agent.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `title` | string | yes | Short title shown in live progress, for example "Delay by affected component, last quarter". |
| `objective` | string | yes | What the worker must find out, stated as a question with a definition of done. |
| `definitions` | list of objects | no | Agreed or assumed definitions, each with `term`, `meaning`, `source` (`user`, `knowledge`, or `assumption`) and optional `cite`. |
| `scope` | object | yes | `item_types` (names), `time_window` (from, to, and which date property defines it), `filters` (plain-language constraints the worker must honour), and optional `seed_ids` (Aras ids to start from). |
| `schema_context` | list of strings | no | Item types and lists whose discovered schema should be included in the brief. The Main Agent's runtime attaches the `full` schema for these. |
| `knowledge_ids` | list of strings | no | Knowledge entries to include in the brief in full, with their citation ids and usage notes. |
| `prior_data` | list of strings | no | Sandbox paths of data already fetched in this conversation that the worker may reuse; the handler attaches their manifest entries. |
| `expected_output` | object | yes | `findings_shape` (for example "ranked table of top 5 components with counts"), `numbers` (list of the figures to compute), and `max_findings`. |
| `budget` | object | no | A share of the remaining question budget: `queries`, `rows`, `bytes`, `wall_seconds`. Defaults to an even split of the remainder among the workers started in the same step. |

**Outputs (`result`).** The worker's accepted report (`report`, with the fields defined for [`submit_report`](#submit_report) plus usage). The worker does not submit a citation list: the handler derives the report's citation list from the citation ids used in the report, and returns it as the envelope's `citations`. The result also carries:

| Field | Type | Meaning |
|---|---|---|
| `job_id` | string | Analysis job id, prefix `anj_`. |
| `status` | `completed`, `failed`, `cancelled`, or `budget_exhausted` | The analysis job's status; the report's own `status` (`complete`, `partial`, `failed`) says how complete the answer is. |
| `steps_used` | integer | Model steps the worker used. |
| `usage` | object | What the worker consumed in total, with the budget keys `queries`, `rows`, `bytes`, and `wall_seconds`. This is the worker's own total, distinct from the envelope's `usage`, which describes the `start_analysis` call itself. |

**Tool-specific errors.** `INVALID_INPUT` (missing objective or scope), `BUDGET_EXHAUSTED` (no budget left to give), `SANDBOX_UNAVAILABLE`, `INTERNAL`. Calls beyond `analysis.max_parallel_workers` in one turn are not refused: the extra jobs queue and start when a worker slot frees.

**Limits.** Up to `analysis.max_parallel_workers` workers in parallel per turn (Proposed 4), further jobs queued; each worker up to `analysis.worker_max_steps` model steps (Proposed 60); all workers together share the turn's question budget (all Proposed defaults, see [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md)).

**Citations.** The report's citations are ids the worker obtained from `aml_fetch` and `file_fetch` (and knowledge citations passed in the brief). The Main Agent may cite them directly. Report validation (see [`submit_report`](#submit_report)) guarantees that every citation id in an accepted report was issued in this conversation; any that survive the fix attempts are removed and recorded as limitations.

### `start_background_job`

**Purpose.** Propose long-running work that should not block the conversation, for example a full-text scan across the contents of many files when the question has no item anchor. The job runs only after the user confirms it in the Chat App.

**Used by.** Main Agent.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `kind` | `full_text_scan` | yes | Job kind. MVP offers only `full_text_scan` to the model; batched learning is started by the service, not the model. |
| `title` | string | yes | Shown to the user in the confirmation card. |
| `explanation` | string | yes | The purpose shown to the user in one sentence (for example "Find test reports that mention seal leakage"), what the job will do, and what the user will get. |
| `candidate_query` | string | yes | Read-only AML that enumerates the candidate files or the items that carry them, run as the user; it passes the Query Analyzer like any query and is paged. |
| `file_selection` | object | yes | How files are reached from the candidate items: `mode` (`files_themselves` or `via_relationship`) and, for `via_relationship`, the relationship path as discovered (for example a document-to-file relationship; names are examples to verify against the target installation). |
| `search_terms` | list of objects | yes (1 or more) | Each with `text` (a literal term or phrase, not a regular expression), `case_sensitive` (boolean, default false), and `whole_word` (boolean, default false). |
| `combine` | `any` or `all` | no (default `any`) | Whether a file matches on any term or only when all terms occur. |
| `proximity_chars` | integer | no | With `all`: the terms must occur within this many characters of each other. |
| `context_chars` | integer | no (default `jobs.scan_snippet_chars`) | Characters of surrounding text kept with each match. |
| `max_files` | integer | no | Upper bound on files scanned, never above `jobs.scan_max_files` (Proposed 5,000). |

**Outputs (`result`).**

| Field | Type | Meaning |
|---|---|---|
| `job_id` | string | Background job id, prefix `bgj_`. |
| `status` | `proposed` | The job is proposed; nothing has run against file contents yet. |
| `estimate` | object | `candidate_files` (count from the scoping query), `cached_files` (already extracted), `expected_duration` (range), `query_load` (approximate number of queries). |
| `confirmation_expires_at` | timestamp | After this (`jobs.proposal_ttl_hours`, Proposed 24) the proposal expires as unconfirmed. |

The tool returns immediately after the proposal is recorded. The user's decision and, later, the job's result arrive as new turns in the conversation (see [Background Jobs](background-jobs.md)); the model does not wait.

**Tool-specific errors.** `INVALID_INPUT`, `ANALYZER_REJECTED` (the scoping query failed analysis), `NOT_READ_ONLY`, `THROTTLED` (the user already has `jobs.max_proposed_per_user` proposals awaiting a decision), `JOB_NOT_CONFIRMED` (an identical proposal in this conversation was declined, expired, or already decided).

**Limits.** Up to `jobs.max_proposed_per_user` proposals awaiting a decision per user (Proposed 3); at most one running job per user and three system-wide, with further confirmed jobs waiting in `queued`; candidate set capped at `jobs.scan_max_files` (Proposed 5,000). All Proposed defaults, see [Background Jobs](background-jobs.md).

**Citations.** None at proposal time. When the job completes, its result message carries `file` citations for every matching passage and `record` citations for the parent items.

### `record_learning`

**Purpose.** Propose a learned knowledge entry during the conversation (live learning): a definition, a process meaning, a mapping from business vocabulary to configuration, or a working technique for this installation. Learned knowledge describes how the installation is used, never what its data contains. The tool carries only learning proposals. It is not used to report confirmations, objections, verifications, or analyzer failures for existing entries: the Learner evaluates the user's next message for confirmation or objection, and the Tool Gateway and the Knowledge Service mark entries uncertain (see [Knowledge and learning](knowledge-and-learning.md)).

**Used by.** Main Agent.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `signal` | `correction`, `retried_question`, `empty_result`, or `explicit_teaching` | yes | The signal that justifies learning. Calls without one are rejected. |
| `title` | string | yes | Short name, for example "Delayed ECO". |
| `description` | string | yes | One line, as it will appear in `knowledge_index`. |
| `body` | string | yes | The knowledge itself, in plain language, referring to configuration by name. Must not contain business data values (part numbers, names of people, quantities, dates of specific records). |
| `config_refs` | list of objects | yes (at least one) | Configuration elements the entry refers to, each with `element_kind` (item type, property, relationship type, list value, lifecycle state, workflow activity, and so on), `name`, and `parent` (for example the item type of a property). Resolved against the discovered schema. |
| `evidence_message_ids` | list of strings | no | Messages in the current conversation that carry the signal. |
| `scope` | `organisation` or `personal` | no (default `organisation`) | `personal` stores a user preference instead of shared knowledge. |
| `supersedes` | string | no | An existing entry id this proposal corrects. |

**Outputs (`result`).**

| Field | Type | Meaning |
|---|---|---|
| `entry_id` | string or null | The entry created, merged into, versioned, or on which a contradiction was recorded; null when rejected. |
| `action` | `created`, `merged`, `new_version`, `contradiction_recorded`, `preference_saved`, or `rejected` | What the Learner did, as defined in [Knowledge and learning](knowledge-and-learning.md). `merged`: an entry about the same term or configuration element existed and was updated. `new_version`: the teacher changed their own entry. `contradiction_recorded`: the proposal disagreed with an entry another user taught; the disagreement is recorded and the entry becomes contested. `rejected`: nothing was written. |
| `rejection_reason` | string or null | When rejected: `no_signal`, `contains_data_values`, `unresolved_references`, or `malformed`. |
| `diagnostics` | list of diagnostics | When rejected: for example `LRN-DATA-VALUE` (severity `reject`) pointing at the offending text, or unresolved configuration references with `suggestions`. |
| `state` | string | Resulting entry state. |
| `confirmation` | string | Resulting confirmation status. |
| `notes` | list of strings | For example "config ref 'Delay Reason' resolved to property delay_reason on item type Express ECO". |

**Tool-specific errors.** `INVALID_INPUT` (missing or malformed fields), `KNOWLEDGE_CONFLICT` (concurrent update of the same entry, after one retry by the Learner). A proposal the Learner refuses is not an error: it returns `action = rejected` with a reason, and the model may reformulate once. The data-value guard runs in the Learner (diagnostic `LRN-DATA-VALUE`): **Proposed**, it rejects bodies containing strings shaped like Aras ids, or exact substrings of the tool results cited as evidence (configuration names excepted); scheduled leak audits are deferred (see [Roadmap](../product/roadmap.md)).

**Limits.** Body up to 4,000 characters; up to 10 learning proposals per turn (Proposed).

**Citations.** None for the call itself. The new entry becomes citable through `knowledge_read` in later turns.

### `aml_fetch`

**Purpose.** Run read-only AML as the user and write the full result into the conversation's Sandbox as a file, so it can be processed with code. Unlike `aml_query`, no rows beyond a small `sample` are returned to the model; the rest is a summary. Handles paging automatically.

**Used by.** Analysis Worker.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `aml` | string | yes | One AML request using allowed read actions, with a `select` and a page size (rules QA-C01 and QA-C02). |
| `purpose` | string | yes | One sentence, recorded in audit and progress. |
| `name` | string | no (default: the tool call id) | File name stem, letters, digits, hyphens, and underscores only. The Tool Gateway writes the result to `data/aml/<name>.jsonl` (or `.csv`) in the Sandbox data area, which only the Gateway can write (see [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md#53-how-the-tool-gateway-writes-into-the-sandbox)). An existing name is rejected with `PATH_INVALID`. |
| `format` | `jsonl` or `csv` | no (default `jsonl`) | File format. `jsonl` keeps nested relationships; `csv` flattens to one row per top-level item with relationship data omitted. |
| `all_pages` | boolean | no (default `true`) | Fetch every page up to the limits; `false` fetches only the page named in the AML. |
| `max_rows` | integer | no | Stop after this many rows (never more than the remaining budget). |

**Outputs (`result`).**

| Field | Type | Meaning |
|---|---|---|
| `path` | string | Path written in the Sandbox, relative to the Sandbox root (`data/aml/<name>.jsonl` or `.csv`). |
| `row_count` | integer | Top-level rows written. |
| `nested_row_count` | integer | Related rows written. |
| `pages_fetched` | integer | Number of AML requests made. |
| `complete` | boolean | `false` when the fetch stopped on a limit before the last page. |
| `stop_reason` | string or null | `max_rows`, `budget`, `timeout`, or null. |
| `columns` | list | Field names and the Aras data type of each, as written. |
| `sample` | list of objects | Up to 3 rows, so the model can see the shape. |
| `citation_set_id` | string | Id of the citation set covering every record in the file (prefix `cst_`, see 3.5). |

Every written item, top-level or nested, carries a `_cite` field holding the citation id the Citation Registry issued for that item (an item already registered in the conversation keeps its existing id), plus `_type`, `_id`, and `_keyed_name`; the full file format is defined in [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md#53-how-the-tool-gateway-writes-into-the-sandbox). The Gateway also appends an entry for the file to `data/manifest.json`. Code in the Sandbox carries these ids through to computed results, so findings can cite the records they are based on.

**Tool-specific errors.** As for `aml_query`, plus `PATH_INVALID`, `SANDBOX_UNAVAILABLE`.

**Limits.** Draws on the job budget handed down by `start_analysis`; page size up to `gateway.max_page_size`; each page subject to `throttle.query_timeout_seconds`; total file size up to the Sandbox disk limit.

**Citations.** Every written item and related item is registered as a `record` citation. The citation summaries returned in the envelope are capped at 20 (the sample); the rest are in the `_cite` fields of the file, and the whole file is covered by its citation set.

### `file_fetch`

**Purpose.** Fetch the extracted text of many files into the Sandbox, after the access re-check for each file, so the worker can search and analyse them with code.

**Used by.** Analysis Worker.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `files` | list of objects | yes (1 to `file_service.fetch_max_files`, Proposed 50) | Each with `file_id` and optional `via_item` (`item_type`, `aras_id`). |

**Outputs (`result`).**

| Field | Type | Meaning |
|---|---|---|
| `written` | list | For each fetched file: `file_id`, `file_name`, `path` (always `data/files/<file id>.txt`), `segments_path` (`data/files/<file id>.segments.json`, listing each segment's kind, label, character offsets, no-text flag, and citation id), `segments`, `chars`, `status` (`read` or `partial`), and `reason` when partial (`no_text_layer` or `truncated`). |
| `unreadable` | list | Files that could not be read, each with `file_id`, `file_name`, `status` (`unreadable`, `too_large`, or `failed`), and `reason` (`no_text_layer`, `encrypted`, `corrupt`, `extraction_timeout`, `extraction_failed`, or `file_too_large`). |
| `unsupported` | list | Files in formats outside MVP, with `file_id`, `file_name`, `format`. |
| `denied` | list | Files the access re-check refused or Aras reported as not found, with `file_id` only (the name is not disclosed if Aras did not return it). |
| `manifest_updated` | boolean | True when entries for the written files were appended to `data/manifest.json`. |

**Tool-specific errors.** `INVALID_INPUT`, `PATH_INVALID`, `SANDBOX_UNAVAILABLE`, `BUDGET_EXHAUSTED`, `THROTTLED`, `ARAS_UNAVAILABLE`, `ARAS_AUTH_EXPIRED`. Per-file failures do not fail the call; they appear in `unreadable`, `unsupported`, and `denied`, which the worker must carry into its coverage.

**Limits.** Up to `file_service.fetch_max_files` files per call (Proposed 50); each file up to `file_service.max_file_bytes`; bytes count against the job budget.

**Citations.** Each segment is registered as a `file` citation with its locator, and its id is written into the segment map; the worker cites the segment citations that support a finding, or asks for a finer passage citation by segment citation id plus character range in `submit_report`, which the handler registers (see [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md#55-report-validation-and-citation-registration)).

### `python_run`

**Purpose.** Run Python code in the conversation's Sandbox to aggregate, join, rank, compute trends, mine process paths, or search fetched text.

**Used by.** Analysis Worker.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `code` | string | yes | Python source to execute. |
| `purpose` | string | yes | One sentence, recorded and shown as progress when long-running. |
| `timeout_seconds` | integer | no (default `sandbox.python_default_timeout_seconds`, Proposed 120; maximum 600, Proposed) | Execution time limit. |
| `session` | `persistent` or `fresh` | no (default `persistent`) | `persistent` keeps interpreter state between calls within the job; `fresh` starts a clean interpreter. **Proposed.** |

**Outputs (`result`).**

| Field | Type | Meaning |
|---|---|---|
| `stdout` | string | Standard output. |
| `stderr` | string | Standard error. stdout and stderr together are capped at `analysis.python_output_max_chars` (Proposed 20,000), with a truncation note. |
| `exit_status` | `ok`, `exception`, `timeout`, `killed_memory`, or `killed_disk` | How the run ended. |
| `files_changed` | list | Paths created or modified, with sizes. |
| `duration_ms` | integer | Run time. |

**Tool-specific errors.** `SANDBOX_UNAVAILABLE`; `TIMEOUT` (the run exceeded its time limit and was stopped); `SANDBOX_EXEC_FAILED` (with `stderr` in the message, for exceptions and for runs killed for memory or disk). A Python exception is returned as `SANDBOX_EXEC_FAILED` so the model sees it as a failure to fix.

**Limits.** CPU, memory, and disk as configured for the Sandbox (Proposed defaults 2 CPUs, 4 GB, 10 GB). No network. A fixed set of preinstalled analysis libraries; no package installation at run time (Proposed; see [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md)).

**Citations.** None directly. Code is expected to carry the `_cite` fields of input rows and segments into its outputs; the report then cites those ids.

### `sandbox_read`

**Purpose.** Read a file in the Sandbox, for example a computed result table or a section of fetched text, without writing code.

**Used by.** Analysis Worker.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `path` | string | yes | Path relative to the Sandbox root, inside `data/` (written only by the Tool Gateway, read-only to the worker) or the job's `work/<job id>/` area. |
| `offset` | integer | no (default 0) | Line (text) or row (jsonl, csv) to start at. |
| `limit` | integer | no (default 200, max 1,000) | Lines or rows to return. |

**Outputs (`result`).**

| Field | Type | Meaning |
|---|---|---|
| `content` | string | The requested lines or rows. |
| `total_lines` | integer | Lines or rows in the file. |
| `next_offset` | integer or null | Where to continue. |

**Tool-specific errors.** `PATH_INVALID`, `NOT_FOUND`, `SANDBOX_UNAVAILABLE`, `RESULT_TOO_LARGE` (when the requested range exceeds 20,000 characters; narrow the range).

**Limits.** Up to 20,000 characters per call.

**Citations.** None. Rows read this way still carry their `_cite` fields, which the worker may use in its report.

### `report_progress`

**Purpose.** Publish a short progress line to the user's live view while a worker runs, so long answers are visibly moving.

**Used by.** Analysis Worker.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `message` | string | yes | One plain-language line, at most `progress.max_text_chars` characters (Proposed 300), for example "Fetched 412 ECOs closed last quarter; joining affected items". Must not contain sensitive values beyond what the user could see in Aras; it is shown only to the requesting user. |
| `fraction` | number | no | Estimated completion between 0 and 1. |

**Outputs (`result`).** `accepted` (boolean); `throttled` (boolean, true when the line was coalesced because the worker published too often).

**Tool-specific errors.** `INVALID_INPUT` (empty or over-long message).

**Limits.** At most one line per second per worker is delivered (`progress.min_interval_ms`, see [Audit and progress](audit-and-progress.md#53-progress-events)); extra lines replace the pending one (Proposed).

**Citations.** None.

### `submit_report`

**Purpose.** End the worker's job and return its report to the Main Agent. A worker's job ends only through this tool, through the step limit, or through cancellation. The report is the job contract defined in [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md#32-job-contract-the-report); this section gives the agent-facing fields.

**Used by.** Analysis Worker.

**Inputs.**

| Field | Type | Required | Meaning |
|---|---|---|---|
| `status` | `complete`, `partial`, or `failed` | yes | `partial` when a budget, step limit, or deadline was reached or some data could not be read; `failed` with a reason in `limitations`. |
| `summary` | string | yes | A short plain-language answer to the objective, with `[cite:...]` markers. |
| `findings` | list of objects | yes (may be empty) | Each with `statement` (with citation markers), `support` (citation ids or citation set ids), `strength` (`counted`, `observed_in_text`, `association`, or `weak_signal`), and `numbers` (names of entries in `computed_numbers` the statement relies on). |
| `computed_numbers` | list of objects | yes (may be empty) | Each with `name`, `value`, `unit`, `method` (one sentence: population, filters, definitions applied), `source_paths` (Sandbox files it was computed from), and `basis` (a citation set id covering the records counted, plus up to `analysis.sample_citations_per_number` representative citation ids). |
| `tables` | list of objects | no | Small result tables (up to `analysis.report_max_table_rows` rows each) with `title`, `columns`, `rows`; each row carries a citation id or citation set id. |
| `citation_sets` | list of objects | no | Narrower sets to register, each with `label` and `members` (citation ids, or a Sandbox path plus the `_cite` field to read them from). The result returns the new set ids, which may then be used in `findings` and `computed_numbers` of a resubmitted report. |
| `passages` | list of objects | no | Passage citations to register, each with a segment `citation_id` and a `char_start` and `char_end` within the file's extracted text. |
| `assumptions` | list of objects | yes (may be empty) | Each with `assumption`, `reason`, `source` (`user`, `knowledge` with its citation id, `discovery_default`, or `worker_default`), and `disclose` (boolean: whether the Main Agent should state it in the answer). |
| `coverage` | object | yes | `records` (`in_scope` among those the user can access, `analysed`, `without_linked_evidence`, per item type), `files` (`requested`, `read`, `partial`, `unreadable` with reasons, `unsupported` with formats, `too_large`, `denied`), `queries` (`run`, `truncated`, `rejected_unrecovered`), and `notes`. |
| `limitations` | list of strings | no | What the analysis could not establish and why, including that associations are not causes. |
| `candidate_learnings` | list of objects | no | How-to knowledge found during the job (never data values); the Main Agent decides whether to call `record_learning`. |
| `artifacts` | list of strings | no | Sandbox paths of result files a follow-up job may reuse. |

Usage (queries, rows, bytes, model steps, wall time) is added by the handler, not reported by the model.

**Outputs (`result`).** `accepted` (boolean); `problems` (validation problems the worker must fix when not accepted: unknown citation ids, findings without support, numbers without source paths or basis, coverage inconsistent with the manifest, causal wording without a source stating the cause); `registered` (ids issued for requested `citation_sets` and `passages`); and `attempts_left`. After `analysis.report_fix_attempts` failed attempts the report is accepted with status `partial` and its problems recorded as limitations.

**Tool-specific errors.** `INVALID_INPUT` (a required field is missing or malformed).

**Limits.** Report up to 30,000 characters (**Proposed**); tables up to `analysis.report_max_table_rows` rows each.

**Citations.** The report may use only citation ids obtained by this worker's `aml_fetch` and `file_fetch` calls, passed in the brief, or registered through this tool. The causal-honesty rule applies: a pattern drawn from co-occurrence has strength `association` unless a cited source states the cause.

## 5. Behaviour

### 5.1 Common call handling

1. The agent loop receives a tool call from the model and validates the arguments against the tool's input table. Invalid arguments return `INVALID_INPUT` without side effects.
2. The loop attaches the session context: user, conversation, turn, and, for workers, the analysis job and Sandbox.
3. The call is recorded in [tool_calls](persistence.md#tool_calls) with its arguments and start time.
4. The owning module executes the call. Calls that reach Aras go through the [Tool Gateway](tool-gateway.md), which applies the Query Analyzer and the Throttle.
5. Citable sources are registered in the Citation Registry before the result is returned, so every id in the result already exists.
6. The result envelope is recorded against the tool call, an audit event is written where the call touched Aras (see [Audit and progress](audit-and-progress.md)), and the envelope is returned to the model.

### 5.2 Retry guidance

The model may retry a call after `ANALYZER_REJECTED`, `TIMEOUT`, `RESULT_TOO_LARGE`, `SANDBOX_EXEC_FAILED`, and `INVALID_INPUT`, correcting the input. The service counts consecutive rejections of the same intent: after `tools.consecutive_rejection_hint` (Proposed 3) consecutive rejected AML calls (`aml_query` or `aml_fetch`) in a row within a turn, the next rejection's message carries a hint that depends on the caller. To the Main Agent it says to consult the schema with `schema_describe`, or to tell the user what it could not express. An Analysis Worker has no `schema_describe` tool and cannot ask the user, so the hint tells it to re-read the schema extract in its brief and, if the query still cannot be expressed, to submit a `partial` report that names the failing query in `limitations`. **Proposed.**

### 5.3 Result size discipline

Tools that return data to the model (`aml_query`, `file_read`, `sandbox_read`, `python_run`) are capped so the model's context stays usable. The tools apply the caps themselves (the Tool Gateway for `aml_query`, the File Service for `file_read`); the Main Agent validates its answer against them, for example by not computing aggregates from a result marked `truncated`. When a lookup turns out to need more data than the caps allow, the expected reaction is `start_analysis`, not repeated paging through `aml_query`.

## 6. Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `tools.max_result_chars` | integer | 60,000 (Proposed default) | Character cap for any single result returned to the model. |
| `tools.python_run.max_timeout_seconds` | integer | 600 (Proposed default) | Maximum `timeout_seconds` a `python_run` call may request. |
| `tools.record_learning.max_per_turn` | integer | 10 (Proposed default) | Learning proposals per turn. |
| `tools.consecutive_rejection_hint` | integer | 3 (Proposed default) | Rejections before the caller-specific hint (section 5.2). |

Limits defined in other specifications are referenced by their keys and not redefined here: `main_agent.direct_result_max_rows` ([Main Agent](main-agent.md)); `analysis.max_parallel_workers`, `analysis.worker_max_steps`, `analysis.python_output_max_chars`, `analysis.report_max_table_rows`, `analysis.report_fix_attempts`, `analysis.sample_citations_per_number`, and `sandbox.python_default_timeout_seconds` ([Analysis Worker and Sandbox](analysis-worker-and-sandbox.md)); `file_service.read_max_chars`, `file_service.fetch_max_files`, and `file_service.max_file_bytes` ([File Service](file-service.md)); `knowledge.index_max_lines` and `knowledge.read_max_entries` ([Knowledge and learning](knowledge-and-learning.md)); `jobs.max_proposed_per_user`, `jobs.proposal_ttl_hours`, `jobs.scan_max_files`, and `jobs.scan_snippet_chars` ([Background Jobs](background-jobs.md)); `progress.max_text_chars` and `progress.min_interval_ms` ([Audit and progress](audit-and-progress.md)); throttle and gateway keys ([Tool Gateway](tool-gateway.md)).

## 7. Limits

The per-tool limits are listed in each entry. All limits that touch Aras also draw on the question budget for the turn (Proposed defaults: 200 queries, 50,000 rows, 200 MB fetched, 30 minutes wall time, including all workers), enforced by the [Tool Gateway](tool-gateway.md).

## 8. Failure modes and recovery

| Failure | Detection | Effect on user | Recovery |
|---|---|---|---|
| Model calls a tool not available to its agent | Loop checks the agent's tool list | None | `INVALID_INPUT` with the list of available tools. |
| Model invents a citation id | Citation validation in the Main Agent or `submit_report` | Statement shown with an "unverified" note if not fixed | Marker removed; model asked once to fix. |
| Tool result exceeds caps | Size check before returning | Lookup answered partially or handed to a worker | `RESULT_TOO_LARGE` or truncation flag with a hint. |
| Aras session expires mid-turn | `ARAS_AUTH_EXPIRED` from the gateway | The turn ends `failed` with reason `auth_expired`; after signing in again the user chooses "Continue" | A new turn resumes from the saved transcript (see [Main Agent](main-agent.md) and [Chat App](chat-app.md)). |
| Repeated analyzer rejections | Consecutive rejection counter | The answer may say what could not be queried | Caller-specific hint (section 5.2): the Main Agent consults the schema or asks the user; a worker re-reads its brief's schema extract or submits a partial report. |
| Sandbox lost during a job | `SANDBOX_UNAVAILABLE` | Worker reports partial coverage | Worker retries once in a new Sandbox, or submits a partial report. |

## 9. Security considerations

- No tool accepts credentials, user identifiers, server URLs, or SQL from the model. Identity always comes from the session.
- Every tool that touches Aras runs as the signed-in user, so Aras permissions apply unchanged. Nothing in the catalog lets a model see data the user could not see in Aras.
- Read-only is enforced in the gateway (rule QA-V02 and `NOT_READ_ONLY`), not by prompting.
- Worker tools operate only within the conversation's Sandbox; reads are confined to `data/` and the job's `work/<job id>/` area, and only the Tool Gateway writes `data/` (`PATH_INVALID`).
- `record_learning` refuses bodies that appear to contain data values; shared knowledge is meant to hold meaning, not data.
- Tool results are data, not instructions. Text inside Aras records and files can contain instructions written by anyone with edit rights in Aras; prompts tell the model to treat it as content only (see [Prompts](../prompts/README.md)).

## 10. Observability

- Every call: a [tool_calls](persistence.md#tool_calls) row with tool name, arguments (AML text included), result status, error code, duration, usage, and citation count.
- Calls that touch Aras: an audit event (see [Audit and progress](audit-and-progress.md)).
- Metrics: calls per tool, error rate per code, analyzer rejection rate, rejection-then-success rate (a measure of how well diagnostics help the model), average rows and bytes per call, citation validation failures.

## 11. MVP versus deferred

In MVP: the 14 tools above. Deferred (see [Roadmap](../product/roadmap.md)): exposing the tool layer to outside agents over MCP; any write tool (the product is read-only); export tools for charts and results; tools over sources outside Aras; vector search tools over files or knowledge.

## 12. Open questions

1. **Proposed:** Analysis Workers receive schema and knowledge through the job brief rather than through `schema_describe` and `knowledge_read`. Should workers get read-only access to those two tools for cases the Main Agent did not anticipate?
2. **Proposed:** the citation marker syntax `[cite:<id>]`. The prompt set must use the same syntax.
3. **Proposed:** all numeric caps in this catalog (rows, characters, files per call, timeouts, per-turn learning proposals). To be tuned once the system runs against a real installation.
4. **Proposed:** the Learner's lightweight data-value guard on `record_learning` bodies (`LRN-DATA-VALUE`: id-shaped strings and exact substrings of cited tool results) in MVP, ahead of the deferred scheduled audit.
5. **Proposed:** `python_run` keeps a persistent interpreter session per job by default.
6. **Proposed:** the consecutive-rejection hint after 3 consecutive rejected AML calls.
7. **Proposed:** `report_progress` coalescing at one line per second per worker, shared with all progress sources.
8. **Proposed:** extra `start_analysis` calls beyond the parallel-worker limit queue rather than fail.
9. **Proposed:** `file_read` and `file_fetch` register one citation per segment; finer passage citations are registered only through `submit_report`.
10. **Proposed:** `aml_fetch` lets the worker name its result file (`name`), defaulting to the tool call id, and offers CSV as an alternative to JSON Lines.
