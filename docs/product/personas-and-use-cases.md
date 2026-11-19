# Personas and use cases

Aras Insight serves business end users in every role that touches product data. It is not aimed at Aras developers or administrators in the MVP (see [roadmap](roadmap.md)); the audit and usage reports and the compatibility check are operations tooling for the customer's infrastructure staff, not features for Aras administrators. Every user sees only what their own Aras account can see, so the same question asked by two people can return different, equally correct answers.

Item type, relationship, and state names in the examples below (for example Part, Document, Express ECO, Problem Report, Affected Item, Workflow Process, Activity) are typical of standard Aras installations. They are illustrations only. Aras Insight discovers the actual names and structures of each installation, and the examples would be phrased in that installation's own vocabulary.

## Capabilities referenced

The use cases map to these capabilities. Each is described in the [scope](scope.md) and specified in the [module specifications](../specs/README.md).

| Id | Capability | Short description |
|---|---|---|
| C1 | Lookup | Find and describe specific items, their relationships, and their current state. |
| C2 | Analytics | Counts, rankings, trends, durations, and comparisons computed with code over fetched records. |
| C3 | Why | Explanations drawn from long text fields, workflow sign-off comments, and linked files. |
| C4 | Cross-source | Correlating structured records (revisions, changes, workflow history) with the text of linked files. |
| C5 | Process analysis | Comparing defined workflows and lifecycles with the paths and timings items actually followed. |
| C6 | Unanchored search | Questions with no starting item: searching names, titles, descriptions, and file names, and optionally a user-confirmed background full-text scan of file contents. |
| C7 | Citations and coverage | Every answer cites its sources and states what it covered and what it could not read. |
| C8 | Learning | Local terms and process meanings taught in conversation are remembered for everyone, with provenance. |

## Roles

### Executives and senior leadership

**Needs:** a fast, trustworthy picture of program health, change activity, and quality trends, without commissioning a report; confidence that the numbers are real.

| Example question | Capabilities |
|---|---|
| How many change orders were released last quarter compared with the quarter before, by product line? | C2, C7 |
| Which programs have the most open problem reports older than 60 days? | C2, C7 |
| What are the three most common reasons given for late change approvals this year? | C2, C3, C7 |

### Program and project managers

**Needs:** status across many items at once, early warning on what is stuck, and evidence for schedule conversations.

| Example question | Capabilities |
|---|---|
| Which change orders affecting my program are waiting on an approval step, and for how long? | C1, C2, C5 |
| How has the average time from change request to released change order changed over the last four quarters? | C2, C5 |
| List the open items that block the next milestone for this assembly, with owners. | C1, C7 |

### Engineering managers

**Needs:** where engineering time goes, which designs generate rework, and how review steps perform.

| Example question | Capabilities |
|---|---|
| Which five parts appear most often on change orders that finished late last quarter, and what do the linked engineering reports say went wrong with them? | C2, C3, C4, C7 |
| Which review activities have the longest median wait time this year? | C2, C5 |
| How many parts were revised more than three times in the last year, and in which assemblies? | C2 |

### Design engineers

**Needs:** history and rationale for a part or document, quickly, without searching by hand.

| Example question | Capabilities |
|---|---|
| Why has this part been revised so often? | C3, C4, C7 |
| What changed between revision B and revision D of this document, and which change orders drove it? | C1, C3 |
| Where else is this component used? | C1 |

### Quality engineers

**Needs:** recurring problems, their documented root causes, and links between problems and changes.

| Example question | Capabilities |
|---|---|
| Which problem reports from the last six months mention seal leakage, and which parts do they reference? | C3, C6, C7 |
| Do problem reports on this subassembly share a common documented root cause in their attached analysis reports? | C3, C4 |
| How many problem reports led to a change order, and how long did that take on average? | C2, C5 |

### Manufacturing and production engineering

**Needs:** what changed that affects the shop floor, and why.

| Example question | Capabilities |
|---|---|
| Which released changes in the last month affect parts used in this manufacturing assembly? | C1, C2 |
| What do the manufacturing notes on recent changes to this part say about tooling impact? | C3 |

### Change and configuration management

**Needs:** process health, deviations from the defined process, bottlenecks, and consistency of records.

| Example question | Capabilities |
|---|---|
| Do our document review runs follow the defined review workflow, and which steps take the longest? | C5, C2, C7 |
| How many change orders skipped or repeated a review step this year? | C5, C2 |
| Which change orders were released with no linked change request? | C2, C7 |

