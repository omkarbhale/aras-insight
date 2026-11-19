# Module specifications

This folder specifies every module of Aras Insight in enough detail to build it without a further design conversation. The specifications describe behaviour, interfaces, data, limits, and failure handling in prose and tables; they contain no code. They sit below the [architecture overview](../architecture/overview.md), which explains how the modules fit together, and the [decision records](../decisions/README.md), which explain why.

## How to read these documents

- **Decision-makers** can read section 1 (Purpose), section 11 (MVP versus deferred), and section 12 (Open questions) of each specification to see what will be built, what will not, and what still needs a choice.
- **Engineers** should start with the [architecture overview](../architecture/overview.md) and the [question lifecycle](../architecture/question-lifecycle.md), then read [tool-catalog.md](tool-catalog.md) and [persistence.md](persistence.md), which every other specification refers to.

Every module specification uses the same sections: purpose; responsibilities and non-responsibilities; interfaces; data owned; behaviour; configuration; limits; failure modes and recovery; security considerations; observability; MVP versus deferred; open questions.

## Conventions used throughout

- **Proposed** marks a choice the specification makes where the product decisions do not. Every proposed choice also appears in that document's open questions, so it can be accepted or changed deliberately.
- **Aras names are examples.** Everything about a customer's Aras installation is discovered at run time. Where a specification names standard Aras concepts (for example ItemType, Life Cycle Map, Workflow Process, Activity, Express ECO, Affected Item, the `maxRecords` and `select` AML attributes, or server events such as onBeforeGet), it does so as typical examples from standard Aras releases that must be verified against the target Aras version, never as guaranteed facts.
- **Technology neutrality.** The decided technology is: one PostgreSQL database, Python inside the Sandbox, one hardened container per conversation (a full VM optional), OpenAI-compatible model clients plus native clients, and our own thin agent loop. Any other language, framework, or product named in these documents is a proposal.
- **Shared shapes.** The tool result envelope, diagnostic shape, error codes, and citation marker are defined once in [tool-catalog.md](tool-catalog.md). Identifier prefixes and every table are defined once in [persistence.md](persistence.md).

## Index

| Specification | Component | What it covers |
|---|---|---|
| [chat-app.md](chat-app.md) | Chat App | Sign-in through Aras, conversation view, live progress, citations that open the source in Aras, background job confirmation, corrections and feedback, contested-knowledge clarification, returning to a long-running answer. |
| [main-agent.md](main-agent.md) | Main Agent | The conversational agent loop: context assembly, direct lookups, when and how to start Analysis Workers, clarifying questions, assumptions, coverage, citation validation, resumption. |
| [tool-gateway.md](tool-gateway.md) | Tool Gateway | The only component that talks to Aras: credentials, AML Executor, Query Analyzer, Throttle, File Service, Citation Registry, and writing results into the Sandbox. |
| [query-analyzer.md](query-analyzer.md) | Query Analyzer | Every validity and cost rule applied to AML before it runs, with severity, diagnostic, and expected model reaction. |
| [file-service.md](file-service.md) | File Service | Fetching vault files through Aras as the user, text extraction for MVP formats, the extracted-text store keyed by file id, access re-checks, locators. |
| [analysis-worker-and-sandbox.md](analysis-worker-and-sandbox.md) | Analysis Worker, Sandbox | The job contract with the Main Agent, Sandbox lifecycle and isolation, how data arrives in the Sandbox, the report format. |
| [discovery.md](discovery.md) | Discovery | Reading the Aras configuration as the user, change detection through modification stamps, regenerating the discovered layer, triggering knowledge invalidation. |
| [knowledge-and-learning.md](knowledge-and-learning.md) | Knowledge Service, Learner | Knowledge layers, the entry schema, the state machine, re-derivation, confirmation and contested handling, merging, signal detection, live and batched learning. |
| [background-jobs.md](background-jobs.md) | Background Jobs | User-confirmed full-text scans, batched learning, the job-scoped renewable credential. |
| [model-configuration.md](model-configuration.md) | Model Router | The three model slots, endpoint configuration, capability requirements, retries and failover, usage records. |
| [persistence.md](persistence.md) | Database | The complete logical schema of the one PostgreSQL database, by owning module, with retention. |
| [audit-and-progress.md](audit-and-progress.md) | Audit and progress | Audit events, the live progress stream, usage metrics, retention. |
| [tool-catalog.md](tool-catalog.md) | All agent tools | Every Main Agent and Analysis Worker tool: purpose, inputs, outputs, errors, limits, citations. |

