# Scope

Aras Insight sorts every capability into three categories:

- **Core:** required for the MVP. The product is not fit to pilot without it.
- **Quality of life:** in the MVP design and expected in the first release, but the product still works if one of these slips.
- **Not supported:** out of the MVP. Each item is listed in the [roadmap](roadmap.md) with the reason and the trigger for revisiting it.

The three categories are decided. The assignment of individual capabilities to core or quality of life is a **Proposed** classification unless a decision states it; items that the decisions place in the MVP outright, such as throttling, saved sessions, and discovery, are core. Proposed assignments are marked below and listed under open questions.

## Core

### Users and access

- Business end users in every role: managers, executives, engineers, and the roles around them.
- Sign-in to the Chat App through Aras. Every data access runs through the Aras server as the signed-in user, with that user's permissions. No service account, no admin-granted organisation-wide view, no hybrid access model.
- Read-only. Nothing is ever written to Aras.

### Sources

- Aras and what it connects to, such as its vault server. Nothing else.
- Readable file formats: PDFs with a text layer, Word, Excel, PowerPoint, and plain text formats (txt, csv, html, xml).
- Extracted file text is stored keyed by file id (Aras files are immutable), and access is re-checked through Aras each time the text is used.

### Answering

- Plain-language questions answered directly for simple lookups.
- Analytics (aggregation, rankings, trends, durations, process questions) performed by Analysis Workers, which fetch data into a Sandbox and compute with Python. There is no separate aggregation engine.
- Several Analysis Workers in parallel for multi-part questions. *(Proposed as core: the decisions allow it but do not classify it.)*
- "Why" questions, with sources read in this order: long text fields on items, workflow sign-off comments, then linked files read with code.
- Questions with no starting item: search across item names, titles, descriptions, and file names through AML.
- Full-text search across file contents as a background job that the user confirms, batched and throttled, as that user, with a renewable credential scoped to the job where the Aras sign-in offers one (otherwise scans run only while the user's own session is live).
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
- Knowledge entry states (confirmed, uncertain, contested, retired) and the rules that move entries between them, including invalidation when the configuration they depend on changes.
- Learned knowledge describes how the installation works, never business data values. In the MVP this relies on a strong initial prompt.

### Operation

- Self-hosted on the customer's infrastructure.
- Three configurable model slots (main, worker, background) using OpenAI-compatible endpoints or native clients. No embedding model.
- One PostgreSQL database for sessions, transcripts, knowledge, extracted file text, job state, and audit and usage records.
- One hardened container per user conversation as the Sandbox, with no network, expiring with the conversation. A full virtual machine is available for customers that require it.
- Sessions saved in the database during inference; live progress for long answers; users can leave and come back.
- Audit and usage records of every query run on a user's behalf. *(Proposed as core: required for security review and for measuring value.)*

## Quality of life

- Statement of assumptions when the agent uses a learned entry that nobody other than its teacher has confirmed yet; silent use after one independent confirmation. *(Proposed as quality of life.)*
- Asking which meaning the user intends when a learned entry is contested. *(Proposed as quality of life.)*
- Per-user preferences, kept in a personal layer separate from shared knowledge. *(Proposed as quality of life.)*
- Live learning during a conversation, in addition to batched learning over transcripts after an inactivity threshold. *(Proposed as quality of life: batched learning alone satisfies the core requirement.)*
- Size estimates shown before a user confirms a background job. *(Proposed.)*

## Not supported

Each of these is deferred, not rejected. The [roadmap](roadmap.md) gives the reason and the revisit trigger for each.

| Area | Not supported in the MVP |
|---|---|
| Writing | Creating, editing, promoting, or deleting anything in Aras |
| Formats | OCR of scanned documents and images; email files (.msg, .eml); other formats beyond the list above |
| Engineering data | Reading or reasoning over CAD geometry |
| Performance | Caching of answers or query results |
| Data | A history of Aras data kept by the product; sources outside Aras (ERP, MES, supplier systems); several Aras instances in one deployment |
| Load | An analytics path that avoids the production Aras server (read replica, reporting copy, SQL path) |
| Knowledge | A deployment hook to refresh knowledge on configuration deployment; ageing out unused knowledge; a vector index of knowledge; scheduled checks that learned knowledge holds no data values |
| Interfaces | Teams and Slack integration; an MCP interface for outside agents |
| Outputs | Export of answers, tables, and charts; saved questions and dashboards; scheduled reports; suggested questions; an ROI and usage dashboard |
| Users | Features for Aras developers and administrators |
| Models | Fine-tuning models on customer data |

## Open questions

1. The classification of individual capabilities into core and quality of life, where marked **Proposed** above, needs confirmation. This includes assumption statements for unconfirmed learned knowledge and the clarifying question for contested knowledge: both are part of the decided safety design for shared learning ([decision record 0020](../decisions/0020-organisation-wide-learnings-with-disclosed-assumptions.md)), so what is open is only their category.
2. Whether audit and usage records are core (proposed) or quality of life.
