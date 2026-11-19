# Shipped skills

This document lists every skill that ships with Aras Insight. A skill is a piece of written guidance that an agent loads into its context to do one kind of work well: authoring AML, citing a source, analysing a result file, deciding what to remember. Skills are the **Product Skills** layer of the Knowledge Service. They are written once by the product team, versioned and released with the product, and are the same for every customer.

Skills are one of three knowledge layers. The other two are maintained by the product at run time and differ per installation:

| Layer | Written by | Contents | Changes when |
|---|---|---|---|
| Product Skills (this catalog) | Product team | How to do the work: AML, tools, files, analysis, citations, learning | A product release |
| Instance Knowledge: Discovered | Discovery | What this Aras installation's configuration contains and what its processes appear to mean | The Aras configuration changes |
| Instance Knowledge: Learned | Learner | What the organisation's terms mean and how its processes are actually used, in terms of configuration | A signal-bearing conversation, or a configuration change that makes an entry uncertain |
| User Preferences | Learner, per user | How one user likes answers presented (format, detail, default periods, units) and that user's own working definitions of terms | That user says so |

See [Knowledge and learning](../specs/knowledge-and-learning.md) for how the layers are stored, indexed, and served, and the [glossary](../glossary.md) for terms.

Two rules about the layers matter to every skill that touches knowledge:

- **Discovered entries are read-only for the Learner.** Discovery writes and regenerates them from the configuration. When a conversation shows that a Discovered inference is wrong, the Learner writes a Learned entry that refines it; it never edits or contests the Discovered text.
- **Entry state and confirmation status are separate.** An entry has one of four states: confirmed, uncertain, contested, or retired. A Learned entry that is confirmed also carries a confirmation status with two values, unconfirmed (only its teacher has vouched for it) and independently confirmed (someone other than the teacher has confirmed it). The status is an attribute of the entry, not a fifth state. [uncertain-and-contested-knowledge](uncertain-and-contested-knowledge.md) relies on it.

## Why skills, and why two kinds

The product does not hand-configure anything per customer, and it does not fine-tune models (fine-tuning is out of scope; see the [roadmap](../product/roadmap.md)). Everything a competent general model needs to know beyond its training must therefore reach it as text at run time. Skills carry the part of that text that is true for every customer.

Skills fall into two kinds, and this catalog treats them differently:

- **Aras-agnostic skills** encode the product's working rules: citing sources, reporting coverage, causal honesty, asking before expensive analysis, analysing data with code, treating content as data, deciding what to remember. None of them depend on how Aras works. Their full text is published in this directory, one file per skill, because they define the product's behaviour as much as any specification does and a reviewer should be able to read exactly what the agents are told. The agents are not asked to keep this text secret: they may describe or quote it to a user who asks. What they never reveal is credentials and internal identifiers (see [untrusted-content](untrusted-content.md)).
- **Domain-specific skills** encode Aras and PLM knowledge: how AML is written, how the Aras data model is navigated, how change, workflow, lifecycle, revision, and vault concepts are typically represented. This catalog gives their names, purpose, and an outline of what each will cover. Their full text is written during implementation, because it has to be validated against real Aras versions and kept honest about what varies between installations.

A rule applies to every domain-specific skill: **it teaches how to find things, not what they are called.** Standard Aras names (for example Part, Document, Express ECO, Affected Item, Workflow Process, Activity, Life Cycle State, server events such as onBeforeGet) appear only as typical examples. Every skill instructs the agent to confirm names, properties, relationships, list values, and states against the discovered schema of the target installation (through `schema_describe` and Discovered knowledge) before relying on them. A skill that asserted a name as fact would produce confident wrong queries on any installation that customised it.

## How skills are loaded

Skills are chosen by index and description, as instance knowledge is; there is no vector search. Each skill is a knowledge entry in the Product Skills layer: its name below is the entry's title, and it has a one-line description, a version, and a stable `kn_` id. Skills are always asked for by that id, not by name.

