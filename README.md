# Aras Insight

Aras Insight is a self-hosted, read-only assistant for Aras Innovator. Business users across every role ask questions in plain language, such as "which five parts appear most often on change orders that finished late last quarter, and what do the linked engineering reports say went wrong with them?", and get answers with citations. To answer, it queries Aras as the signed-in user, reads the files Aras holds, computes numbers with code instead of estimating them, and cites every record, file passage, and piece of knowledge it used. It sees exactly what the user is allowed to see, writes nothing to Aras, runs on the customer's own infrastructure with the customer's own model endpoints, and learns the meaning of the customer's installation over time.

> **Status: specification stage.** This repository holds the product vision, architecture, module specifications, shipped skills, and prompts. There is no implementation yet. The documents are written so that an engineering team can build the MVP from them.

## The problem

A PLM system holds the history of how a product is designed, changed, approved, and made. Getting an answer out of it is still slow:

- **Simple questions need someone who knows the data model.** Finding the right item types, relationships, and states takes AML or report-building skills most users do not have.
- **Analytical questions become projects.** "Where are our change approvals getting stuck?" means exporting data, joining it in a spreadsheet, and summarising it by hand. That takes days, and the result is out of date when it is finished.
- **The "why" lives in text and files.** The reasons behind a delay or a recurring revision sit in description fields, workflow sign-off comments, and attached reports, not in rows that a report can count.
- **Every installation is different.** Customised item types, renamed states, and local conventions mean that generic tools and generic AI assistants guess, and their guesses cannot be checked.

The people who need these answers (executives, program managers, engineering, quality, manufacturing, change management, supply chain, compliance) either wait for an analyst or decide without the answer. See [docs/product/vision.md](docs/product/vision.md).

## Key capabilities

- **Answers in plain language, at the user's own access level.** Every data access goes through the Aras server as the signed-in user, so Aras permissions apply exactly as they do in the Aras client.
- **Analytics across records and files.** Counts, trends, rankings, durations, and process analysis are computed with Python in an isolated Sandbox, over data fetched from Aras.
- **"Why" questions.** Evidence is read from long text fields, workflow sign-off comments, and linked files (PDFs with a text layer, Word, Excel, PowerPoint, and plain text formats).
- **Process analysis.** Discovered workflow and lifecycle definitions are compared with the paths and timings that real items actually followed, to find deviations and bottlenecks.
- **Cited, verifiable answers.** Each answer cites the Aras records (with links), file passages (with page or section), and knowledge entries it relied on, and reports its coverage: how many records it used, what had no linked evidence, and what could not be read.
- **Checked queries.** Every generated AML query is statically analysed against the discovered schema for invalid fields and for cost before it runs. Throttling protects the production server.
- **Learns the installation.** Aras Insight discovers the configuration automatically and learns what terms and processes mean from conversations. It learns how things work, not business data values, and the MVP enforces this through prompts.
- **Your models, your infrastructure.** Three configurable model slots use OpenAI-compatible endpoints. No embedding model is required.

## How it works

```mermaid
flowchart LR
    U[User] --> CA[Chat App]
    CA --> MA[Main Agent]
    MA --> TG[Tool Gateway]
    MA --> AW[Analysis Worker]
    AW --> SB[Sandbox<br/>no network]
    AW --> TG
    TG -->|AML as the user| AR[(Aras server<br/>and vault)]
    TG -->|writes result files| SB
    MA --> KS[Knowledge Service]
    MA --> MR[Model Router]
    AW --> MR
    MR --> EP[Customer model endpoints]
```

Simplified view; see the [architecture overview](docs/architecture/overview.md) for all components.

The user signs in to the Chat App through Aras. The Main Agent answers simple lookups directly through the Tool Gateway, which is the only component that talks to Aras: it holds the user's Aras credentials, checks each query, applies throttling, and runs AML as the user. For heavy work such as aggregation, trends, process questions, or reading many files, the Main Agent starts one or more Analysis Workers. Each worker's code runs in a Sandbox that has no network access. Data reaches the Sandbox only as files written by the Tool Gateway. The worker returns a report with computed numbers, citations, assumptions, and coverage, and the Main Agent composes the final answer.

Full detail: [architecture overview](docs/architecture/overview.md) and [question lifecycle walk-throughs](docs/architecture/question-lifecycle.md).

