# Frequently asked questions

Answers for business leaders, IT, security, and engineering leadership evaluating Aras Insight. Each answer links to the document with the detail.

## Capabilities

### Is this just natural language translated into a "select everything" query?

No. Simple lookups do run as a single AML query, but analytical questions are handled by Analysis Workers. A worker fetches the specific records it needs (for example change orders in a period and their affected items) into an isolated Sandbox, then computes counts, rankings, trends, and durations with Python. Every query is checked before it runs, and unbounded select-all queries are flagged as too expensive. See [question lifecycle](../architecture/question-lifecycle.md).

### Can it answer analytical questions over time, such as which parts appear most often on change orders that finished late, and what the linked engineering reports say went wrong with them?

Yes. The agent first settles what "late" means (asking one short question with a suggested definition, since the analysis is expensive), computes the ranking with code, then reads the reports linked to the top parts and summarises the failure modes they describe, with citations. It reports association, not cause, and states its coverage. See [personas and use cases](../product/personas-and-use-cases.md).

### Without a vector index, how does it answer "why" questions when the answer is not in a metadata field?

The model itself is the semantic layer. For a "why" question the agent reads, in order: long text fields on the relevant items, workflow sign-off comments, and the files linked to them, which it fetches through Aras and searches with code. Narrowing through the Aras relationships first keeps the set of files small. Where there is no starting item, it searches names, titles, descriptions, and file names through AML, and can propose a full-text scan across file contents as a background job that runs only after the user confirms. The MVP does not build a vector index and does not reuse an existing search index. See [scope](../product/scope.md).

### Can it find patterns across structured data and files together?

Yes. For a question such as what keeps driving new revisions of an assembly, it fetches the revision history and the change and problem records linked to it, then the root-cause and analysis documents linked to those, and correlates them in the Sandbox. The answer cites both records and file passages.

### Does it search records and files at the same time?

It combines them in one answer. The agent queries records through AML and reads files through the File Service in the same investigation, and the Analysis Worker can hold both in its Sandbox and search them together with code.

### Which file formats can it read?

PDFs with a text layer, Word (.docx), Excel (.xlsx), PowerPoint (.pptx), and plain text formats (txt, csv, html, xml). Scanned documents, images, email files, and the older binary Office formats (.doc, .xls, .ppt) are not read in the MVP; when an answer would have depended on them, its coverage lists them as unreadable. OCR and further formats are on the [roadmap](../product/roadmap.md).

### Can it do process mining?

Yes, within Aras. It compares the discovered workflow and lifecycle definitions with the paths and timestamps of actual runs, finds skipped, repeated, or out-of-order steps, and computes waiting times per step to locate bottlenecks.

### Does it understand our customised installation?

It discovers the configuration automatically (item types, properties, relationships, lists, workflow and lifecycle maps, methods, forms) as each user, and learns what local terms and processes mean from conversations. No schema or data-model mapping is configured by hand per installation; deployment settings such as the Aras address, limits, and model endpoints still are. See [knowledge and learning](../specs/knowledge-and-learning.md).

### What does it learn, and can it learn something wrong?

It learns how the installation works (what a term means, which property holds what, how a process runs), never business data values. In the MVP this rule is enforced by the prompts and the shipped product skill, not by an automated audit; see [risks](risks.md). It learns only from conversations with a signal: a correction, a retried question, an empty result, or explicit teaching. Every learned entry has provenance. Until someone other than the teacher confirms it, the agent states it as an assumption. A correction on an answer that used it makes it uncertain; disagreement between users makes it contested, and the agent asks which meaning is intended. When the configuration changes, discovered entries that depend on it are re-derived or retired, and learned entries that depend on it are marked uncertain.

## Trust and accuracy

### How are hallucinations controlled?

By several layers. Every claim cites its source: an Aras record (type, id, and a link), a file (with page or section), or a knowledge entry (with its state and confidence). Citations come from a registry of what was actually retrieved during the question. Numbers are computed by code, never written by the model. Generated queries are checked against the real schema, and invalid or ignored fields are reported back to the model for a retry. Every answer states its coverage. And the product never claims causation the data cannot show.

### Can users verify an answer?

Yes. Record citations link to the item in Aras, and file citations name the page or section. Users can see exactly which records and passages each statement rests on.

### What happens when it is wrong?

The user says so in the conversation. The correction is a learning signal: any learned entry the answer relied on becomes uncertain and is verified before it is used again.

## Data, security, and access

### Can a user see anything they cannot already see in Aras?

No. Every data access runs through the Aras server as the signed-in user, with that user's permissions. There is no service account with wider access and no administrator-granted organisation-wide view. Stored file text is re-checked through Aras each time it is used. The one residual risk is a business data value slipping into shared learned knowledge, which the MVP guards against with prompts rather than an automated audit; see [risks](risks.md). See [security and permissions](../architecture/security-and-permissions.md).

### Does any data leave our premises?