## Module map

```mermaid
flowchart LR
    U[User browser] --> CA[Chat App]
    CA --> MA[Main Agent]
    MA --> MR[Model Router]
    MA --> TG[Tool Gateway]
    MA --> KS[Knowledge Service]
    MA --> AW[Analysis Worker]
    AW --> MR
    AW --> TG
    AW --> SB[Sandbox]
    TG -->|writes result files| SB
    TG --> AR[(Aras server)]
    TG --> FS[File Service]
    FS --> AR
    DI[Discovery] --> TG
    DI --> KS
    LE[Learner] --> KS
    LE --> MR
    BJ[Background Jobs] --> TG
    BJ --> LE
    MA --> DB[(PostgreSQL)]
    TG --> DB
    KS --> DB
    BJ --> DB
```

The Tool Gateway is the only path to Aras. The Sandbox has no network and holds no credentials; it receives data only as files written by the Tool Gateway.

## Related documents

- Product context: [vision](../product/vision.md), [personas and use cases](../product/personas-and-use-cases.md), [scope](../product/scope.md), [roadmap](../product/roadmap.md).
- Architecture: [overview](../architecture/overview.md), [question lifecycle](../architecture/question-lifecycle.md), [security and permissions](../architecture/security-and-permissions.md), [deployment and operations](../architecture/deployment-and-operations.md).
- Agent behaviour: [skills catalog](../skills/README.md), [prompts](../prompts/README.md).
- Terms: [glossary](../glossary.md).

## Open questions across the specifications

Each specification ends with its own open questions. The ones that cut across modules are collected here.

1. **Citation marker.** The inline marker `[cite:<id>]` (for citations `cit_` and citation sets `cst_`) and the tool names are shared with the [prompts](../prompts/README.md); the two must stay in step when either changes. See [tool-catalog.md](tool-catalog.md#35-citation-summaries).
2. **Citation sets (Proposed).** A computed number over many records cites one set of record citations rather than every record. Sets are a grouping, not a fourth citation kind. See [tool-catalog.md](tool-catalog.md#35-citation-summaries) and [persistence.md](persistence.md#citation_sets).
3. **Job-scoped renewable credential.** Background full-text scans need a credential for the requesting user that can be renewed while the user is away and dropped when the job ends. Whether the target Aras version's sign-in offers a renewable grant must be verified; if it does not, scans can only run while the user's own session is live. See [tool-gateway.md](tool-gateway.md) and [background-jobs.md](background-jobs.md).
4. **Read actions beyond `get`.** The read-only allowlist of AML actions must be confirmed per Aras version. See [query-analyzer.md](query-analyzer.md).
5. **Workers' access to schema and knowledge.** In MVP an Analysis Worker receives schema and knowledge only through its job brief, matching the tool list in the [tool catalog](tool-catalog.md). Whether to give workers read-only `schema_describe` and `knowledge_read` is open. See [analysis-worker-and-sandbox.md](analysis-worker-and-sandbox.md).
6. **Shared versus per-user discovered schema (Proposed: shared).** Every Aras user can read configuration, so one discovered copy serves the deployment. An installation that restricts configuration reads would need per-user filtering. See [discovery.md](discovery.md).
7. **Default numbers.** Every limit, budget, timeout, and retention period in these specifications is a proposed default to be tuned in a pilot (see the [pilot plan](../business/pilot-plan.md)); per-question file budgets and relevance ordering are, by decision, tuned once the system runs.
8. **Single-user deployments.** The rule that a teacher's word is enough in a single-user deployment is expressed as a configuration flag rather than a user-count threshold. See [knowledge-and-learning.md](knowledge-and-learning.md).