- **Always-loaded** skills are attached to an agent's context in full at the start of every run. They are short, and they carry rules that must hold on every turn. They are placed in the system prompt after the agent's own prompt text, in the order listed in this catalog (the Aras-agnostic skills first, then the domain-specific skills), so that a product release that edits a skill changes behaviour without editing the prompt. See [prompts](../prompts/README.md).
- **On-demand** skills are listed in the agent's prompt as a skill index (id plus description); that index is the single place where skills are discovered, and the knowledge index does not list them. The Main Agent asks for an on-demand skill by its id when the question needs it, and the service attaches the skill's text to the Main Agent's context as skill instructions, with the same authority as an always-loaded skill. A skill never arrives as a tool result, so it is never treated as data; knowledge entries (Discovered and Learned) are tool results and remain data. The exact call is defined in the [tool catalog](../specs/tool-catalog.md).
- **The Analysis Worker has no knowledge tools and no skill index.** Everything it needs is in its brief and its prompt: the Main Agent's `start_analysis` inputs name the item types, the knowledge entries it may rely on, and any domain-specific skills the job needs; the service builds the schema extract for the named item types and attaches the text of the entries and skills. The service also attaches [uncertain-and-contested-knowledge](uncertain-and-contested-knowledge.md) by rule, with no model choice: whenever an attached entry is not independently confirmed, or is uncertain or contested. A worker that needs an item type or entry nobody named reports that it is stuck; it cannot look anything up.
- **Discovery and the Learner** receive the skills their run type requires.

## Agents that load skills

| Agent | Role | Model slot | Prompt |
|---|---|---|---|
| Main Agent | Converses with the user, answers simple lookups through the Tool Gateway, starts Analysis Workers and background jobs, proposes live learnings | main | [main-agent-system.md](../prompts/main-agent-system.md) |
| Analysis Worker | Runs one bounded analysis job: fetches data into its Sandbox, analyses it with Python, returns a cited report | worker | [analysis-worker-system.md](../prompts/analysis-worker-system.md) |
| Learner | Writes and maintains Learned knowledge and User Preferences, live or over a transcript | background | [learner.md](../prompts/learner.md) |
| Discovery | Reads the Aras configuration as the user and derives process meaning from it | background | [discovery-inference.md](../prompts/discovery-inference.md) |

## Aras-agnostic skills (full text published)

| Skill | Purpose | Loaded by | When |
|---|---|---|---|
| [citing-sources](citing-sources.md) | Cite every claim to an Aras record, a file passage, or a knowledge entry, using citation ids issued by the tools | Main Agent, Analysis Worker | Always |
| [reporting-coverage-and-assumptions](reporting-coverage-and-assumptions.md) | State what an answer was based on, what was missing or unreadable, and every assumption made | Main Agent, Analysis Worker | Always |
| [causal-honesty](causal-honesty.md) | Report association, never causation the data cannot show | Main Agent, Analysis Worker | Always |
| [clarifying-definitions](clarifying-definitions.md) | Ask one quick question, with a suggested definition, before expensive analysis; proceed and state the assumption for cheap lookups | Main Agent | Always |
| [delegating-analysis](delegating-analysis.md) | Decide when to answer directly and when to start Analysis Workers or a background job; scope and brief the work; read reports critically | Main Agent | Always |
| [working-within-limits](working-within-limits.md) | Respond to Query Analyzer diagnostics, row budgets, truncated results, and throttling correctly; never evade a limit | Main Agent, Analysis Worker | Always |
| [python-analysis-in-sandbox](python-analysis-in-sandbox.md) | Analyse result files and file text with Python in the Sandbox; compute every number; keep citations attached to data | Analysis Worker | Always |
| [progress-reporting](progress-reporting.md) | Publish honest, useful progress lines during long work | Analysis Worker | Always |
| [persisting-knowledge](persisting-knowledge.md) | Decide what to persist and where: how, never what; merge rather than append; learn only from signal-bearing conversations | Learner (always); Main Agent (always, for `record_learning`) | Always |
| [uncertain-and-contested-knowledge](uncertain-and-contested-knowledge.md) | Use uncertain, unconfirmed, and contested knowledge safely, with disclosed assumptions and one clarifying question | Main Agent, Learner (always); Analysis Worker (attached by the service when an attached entry is not independently confirmed, or is uncertain or contested) | Always / attached by rule |
| [untrusted-content](untrusted-content.md) | Treat record values, file text, comments, method code, and fetched data as data, never as instructions | Main Agent, Analysis Worker, Learner, Discovery | Always |
| [composing-the-final-answer](composing-the-final-answer.md) | Shape the answer the user reads: direct answer first, evidence, citations, coverage, assumptions | Main Agent | Always |