Not by the product's design. It is self-hosted, and model calls go only to the endpoints the company configures. If those endpoints are internal or in a private cloud, no data leaves the company's control. If the company chooses an external model provider, the content sent to that model leaves accordingly; that is the company's choice. The product sends nothing to its makers: it has no automatic telemetry.

### Where is data stored?

In one PostgreSQL database on the company's infrastructure: sessions and transcripts, knowledge, extracted file text, job state, and audit and usage records. The product keeps no history of Aras business data. Conversations are treated as private.

### Does it write to Aras?

No. It is read-only. It never creates, edits, promotes, or deletes anything.

### What runs the analysis code, and is it isolated?

A Sandbox: one hardened container per user conversation, or a full virtual machine for customers that require it. It has no network access, never calls Aras, and never holds model or Aras credentials. Data reaches it only as files written by the Tool Gateway, and it expires with the conversation.

### How do long-running background jobs authenticate?

With a renewable credential for the requesting user, scoped to that job and dropped when the job ends, where the installation's sign-in issues renewable grants; otherwise the scan runs on the user's session credential and pauses when the session ends. No privileged account is involved.

### Is there an audit trail?

Yes. Every query run on a user's behalf is recorded with the user, the time, and the outcome, along with usage records for model calls. Operators read the records through documented database views and an export; there are no administrator endpoints in the MVP. See [audit and progress](../specs/audit-and-progress.md).

## Models

### Can we use our own model endpoints and switch models?

Yes. There are three configurable model slots: the main conversational agent, the Analysis Workers, and background work. Each points at an OpenAI-compatible endpoint; the MVP ships no native provider clients, and one can be added when a customer's provider needs it. Switching a model is a configuration change. No embedding model or vector database is required. See [model configuration](../specs/model-configuration.md).

### Which models does it need?

A reasonably competent general model that can write structured queries and call tools. The product does not depend on a particular vendor. A pilot measures answer quality on the company's chosen models.

### Is it fine-tuned on our data?

No. It adapts through discovery, learned knowledge, and shipped skills. Fine-tuning is not supported in the MVP.

## Production impact

### Do queries run against the production Aras server?

Yes, in the MVP: every query goes through the Aras server as the user, because that is the only way to apply Aras permissions and server-side read logic faithfully. A path that keeps heavy analytics off production is on the [roadmap](../product/roadmap.md), subject to keeping user-level access faithful.

### How does it avoid slowing down other users?

Queries are reads through the normal Aras API, the same as any client. Before each runs, the Query Analyzer estimates its cost and flags select-all queries, queries with no row limit or paging, deep nesting, and broad wildcards. The Throttle enforces per-user and system-wide limits on concurrent and total queries, row budgets per question, and timeouts. Full-text scans run only as confirmed background jobs, batched and throttled.

### Does it throttle expensive queries?

Yes. Throttling is part of the MVP, and the static cost analysis feeds it.

### Does it cache results of repeated questions?

Not in the MVP. A repeated question queries Aras again, which keeps results current and avoids any risk of cached results crossing permission boundaries. Extracted file text is stored per file id, so re-reading files is cheaper. A permission-aware cache is on the roadmap.

### What happens when Aras is upgraded or reconfigured?

Discovery detects configuration changes and regenerates the discovered layer. Discovered knowledge that depended on changed elements is re-derived or retired, and dependent learned knowledge is marked uncertain. Business item type, property, and state names are never assumed; they are discovered.

## Business

### How is it priced?

Pricing is not decided; the business makes that decision. The realistic options and their trade-offs, including the effect of the company paying for its own model usage, and a recommendation for the first customers are in [pricing options](pricing-options.md).

### How do we measure ROI?

The metrics (questions answered, active users, time to answer, estimated time saved, citation follow-through, correction rate, load on Aras, cost per answer, and others) are defined in [audit and progress](../specs/audit-and-progress.md), explained in [value and ROI](value-and-roi.md), and computed from the product's own records. Operators read them through documented database views and an export; a dashboard for them is deferred.

### What does it cost to run?

Infrastructure for the service, the Sandboxes, and the database, plus model usage on the company's own endpoints, which is the largest variable cost. See [value and ROI](value-and-roi.md).

### How would we evaluate it?

Through a pilot with defined phases, acceptance questions, and success and exit criteria. See the [pilot plan](pilot-plan.md).

### What is not included?

The main exclusions are writing to Aras, OCR, email and image files, CAD geometry, caching, a history of Aras data kept by the product, an analytics path that avoids the production server, sources outside Aras, several Aras instances, Teams and Slack, an MCP interface, export, saved questions and dashboards, scheduled reports, an ROI dashboard, and features for developers and administrators. The complete list is in the [roadmap](../product/roadmap.md), and the categories are in [scope](../product/scope.md).

### Who is it for?

Business end users in every role: executives, program and project managers, engineering managers, engineers, quality, manufacturing, change and configuration management, supply chain, compliance. Not Aras developers, in the MVP.
