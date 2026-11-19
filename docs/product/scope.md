# Scope

Aras Insight sorts every capability into three categories:

- **Core:** required for the MVP. The product is not fit to pilot without it.
- **Quality of life:** in the MVP design and expected in the first release, but the product still works if one of these slips.
- **Not supported:** out of the MVP. Each item is listed in the [roadmap](roadmap.md) with the reason and the trigger for revisiting it.

The three categories are decided, and so is the assignment of each capability to them. Items that the decisions place in the MVP outright, such as throttling, saved sessions, and discovery, are core.

## Core

### Users and access

- Business end users in every role: managers, executives, engineers, and the roles around them.
- Sign-in to the Chat App only through the Aras installation's authorization server ([decision record 0033](../decisions/0033-sign-in-through-the-aras-authorization-server.md)). There is no login-name-and-password form, and an installation without an authorization server is not supported in the MVP. Every data access runs through the Aras server as the signed-in user, with that user's permissions. No service account, no admin-granted organisation-wide view, no hybrid access model.
- Read-only. Nothing is ever written to Aras.

### Sources

- Aras and what it connects to, such as its vault server. Nothing else.
- Readable file formats: PDFs with a text layer, Word (.docx), Excel (.xlsx), PowerPoint (.pptx), and plain text formats (txt, csv, html, xml). The older binary Office formats (.doc, .xls, .ppt) are not read and are reported as unreadable in coverage.
- Extracted file text is stored keyed by file id (Aras files are immutable), and access is re-checked through Aras each time the text is used.

### Answering

- Plain-language questions answered directly for simple lookups.
- Analytics (aggregation, rankings, trends, durations, process questions) performed by Analysis Workers, which fetch data into a Sandbox and compute with Python. There is no separate aggregation engine.
- Several Analysis Workers in parallel for multi-part questions.
- "Why" questions, with sources read in this order: long text fields on items, workflow sign-off comments, then linked files read with code.
- Questions with no starting item: search across item names, titles, descriptions, and file names through AML.
- Full-text search across file contents as a background job that the user confirms, batched and throttled, as that user, with a renewable credential scoped to the job where the Aras sign-in offers one (otherwise the scan runs on the user's session credential and pauses when that session ends).
- Process analysis: comparing discovered workflow and lifecycle definitions with the paths and timings of actual runs.
- The model is the semantic layer. There is no vector index of files and no reuse of an existing search index in the MVP.

### Trust

- Citations of three kinds: Aras record (type, id, link), file (with page or section), and knowledge entry (discovered or learned, with state and confidence).
- Coverage on every answer: records used out of those the user can access, items with no linked evidence, and files that could not be read.
- Association, not causation, as a standing rule in every answer.
- Asking one short question before expensive analysis when a definition is missing, offering a definition suggested from discovery. For cheap lookups the agent proceeds and states the assumption.

### Protection of the Aras server

- Static analysis of every generated AML query: validity against the discovered schema (unknown item types, properties, relationships, list or state values, wrong value types, ignored fields) and cost (select-all, no row limit or paging, deep nesting, broad wildcards). Diagnostics go back to the model for a retry.
- Throttling: per-user and system-wide limits on concurrent and total queries, row budgets per question, and timeouts.

### Knowledge

- Discovery of the Aras configuration as each user (item types, properties, relationships, lists, workflow maps, lifecycle maps, methods, forms), with cheap change detection through modification stamps.
- Shipped product skills: how to write AML, how to use the tools and files, and what knowledge to keep and where.
- Learned instance knowledge, recorded automatically without a mandatory approval step, only from conversations that carry a signal (a correction, a retried question, an empty result, or explicit teaching), merged into existing entries rather than appended.
- Live learning: the Main Agent records a lesson during a conversation, so a definition applies later in the same conversation and for other users at once, in addition to batched learning over transcripts after an inactivity threshold.
- Assumption statements: when the agent uses a learned entry that nobody other than its teacher has confirmed yet, it states the entry as an assumption; after one independent confirmation it uses the entry silently.
- Asking which meaning the user intends when a learned entry is contested.
- Per-user preferences, kept in a personal layer separate from shared knowledge. Preferences belong to one user only; role-level preferences are not part of the MVP.
- Knowledge entry states (confirmed, uncertain, contested, retired) and the rules that move entries between them, including invalidation when the configuration they depend on changes.
- Learned knowledge describes how the installation works, never business data values. In the MVP this relies on a strong initial prompt.

### Operation

- Self-hosted on the customer's infrastructure.
- Three configurable model slots (main, worker, background) using OpenAI-compatible endpoints. The MVP ships no native provider clients; one can be added when a customer's provider needs it. No embedding model.
- One PostgreSQL database for sessions, transcripts, knowledge, extracted file text, job state, and audit and usage records.
- One hardened container per user conversation as the Sandbox, with no network, expiring with the conversation. A full virtual machine is available for customers that require it.
- Sessions saved in the database during inference; live progress for long answers; users can leave and come back.
- Audit records of every query run on a user's behalf and usage records of model calls, required for security review and for measuring value. Operators read them through documented database views and an export operation; there are no administrator endpoints in the MVP ([decision record 0040](../decisions/0040-operator-reports-without-administrator-endpoints.md)).
- Deployment and operations tooling is for the customer's infrastructure staff and is not a feature for Aras administrators. In the MVP that tooling is the audit and usage views and export, and a one-time Aras compatibility check run with an ordinary user account. There is no manual discovery trigger (the check at conversation start covers it), and administrators cannot view or cancel other users' background jobs. The operator can pause the background job runner as a whole.

## Quality of life

- Size estimates shown before a user confirms a background job: the number of candidate files, how many already have stored text, the expected duration, and the expected query load. The estimate does not include a download volume.

## Not supported

Each of these is deferred, not rejected. The [roadmap](roadmap.md) gives the reason and the revisit trigger for each.

| Area | Not supported in the MVP |
|---|---|
| Writing | Creating, editing, promoting, or deleting anything in Aras |
| Formats | OCR of scanned documents and images; email files (.msg, .eml); legacy Office formats (.doc, .xls, .ppt); other formats beyond the list above |
| Engineering data | Reading or reasoning over CAD geometry |
| Performance | Caching of answers or query results |
| Data | A history of Aras data kept by the product; sources outside Aras (ERP, MES, supplier systems); several Aras instances in one deployment |
| Load | An analytics path that avoids the production Aras server (read replica, reporting copy, SQL path) |
| Knowledge | A deployment hook to refresh knowledge on configuration deployment; ageing out unused knowledge; a vector index of knowledge; scheduled checks that learned knowledge holds no data values |
| Interfaces | Teams and Slack integration; an MCP interface for outside agents |
| Outputs | Charts in answers (tables are shown as tables); export of answers, tables, and charts (the operators' audit and usage export is the only export); saved questions and dashboards; scheduled reports; suggested questions; an ROI and usage dashboard |
| Users | Features for Aras developers and administrators |
| Models | Fine-tuning models on customer data |