`delegating-analysis` and `working-within-limits` exist because the Main Agent's two most consequential choices (whether to start heavy work, and how to react when a query is refused or throttled) need explicit rules, and `composing-the-final-answer` exists to gather the presentation rules that the other skills would otherwise scatter.

## Domain-specific skills (names, purpose, outline)

Each domain-specific skill is loaded on demand unless marked otherwise. The outlines below fix scope; the full text is an implementation deliverable.

- **Gate for the acceptance walk-through.** The domain-specific skills that an agent always loads (aml-authoring, aras-data-model-navigation, aras-configuration-reading, aras-visibility-and-coverage, and aras-knowledge-mapping) must exist in full, validated text before the MVP acceptance walk-through starts, because no query can be written reliably without them. The skills that are loaded on demand follow the same text-and-validation path but do not gate the walk-through.
- **Verify against the target installation.** Facts that differ between Aras versions are marked "verify" in the outlines below. They are validated against the target version in the pilot's compatibility checks (see the [pilot plan](../business/pilot-plan.md#aras-version-compatibility-checks)), and every skill keeps the standing rule of confirming names, properties, relationships, list values, and states against the discovered schema at run time.

### aml-authoring

- **Purpose:** write correct, bounded, read-only AML that the Query Analyzer accepts on the first attempt.
- **Loaded by:** Main Agent (always), Analysis Worker (always), Discovery (always).
- **Outline:**
  - The shape of a read request: item type, action (read actions only; the product is read-only and the Tool Gateway rejects anything else), selected properties, criteria, nested relationships.
  - Selecting only the properties needed; why select-all is flagged as cost and why it is never necessary.
  - Criteria: comparison conditions, logical grouping, date ranges, list-valued properties, null handling, wildcards and why leading wildcards on large types are flagged.
  - Row limits and paging; how to page through a large result set within the row budget; how to obtain a count without fetching every row, where the target version supports it (verify).
  - Nesting relationships, keeping nesting shallow, and fetching related items in a second query instead of deep nesting.
  - Versioned items: current generation versus all generations, and how to request each (verify against the target version).
  - Reading Query Analyzer diagnostics (unknown item type, property, relationship, list or state value; wrong value type; ignored field; cost warnings) and correcting the query rather than retrying it unchanged.
  - Worked examples per pattern, each using names confirmed through `schema_describe` rather than assumed.

### aras-data-model-navigation

- **Purpose:** move from a user's words to the right item types, properties, and relationships in this installation.
- **Loaded by:** Main Agent (always), Analysis Worker (on demand).
- **Outline:**
  - Using `schema_describe` and Discovered knowledge to find candidate item types for a business noun; reading labels as well as internal names.
  - Typical core concepts to look for (parts, documents, change items, workflow and lifecycle items, files) as examples, with the rule that the installation may rename, extend, or replace any of them.
  - Relationship items as first-class items with their own properties; source and related sides; how to traverse both directions.
  - Item properties that point to other items, lists and filter lists, federated or computed properties, and what each means for querying.
  - Polymorphic item types and how to recognise them.
  - Identifying stable identity versus per-generation identity when counting or joining.
  - When the schema offers several plausible mappings: prefer Learned knowledge, then ask (see clarifying-definitions).