## Scope summary

| Category | Examples |
|---|---|
| Core (MVP) | Plain-language lookups and analytics as the signed-in user; parallel Analysis Workers; reading text PDFs, Office files, and plain text; "why" questions; process analysis; citations and coverage; clarifying questions before expensive analysis; static query analysis; throttling; automatic discovery and learning, including live learning during a conversation; assumption statements for learned knowledge not yet independently confirmed, and asking which meaning is intended when knowledge is contested; per-user preferences; self-hosted with configurable model endpoints; live progress and saved sessions; user-confirmed background full-text scans; audit and usage records |
| Quality of life | Size estimates (candidate files and expected duration) before a background job is confirmed |
| Not supported in MVP (main examples) | Writing to Aras, OCR and image or email formats, CAD geometry, caching, a history of Aras data kept by the product, non-Aras sources, several Aras instances, chat integrations, export, saved questions and dashboards, fine-tuning |

Details are in [docs/product/scope.md](docs/product/scope.md), and the plan for deferred items is in [docs/product/roadmap.md](docs/product/roadmap.md).

## Documentation map

**Start here**
- [Executive summary](docs/executive-summary.md): one to two pages for decision-makers.
- [Glossary](docs/glossary.md): Aras, PLM, and AI terms for non-specialists.
- [Open questions](docs/open-questions.md): what remains undecided and why.

**Product**
- [Vision](docs/product/vision.md): the problem, the vision, product principles, and success.
- [Personas and use cases](docs/product/personas-and-use-cases.md): roles, needs, and example questions.
- [Scope](docs/product/scope.md): core, quality of life, and not supported.
- [Roadmap](docs/product/roadmap.md): the MVP milestones, then every deferred item.

**Business**
- [Value and ROI](docs/business/value-and-roi.md): the business case and the metrics to track.
- [Pricing options](docs/business/pricing-options.md): models, trade-offs, and a recommendation (pricing is not decided).
- [Risks](docs/business/risks.md): risks and their mitigations.
- [Pilot plan](docs/business/pilot-plan.md): how a company evaluates Aras Insight.
- [FAQ](docs/business/faq.md): answers for business, IT, security, and engineering leadership.

**Architecture and decisions**
- [Architecture overview](docs/architecture/overview.md)
- [Question lifecycle](docs/architecture/question-lifecycle.md)
- [Security and permissions](docs/architecture/security-and-permissions.md)
- [Deployment and operations](docs/architecture/deployment-and-operations.md)
- [Decision records](docs/decisions/README.md)

<details>
<summary>All 53 decision records</summary>