### Supply chain and procurement

**Needs:** exposure to changes and problems on purchased parts, as far as Aras records them. Data outside Aras (ERP, supplier systems) is out of scope in the MVP.

| Example question | Capabilities |
|---|---|
| Which purchased parts had changes in the last quarter, and which manufacturer parts are linked to them? | C1, C2 |
| Which problem reports mention a specific supplier in their description or attachments? | C3, C6 |

### Compliance, regulatory, and audit

**Needs:** evidence trails: who approved what, when, and on what basis.

| Example question | Capabilities |
|---|---|
| Show the approval history of this change order with each sign-off comment. | C1, C3, C7 |
| Which released documents in this category have no recorded review sign-off? | C2, C5, C7 |
| Which documents mention this standard number? | C6 |

### New employees and occasional users

**Needs:** orientation in the installation's vocabulary and structure.

| Example question | Capabilities |
|---|---|
| What does "pending verification" mean for a change order here, and what happens next? | C1, C8 |
| What is the difference between a change request and a change order in our process? | C5, C8 |

## Generic worked use cases

The four use cases below are representative of the hardest questions the MVP must handle. The [question lifecycle](../architecture/question-lifecycle.md) walks through how the system answers questions of this kind step by step.

### Analytical: ranking with explanation

> "Which five parts appear most often on change orders that finished late last quarter, and what do the linked engineering reports say went wrong with them?"

1. "Late" is not defined by the data. Because the analysis is expensive, the agent asks one short question first, offering a definition suggested from discovery (for example, released later than a target date property, or longer than a given number of days in a review state).
2. An Analysis Worker fetches the change orders for the period and their affected items into the Sandbox, applies the definition with code, and ranks the parts.
3. For the top five, it narrows to the linked reports through Aras relationships, fetches their text, and searches it with code for failure-mode passages.
4. The answer gives the ranking with counts, a summary of failure modes per part with file citations, and coverage: how many change orders were considered, how many affected items had no linked report, and which files were unreadable (for example scanned PDFs). It phrases the result as association, not cause.

### Why: answers outside metadata rows

> "Why was this change order rejected twice before it was approved?"

The agent reads, in order: the long text fields on the item, the workflow sign-off comments on its activities (a first-class source), and then the linked files. It answers with quotations and citations to each comment and passage, and says if no recorded reason was found.

### Cross-source: records plus files

> "What keeps driving new revisions of this assembly?"

The agent fetches the revision history of the assembly and the change orders and problem reports linked to each revision. It then fetches the root-cause and analysis reports linked to those items and searches them with code. The answer correlates the two: which revisions were driven by which changes, and which documented causes recur across them, with record and file citations. Where revisions have no linked explanation, coverage says so.

### Process mining

> "Does our design review process follow its defined workflow, and where do reviews wait the longest?"

The agent reads the discovered workflow definition (the defined activities and paths) and fetches the actual workflow history of items that ran through it: which activities ran, in what order, when they started and finished, and votes or paths taken. An Analysis Worker compares the actual paths with the definition and computes waiting times per activity. The answer reports the common deviations (skipped, repeated, or out-of-order steps) with counts, the activities with the longest waits, and example items for each, with citations.

### Unanchored search

> "Have we documented any issues with this coating supplier?"

There is no starting item. The agent searches item names, titles, descriptions, and file names through AML. If that is not enough, it proposes a full-text scan across file contents as a background job, with an estimate of how many files it covers and how long it is expected to take. The scan runs only after the user confirms, batched and throttled, as that user. The user can leave and come back for the result.

## Mapping roles to capabilities

| Role | C1 | C2 | C3 | C4 | C5 | C6 | C8 |
|---|---|---|---|---|---|---|---|
| Executives | | primary | yes | | | | |
| Program and project managers | yes | primary | | | yes | | |
| Engineering managers | | primary | yes | yes | yes | | |
| Design engineers | primary | | primary | yes | | | |
| Quality engineers | | yes | primary | primary | yes | yes | |
| Manufacturing | primary | yes | yes | | | | |
| Change and configuration management | | yes | | | primary | | |
| Supply chain | yes | yes | yes | | | yes | |
| Compliance and audit | primary | yes | yes | | yes | yes | |
| New employees | yes | | | | yes | | primary |

C7 (citations and coverage) applies to every answer for every role.