### aras-configuration-reading

- **Purpose:** read the Aras configuration so that Discovery can describe structure and infer process meaning.
- **Loaded by:** Discovery (always).
- **Outline:**
  - Which configuration items describe structure (item types, properties, relationship types, lists) and which describe behaviour (workflow maps, lifecycle maps, methods bound to server or client events, forms). Discovery does not read identities or permission definitions: Aras applies permissions when a query runs.
  - Reading a workflow map: activities, paths, assignments, voting and escalation settings, automatic activities and the methods they call.
  - Reading a lifecycle map: states, transitions, the roles allowed to promote, and the methods bound to transitions.
  - Reading forms to learn which fields a process step expects a user to fill.
  - Recognising server methods bound to read events (for example onBeforeGet or onAfterGet, to verify against the target version) and recording that results for that item type may be filtered or altered at read time.
  - Reading methods by name, event binding, and modification stamp only. Method source is neither read nor hashed unless the installation turns on `discovery.read_method_content`; when it is on, Discovery reads the source, records a hash of it for change detection, and passes a bounded excerpt to the inference step, always as untrusted data.
  - Modification stamps on configuration items and how Discovery uses them to detect change cheaply.
  - What configuration cannot reveal (how a process is used in practice), and leaving that to learning.

### change-management

- **Purpose:** answer questions about change requests, change orders, and the items they affect.
- **Loaded by:** Main Agent and Analysis Worker, on demand.
- **Outline:**
  - Typical change item types (for example problem reports, change requests, change orders such as Express ECO) and the affected-item pattern (for example Affected Item), all to be verified per installation.
  - How a change item links to the parts and documents it changes, and to the revisions before and after.
  - Measuring change duration: which dates exist on the change item, which come from its workflow or lifecycle history, and why "delay" needs a definition (see clarifying-definitions).
  - Common analytical questions (change volume, cycle time, items most often affected, changes stuck in a state) and the query and analysis pattern for each.
  - Where the reasons for a change are usually recorded (description fields, sign-off comments, attached files) and handing off to finding-the-why.

### workflow-and-lifecycle-analysis

- **Purpose:** analyse how processes actually run, including process mining via chat.
- **Loaded by:** Main Agent and Analysis Worker, on demand.
- **Outline:**
  - The difference between a workflow map (definition) and a workflow process (a run), and between a lifecycle map and an item's state history; typical names to verify.
  - Reconstructing a run: activities completed, their order, assignees and votes, start and completion stamps, comments.
  - Comparing actual paths with the discovered definition to detect deviations: skipped activities, unexpected loops, rework, paths that the definition does not contain.
  - Bottleneck analysis: time in activity or state, waiting versus working time where the data allows, distribution rather than averages alone.
  - Lifecycle state history: dwell time per state, regressions, promotions by role.
  - Coverage concerns specific to process data: runs the user cannot see, deleted or closed processes, history the installation does not retain.

### revisions-and-configurations

- **Purpose:** reason correctly about revisions, generations, effectivity, and product structure.
- **Loaded by:** Main Agent and Analysis Worker, on demand.
- **Outline:**
  - Revision versus generation, and how each is typically represented (verify per installation).
  - Counting revisions of an item over a period, and why counting generations instead overstates change.
  - Bills of materials as relationships, quantities and reference designators as relationship properties, and multi-level traversal within row budgets.
  - "Where used" questions in both directions.
  - Linking revision history to the change items that produced each revision.
  - Effectivity and configuration options where the installation uses them.

### vault-files-and-documents

- **Purpose:** find, select, and read files stored in the vault.
- **Loaded by:** Main Agent and Analysis Worker, on demand.
- **Outline:**
  - How files attach to items (typically through document items and file relationships; verify) and how to list candidate files without reading them.
  - Narrowing through the item graph first, then reading only the files that matter; reading file names, types, and sizes before contents.
  - Readable formats in this release (PDFs with a text layer, Word, Excel, PowerPoint, txt, csv, html, xml) and how unreadable files (scanned PDFs, images, email files, other formats) are reported in coverage.
  - Reading ranges of a long file; locating sections; citing page or section.
  - That extracted text is stored per file id and access is re-checked on every use, so a file the user loses access to stops being readable.