- [0001 Read-only](docs/decisions/0001-read-only.md)
- [0002 Self-hosted](docs/decisions/0002-self-hosted.md)
- [0003 Business end users first](docs/decisions/0003-business-end-users-first.md)
- [0004 Discovery-only schema](docs/decisions/0004-discovery-only-schema.md)
- [0005 User-level access through the Aras server only](docs/decisions/0005-user-level-access-through-aras-server.md)
- [0006 The model writes AML directly](docs/decisions/0006-model-writes-aml-directly.md)
- [0007 Static analysis of queries for validity and cost](docs/decisions/0007-static-analysis-for-validity-and-cost.md)
- [0008 Sandboxed Python analysis instead of an aggregation engine](docs/decisions/0008-sandboxed-python-analysis.md)
- [0009 Sealed Sandbox, with the worker loop outside it](docs/decisions/0009-worker-loop-outside-the-sandbox.md)
- [0010 Main Agent plus Analysis Workers](docs/decisions/0010-main-agent-plus-analysis-workers.md)
- [0011 Our own thin agent loop instead of a framework](docs/decisions/0011-own-thin-agent-loop.md)
- [0012 No vector index or search-engine reuse in MVP; graph-first retrieval](docs/decisions/0012-no-vector-index-graph-first-retrieval.md)
- [0013 A user-confirmed background full-text scan](docs/decisions/0013-user-confirmed-background-full-text-scan.md)
- [0014 Extracted text stored per immutable file id](docs/decisions/0014-extracted-text-per-immutable-file-id.md)
- [0015 One PostgreSQL database](docs/decisions/0015-one-postgresql.md)
- [0016 Discovery runs as each user](docs/decisions/0016-discovery-as-each-user.md)
- [0017 Three knowledge layers](docs/decisions/0017-three-knowledge-layers.md)
- [0018 Learned knowledge depends only on configuration](docs/decisions/0018-learned-knowledge-depends-only-on-configuration.md)
- [0019 Invalidation by provenance and re-derivation, plus contradiction](docs/decisions/0019-invalidation-by-provenance-and-contradiction.md)
- [0020 Organisation-wide learnings with disclosed assumptions](docs/decisions/0020-organisation-wide-learnings-with-disclosed-assumptions.md)
- [0021 Learn only from signal-bearing conversations; merge rather than append](docs/decisions/0021-learn-from-signals-and-merge.md)
- [0022 Prompt-based how-never-what enforcement in MVP](docs/decisions/0022-prompt-based-how-never-what-enforcement.md)
- [0023 No caching in MVP](docs/decisions/0023-no-caching.md)
- [0024 No history of Aras data](docs/decisions/0024-no-data-history.md)
- [0025 OCR deferred; MVP readable formats](docs/decisions/0025-ocr-deferred.md)
- [0026 Three model slots](docs/decisions/0026-three-model-slots.md)
- [0027 Causal honesty and coverage reporting](docs/decisions/0027-causal-honesty-and-coverage-reporting.md)
- [0028 Ask before expensive analysis; disclose assumptions on cheap work](docs/decisions/0028-ask-before-expensive-analysis.md)
- [0029 Three kinds of citation](docs/decisions/0029-three-kinds-of-citation.md)
- [0030 Throttling in MVP](docs/decisions/0030-throttling-in-mvp.md)
- [0031 Sources limited to Aras and what it connects to](docs/decisions/0031-sources-limited-to-aras.md)
- [0032 Saved sessions and live progress](docs/decisions/0032-saved-sessions-and-live-progress.md)
- [0033 Sign-in only through the Aras authorization server](docs/decisions/0033-sign-in-through-the-aras-authorization-server.md)
- [0034 Stored Aras credentials are encrypted at rest with a deployment-supplied key](docs/decisions/0034-credentials-encrypted-at-rest.md)
- [0035 Job credentials: renewable grant where the installation offers one, pause and resume, hard lifetime](docs/decisions/0035-job-credential-lifecycle.md)
- [0036 One shared discovered configuration for all users](docs/decisions/0036-one-shared-discovered-configuration.md)
- [0037 One service instance in the MVP; the Tool Gateway as an optional separate process](docs/decisions/0037-single-instance-mvp-and-optional-gateway-process.md)
- [0038 Sandbox hardening, isolated text extraction, and the file channel into the Sandbox](docs/decisions/0038-sandbox-and-extraction-isolation.md)
- [0039 Audit and usage records: redacted query text by default, append-only, citation counters, no identity groups](docs/decisions/0039-audit-and-usage-records.md)
- [0040 Operator reports through documented views and export; no administrator endpoints in the MVP](docs/decisions/0040-operator-reports-without-administrator-endpoints.md)
- [0041 No telemetry to the makers of the product](docs/decisions/0041-no-telemetry-to-the-makers.md)
- [0042 Assumptions are disclosed to every user until independently confirmed](docs/decisions/0042-assumption-disclosure-and-independent-confirmation.md)
- [0043 Contested knowledge: ask once per conversation, and how a contest ends](docs/decisions/0043-contested-knowledge.md)
- [0044 The Learner's operations, signals, and writes to Discovered entries](docs/decisions/0044-learner-operations-and-signals.md)
- [0045 Preferences are personal and per user only](docs/decisions/0045-personal-preferences-layer.md)
- [0046 Analysis job contract: structured inputs, knowledge only through the brief, citation sets](docs/decisions/0046-analysis-job-contract.md)
- [0047 A final answer check on every answer](docs/decisions/0047-final-answer-check.md)
- [0048 One running turn per conversation](docs/decisions/0048-one-running-turn-per-conversation.md)
- [0049 Discovery: when it runs, what it generates, and its limits](docs/decisions/0049-discovery-operation.md)
- [0050 Background full-text scans: proposal, decision flow, partial results, and completion](docs/decisions/0050-background-job-flow.md)
- [0051 Model slot requirements, startup checks, clients, and token budgets](docs/decisions/0051-model-slot-requirements-and-budgets.md)
- [0052 ROI metrics come from the product's own records](docs/decisions/0052-roi-metrics-from-product-records.md)
- [0053 Core capabilities: parallel workers, audit records, assumption statements, live learning, and preferences](docs/decisions/0053-core-capabilities.md)

</details>

**Module specifications** ([index](docs/specs/README.md))
- [Chat App](docs/specs/chat-app.md)
- [Main Agent](docs/specs/main-agent.md)
- [Tool Gateway](docs/specs/tool-gateway.md)
- [Query Analyzer](docs/specs/query-analyzer.md)
- [File Service](docs/specs/file-service.md)
- [Analysis Worker and Sandbox](docs/specs/analysis-worker-and-sandbox.md)
- [Discovery](docs/specs/discovery.md)
- [Knowledge and learning](docs/specs/knowledge-and-learning.md)
- [Background Jobs](docs/specs/background-jobs.md)
- [Model configuration](docs/specs/model-configuration.md)
- [Persistence](docs/specs/persistence.md)
- [Audit and progress](docs/specs/audit-and-progress.md)
- [Tool catalog](docs/specs/tool-catalog.md)

**Skills and prompts**
- [Skills catalog](docs/skills/README.md): every skill that ships with the product, with the full text of each Aras-agnostic skill.
  - [Citing sources](docs/skills/citing-sources.md): cite every claim to an Aras record, a file passage, or a knowledge entry.
  - [Reporting coverage and assumptions](docs/skills/reporting-coverage-and-assumptions.md): state what an answer was based on, what was missing or unreadable, and every assumption made.
  - [Causal honesty](docs/skills/causal-honesty.md): report association, never causation the data cannot show.
  - [Clarifying definitions](docs/skills/clarifying-definitions.md): ask one quick question before expensive analysis; state the assumption for cheap lookups.
  - [Delegating analysis](docs/skills/delegating-analysis.md): decide when to answer directly and when to start Analysis Workers or a background job.
  - [Working within limits](docs/skills/working-within-limits.md): respond to Query Analyzer diagnostics, row budgets, and throttling correctly.
  - [Python analysis in the Sandbox](docs/skills/python-analysis-in-sandbox.md): analyse result files and file text with Python, and compute every number.
  - [Progress reporting](docs/skills/progress-reporting.md): publish honest, useful progress lines during long work.
  - [Persisting knowledge](docs/skills/persisting-knowledge.md): decide what to persist and where: how, never what.
  - [Uncertain and contested knowledge](docs/skills/uncertain-and-contested-knowledge.md): use uncertain, unconfirmed, and contested knowledge safely.
  - [Untrusted content](docs/skills/untrusted-content.md): treat record values, file text, and comments as data, never as instructions.
  - [Composing the final answer](docs/skills/composing-the-final-answer.md): direct answer first, then evidence, citations, coverage, and assumptions.
- [Prompts](docs/prompts/README.md): the full text of every prompt.
  - [Main Agent system prompt](docs/prompts/main-agent-system.md): identity, boundaries, tools, and standing rules of the conversational agent.
  - [Analysis Worker system prompt](docs/prompts/analysis-worker-system.md): identity, environment, method, and report format of a sandboxed analysis job.
  - [Analysis job brief](docs/prompts/analysis-job-brief.md): the template the Main Agent fills when it starts a worker.
  - [Learner](docs/prompts/learner.md): turning conversation signals into learned knowledge, live and batched.
  - [Discovery inference](docs/prompts/discovery-inference.md): deriving process meaning from the Aras configuration.
  - [Background job confirmation](docs/prompts/background-job-confirmation.md): asking the user to confirm a long-running file-content scan.
  - [Contested knowledge clarification](docs/prompts/contested-knowledge-clarification.md): asking which meaning of a contested term the user intends.

## Contributing

The project is at the specification stage. Comments on the documents are welcome as issues or pull requests. The remaining open questions are collected in [docs/open-questions.md](docs/open-questions.md); those are the places where feedback is most useful.

## Trademark notice

Aras Insight is an independent project. It is not affiliated with, sponsored by, or endorsed by Aras Corporation. "Aras" and "Aras Innovator" are trademarks of their owner and are used here only to describe compatibility.