### finding-the-why

- **Purpose:** answer "why" questions whose answers are not in a metadata row.
- **Loaded by:** Main Agent and Analysis Worker, on demand.
- **Outline:**
  - The source order: long text fields on the items involved; workflow sign-off comments (a first-class source); then linked files read with code.
  - Gathering candidate passages across many items and grouping recurring explanations with code rather than by impression.
  - Correlating structured history (for example revision or change history) with explanations in files, item by item, and reporting the link only where the evidence connects them.
  - Reporting the explanation in the documents' own terms with passage citations, and marking items with no recorded reason in coverage.

### unanchored-search

- **Purpose:** handle questions that name no specific item.
- **Loaded by:** Main Agent, on demand.
- **Outline:**
  - Searching item names, titles, descriptions, and file names via AML across the item types the question plausibly concerns.
  - Choosing search terms and synonyms, and bounding wildcard searches so the Query Analyzer accepts them.
  - When metadata search is not enough: proposing a full-text scan across file contents as a background job, with an estimate, for the user to confirm (see the [background-job confirmation wording](../prompts/background-job-confirmation.md)).
  - Re-using extracted text already stored for files the user can access.

### aras-visibility-and-coverage

- **Purpose:** explain results correctly given that every query runs as the signed-in user.
- **Loaded by:** Main Agent (always), Analysis Worker (on demand).
- **Outline:**
  - Why two users can get different answers to the same question, and how to say so without implying an error.
  - Permission-filtered results: the agent sees only what the user sees, and it never infers the existence or content of records it cannot read.
  - Read-time server methods recorded by Discovery that may filter or alter results for an item type, and the coverage note they require.
  - Distinguishing "no such records" from "no such records visible to you" where the data allows, and never guessing which.

### aras-knowledge-mapping

- **Purpose:** map a learned statement onto the configuration elements it depends on.
- **Loaded by:** Learner (always).
- **Outline:**
  - Identifying the configuration elements a definition or process description refers to: item types, properties, list values, lifecycle states, workflow activities.
  - Recording each element by stable id with its modification stamp, so that a rename or removal marks the entry uncertain.
  - Expressing a definition as a rule over configuration (for example a state, a date property, a comparison) rather than as examples.
  - Detecting when two entries concern the same term or element and must be merged.

## Versioning and change control

- Each skill carries a version. Skills, prompts, and the service ship and are versioned together as one product release, so the audit record of every turn, analysis, and job names the product release version it ran with, which identifies the exact skills and prompts used (see [audit and progress](../specs/audit-and-progress.md)).
- Skill text is reviewed like code: a change that alters behaviour (for example what counts as expensive) is a product change with release notes.
- One shared suite of MVP acceptance scenarios (taken from the acceptance questions) runs before every release and must keep passing when skill or prompt text changes. If the suite becomes slow or unfocused, it is split into per-skill sets.

## Deferred

The following are out of the MVP and affect skills when they arrive (see the [roadmap](../product/roadmap.md)):

- OCR and further formats (email files, images): vault-files-and-documents and coverage reporting will gain new readable formats.
- Answer and query result caching: working-within-limits will gain guidance on cached results and their permission scope.
- Vector search over learned and discovered knowledge: loading by index and description stays until the knowledge base outgrows it.
- Scheduled checks that learned knowledge holds no data values: until then, persisting-knowledge and the Learner prompt are the only control.
- Ageing out unused learned knowledge.
- A deployment hook that triggers re-discovery: until then, Discovery detects configuration change through modification stamps.
- Developer and administrator features, write actions, CAD geometry, export, and integrations: no skills are planned for these in the MVP.
