# Question lifecycle

This document follows five representative questions from the moment a user types them to the moment they read the answer. Together they cover every path through the system: a direct lookup, a multi-step analysis, a "why" question over unstructured evidence, process analysis, and a question with nothing to anchor on that becomes a background job.

Each walk-through marks four things the product always does, so they can be traced in the diagrams:

- **[D] Definitions** - where the agent asks for, or assumes, the meaning of a term.
- **[A] Assumptions** - where an assumption is disclosed to the user.
- **[C] Coverage** - where the size and gaps of the evidence are measured and reported.
- **[R] Citations** - where citation ids are issued by the Citation Registry and where they surface in the answer.

The item type, relationship, and property names used below (for example a change record type such as Express ECO, an Affected Item relationship, a Workflow Process with Activities) are typical of a standard Aras installation and are used only for illustration. In a real deployment every name comes from Discovery and must be verified against the target Aras version and the customer's configuration. Component names follow the [architecture overview](overview.md); tool names follow the [tool catalog](../specs/tool-catalog.md).

## Common stages

Every question passes through the same stages, though simple ones skip some.

```mermaid
flowchart LR
    q["Question arrives"] --> k["Load knowledge<br/>(index, then entries)"]
    k --> d{"Terms defined?<br/>Cost known?"}
    d -->|"cheap, undefined"| assume["Proceed, disclose assumption [A]"]
    d -->|"expensive, undefined"| ask["Ask one question,<br/>offer a discovered definition [D]"]
    d -->|"defined"| plan
    assume --> plan["Plan reads"]
    ask --> plan
    plan --> direct{"Small bounded read?"}
    direct -->|"yes"| aml["aml_query / file_read<br/>through Tool Gateway [R]"]
    direct -->|"no"| work["start_analysis<br/>Analysis Workers in Sandbox [R]"]
    aml --> compose["Compose answer:<br/>claims + citations + assumptions + coverage [C]"]
    work --> compose
    compose --> learn{"Signal in conversation?"}
    learn -->|"yes"| learner["Learner updates Learned knowledge"]
    learn -->|"no"| done["Done"]
```

What happens inside the Tool Gateway on every read is the same regardless of which agent asked:

```mermaid
sequenceDiagram
    participant Agent as Main Agent or Analysis Worker
    participant QA as Query Analyzer
    participant TH as Throttle
    participant EX as AML Executor
    participant Aras as Aras server
    participant CR as Citation Registry
    participant SB as Sandbox

    Agent->>QA: AML query
    QA->>QA: validate against discovered schema, estimate cost
    alt invalid or too costly
        QA-->>Agent: diagnostics (unknown property, ignored field, no row limit, ...)
        Note over Agent: model fixes the query and retries
    else acceptable
        QA->>TH: admit (cost estimate, user, question budget)
        alt over budget or limit
            TH-->>Agent: refused or queued, with reason (feeds coverage)
        else admitted
            TH->>EX: run
            EX->>Aras: AML as the signed-in user (paged)
            Aras-->>EX: items the user may read
            EX->>CR: register returned records
            CR-->>EX: citation ids
            alt aml_query (Main Agent)
                EX-->>Agent: rows + citation ids
            else aml_fetch (Analysis Worker)
                EX->>SB: write result file
                EX-->>Agent: file path, row count, citation set id (every row carries _cite)
            end
        end
    end
```

Note that the user's Aras permissions are applied by Aras itself in the `Aras server` step. The Tool Gateway never filters results on its own authority, and never widens them.

## 1. Simple lookup answered by the Main Agent alone

> "Which change requests assigned to me are still open?"

This needs one bounded query. No Analysis Worker, no Sandbox.

```mermaid
sequenceDiagram
    actor U as User
    participant App as Chat App
    participant MA as Main Agent
    participant KS as Knowledge Service
    participant GW as Tool Gateway
    participant Aras as Aras server

    U->>App: question
    App->>MA: message (session saved)
    MA->>KS: knowledge_index("open change requests")
    KS-->>MA: entries: discovered item type for change requests,<br/>learned "open" = states other than the closed/cancelled states (state confirmed, not yet confirmed by anyone but its teacher)
    MA->>KS: knowledge_read(ids)
    KS-->>MA: full entries [R: knowledge citation ids]
    Note over MA: "assigned to me" maps cleanly to a discovered property.<br/>Cheap query: proceed. [D]
    MA->>GW: aml_query(get change requests where assignee = me and state not in closed states, with row limit)
    GW->>Aras: AML as user
    Aras-->>GW: 7 items
    GW-->>MA: rows + citation ids [R]
    MA-->>App: answer: 7 items, each linked to its Aras record [R]<br/>"Open is taken to mean any state except Closed or Cancelled" (learned, state confirmed, not yet confirmed by anyone but its teacher) [A]<br/>"Based on 7 change requests assigned to you that you can access" [C]
    App-->>U: answer with clickable citations
```

Points to note:

- **[D] / [A]** If "open" had no learned meaning, the Main Agent would not stop to ask: the query is cheap. It would pick the states Discovery shows as non-terminal in the lifecycle map, run the query, and say so: "I took 'open' to mean any state except Closed and Cancelled; tell me if you mean something else." That sentence, if the user then corrects it, is a learning signal.
- **[R]** Every row in the answer carries a citation id that resolves to item type, id, and a link that opens the record in Aras. The learned definition of "open" is itself cited, with its state and confidence.
- **[C]** Coverage for a lookup is short: the count, and that it is limited to what the user can access. If the Throttle had stopped paging at a row budget, coverage would say "first N shown; more exist" instead of implying completeness.

## 2. Top-N analysis over change records with failure modes from linked reports

> "Which five parts appear most often on change orders that finished late last quarter, and what do the linked engineering reports say went wrong with them?"

This is the canonical heavy question: it needs a definition ("late"), aggregation over many records, traversal to affected items, and reading of linked documents.

### Stage 1: definition first, because the work is expensive

```mermaid
sequenceDiagram
    actor U as User
    participant App as Chat App
    participant MA as Main Agent
    participant KS as Knowledge Service

    U->>App: question
    App->>MA: message
    MA->>KS: knowledge_index("late change orders")
    KS-->>MA: discovered: change order item type, its affected-item relationship,<br/>its workflow map and lifecycle map, document relationships,<br/>no learned entry for "late"
    MA->>KS: schema_describe(change order type, workflow, lifecycle)
    KS-->>MA: properties incl. a target/need date and state-change timestamps
    Note over MA: Undefined term + expensive analysis: ask once. [D]
    MA-->>App: "How should I decide that a change order finished late? Suggested from your configuration:<br/>(a) released after its target date, or<br/>(b) total time in workflow above N days.<br/>I'll use (a) unless you say otherwise."
    App-->>U: question with suggested definition
    U->>App: "(a)"
    App->>MA: answer
    MA->>MA: record_learning(proposed: "late" for change orders = release later than target date, maps to: target date property, released state)
```

Property, state, and relationship names in the diagrams of these walk-throughs (a target date, a released state, affected items, generations of a part, votes, activity assignments) are illustrative; a real installation supplies its own through Discovery, to be verified against the target Aras version.

The suggested definitions are built from what Discovery found (a date property that looks like a target, a released state in the lifecycle map). The agent does not invent fields. The user's choice is a learning signal of the "explicit teaching" kind; the resulting Learned entry is organisation-wide immediately and will be stated as an assumption to other users until someone else confirms it ([ADR 0020](../decisions/0020-organisation-wide-learnings-with-disclosed-assumptions.md)). A learned entry has a state (confirmed, uncertain, contested, or retired) and, separately, whether anyone other than its teacher has confirmed it; until someone has, the agent states it as an assumption.

### Stage 2: bounded jobs to Analysis Workers

```mermaid
sequenceDiagram
    participant MA as Main Agent
    participant W1 as Analysis Worker
    participant GW as Tool Gateway
    participant SB as Sandbox
    participant FS as File Service
    participant App as Chat App

    MA->>W1: start_analysis(job: change orders released last quarter,<br/>late = released after target date, rank affected parts by count of late orders,<br/>for top 5, read linked engineering reports and extract what went wrong, return citations, assumptions, coverage)
    W1->>App: report_progress("Fetching change orders released last quarter")
    W1->>GW: aml_fetch(change orders, released in date range, select needed properties, paged)
    GW->>SB: data/aml/change_orders.jsonl (every row carries _cite, plus one citation set for the file)
    W1->>SB: python_run(compute days late, filter late orders)
    SB-->>W1: 212 released, 64 late [C]
    W1->>GW: aml_fetch(affected items of the 64 late orders)
    GW->>SB: data/aml/affected_items.jsonl
    W1->>SB: python_run(count late orders per part, rank, save the ranking under work/{job id}/)
    SB-->>W1: top 5 parts with counts
    W1->>App: report_progress("Reading engineering reports for 5 parts")
    W1->>GW: aml_fetch(documents related to the top 5 parts or the late orders, file references)
    GW->>SB: data/aml/related_documents.jsonl
    W1->>GW: file_fetch(38 file ids)
    GW->>FS: for each file: re-check access as user, stored text or extract
    FS-->>GW: 31 extracted, 5 scanned (no text layer), 2 access denied
    GW->>SB: data/files/{file id}.txt for 31 files [R: file citations per page or section]
    W1->>SB: python_run(locate failure-mode passages: section headings, keyword windows)
    SB-->>W1: candidate passages with file id + page
    W1->>SB: sandbox_read(passages)
    Note over W1: model reads the passages and groups failure modes
    W1->>MA: submit_report(findings, computed numbers, citations,<br/>assumptions: late definition, date range = calendar quarter,<br/>coverage: 212 orders, 64 late, 7 of top-5 linked reports unreadable)
```

### Stage 3: answer

The Main Agent composes the answer from the report. It does not recompute numbers; it uses the ones the worker computed. The answer contains:

- The ranked list of five parts, each with the count of late change orders it appeared on, each linked to its record **[R]**, and each count traceable to the worker's computation.
- For each part, the failure mode(s) the reports describe, with a citation to file and page **[R]**.
- **Causal honesty:** "These parts appear most often on late change orders." Not "these parts made the change orders late."
- **[A]** "Late means released after its target date (defined by you just now). Last quarter means the previous calendar quarter."
- **[C]** "Based on 212 change orders released last quarter that you can access, of which 64 were late. 5 linked reports are scanned images and could not be read; 2 linked reports are not accessible to you. 3 of the late orders had no linked reports."

Failure modes drawn from the run:

| What goes wrong | What the system does | Where the user sees it |
| --- | --- | --- |
| The model writes AML with a property that does not exist on the change order type. | Query Analyzer returns a diagnostic naming the property and the closest discovered alternatives; the worker retries. | Not shown unless retries fail; then reported as a limitation. |
| The model writes a query with no row limit over all change orders. | Query Analyzer flags cost; Throttle requires paging. | Not shown. |
| The question budget for rows is reached before all affected items are fetched. | Throttle stops further fetches; the worker reports partial coverage. | Coverage: "Affected items fetched for 50 of 64 late orders (row budget reached)." |
| Scanned reports with no text layer. | File Service reports them unreadable (OCR deferred). | Coverage lists them by file name and counts them; an unreadable file has no text to cite. |
| A linked report the user cannot open. | Aras denies the download; File Service records denial. | Coverage counts them without naming their content. |
| A report contains text such as "ignore previous instructions". | Treated as data like any other text; it cannot change tools or permissions. | Not shown; at most quoted as content if relevant. |
| Two workers in parallel (for example one per quarter for a comparison). | Each gets its own bounded job; they share the conversation's Sandbox and draw on the turn's question budget (each worker has a share). | Progress lines from both. |

## 3. Why-question correlating revision history with root-cause documents and sign-off comments

> "Why has the pump housing assembly been revised so often?"

Here the evidence is mostly unstructured. The product guidance fixes the order in which "why" sources are tried: long text fields on items first, then workflow sign-off comments, then linked files read with code ([ADR 0012](../decisions/0012-no-vector-index-graph-first-retrieval.md)).

```mermaid
sequenceDiagram
    actor U as User
    participant MA as Main Agent
    participant W as Analysis Worker
    participant GW as Tool Gateway
    participant SB as Sandbox

    U->>MA: "Why has the pump housing assembly been revised so often?"
    MA->>GW: aml_query(find parts whose item number or name matches "pump housing assembly")
    GW-->>MA: 1 part [R]
    Note over MA: "So often" is undefined (what counts as recurring?) and the analysis is expensive:<br/>ask once. [D]
    MA-->>U: "By 'so often' do you mean the revisions of the last 18 months, looking for reasons that recur across them?<br/>I'll use that unless you say otherwise."
    U->>MA: "Yes."
    MA->>MA: record_learning(proposed: "revised so often" for parts = revisions within the last 18 months, maps to: the part item type's generation and revision properties)
    MA->>W: start_analysis(job: revision history of this part, the change records behind each revision,<br/>their long text fields, sign-off comments on their workflows, linked root-cause documents,<br/>correlate reasons across revisions)
    W->>GW: aml_fetch(all generations of the part: revision, dates, state)
    GW->>SB: data/aml/revisions.jsonl
    W->>GW: aml_fetch(change records listing any generation of this part as affected item,<br/>description and reason text fields)
    GW->>SB: data/aml/changes.jsonl
    W->>GW: aml_fetch(workflow process, activities and sign-off comments for those change records)
    GW->>SB: data/aml/signoffs.jsonl
    W->>GW: aml_fetch(documents related to those change records, with file references)
    GW->>SB: data/aml/docs.jsonl
    W->>GW: file_fetch(root-cause document file ids)
    GW->>SB: data/files/{file id}.txt for each file read
    W->>SB: python_run(timeline: revision -> change record -> stated reason -> sign-off comments -> RCA passages)
    SB-->>W: 9 revisions in 18 months, 6 change records cite a related tolerance issue
    W->>MA: submit_report(themes with supporting citations per revision,<br/>coverage: 9 revisions, 8 with a change record, 5 with an RCA document, 1 RCA unreadable)
    MA-->>U: answer
```

How the correlation works without a vector index or a search engine: the Aras item graph does the narrowing. The part leads to its revisions, the revisions lead to the change records that produced them, the change records lead to their workflow runs and their linked documents. Only the documents reached that way are read, so the set is small and every passage is tied to a specific revision. The model is the semantic layer: it reads the stated reasons, comments, and passages and groups them into themes.

The answer:

- Lays out a timeline of revisions, each with its change record and the reason given in that record **[R]**.
- Names recurring themes ("six of nine revisions reference a tolerance issue at the same interface"), each backed by citations to the change records, sign-off comments, and document passages that state it **[R]**.
- Stays associative: "the documented reasons point to", not "the root cause is", unless a cited document itself states a root cause, in which case the answer quotes and cites it.
- **[A]** "'So often' is taken to mean revisions in the last 18 months, as you confirmed; earlier history is available if you want it."
- **[C]** "9 revisions; 1 had no linked change record; 3 change records had no root-cause document; 1 document could not be read (scanned)."

## 4. Process-bottleneck question comparing workflow definitions with actual runs

> "Does our design review process follow its defined workflow, and where do reviews wait the longest?"

Process mining in this product means comparing the discovered workflow definition (the map) with the paths and timestamps of actual workflow and lifecycle runs. No separate process-mining engine is involved; it is Python over fetched run data.

```mermaid
sequenceDiagram
    actor U as User
    participant MA as Main Agent
    participant KS as Knowledge Service
    participant W as Analysis Worker
    participant GW as Tool Gateway
    participant SB as Sandbox

    U->>MA: question
    MA->>KS: knowledge_index("design review process")
    KS-->>MA: discovered workflow maps, learned: "design review process" = workflow map M (uncertain:<br/>its source map was renamed since it was learned)
    Note over MA: Uncertain entry: verify before use. [D]<br/>Expensive analysis: ask once.
    MA-->>U: "By 'design review process' do you mean workflow map M? And is 'its defined workflow' the<br/>standard review workflow, map S? I'll compare runs from the last 6 months unless you say otherwise."
    U->>MA: "Yes, both."
    Note over MA: The user's confirmation is evaluated by the product (the Learner), not judged by the Main Agent,<br/>and clears the uncertain state of the learned entry.
    MA->>W: start_analysis(job: definition of M and S (activities, paths, votes),<br/>runs of M in last 6 months with activity assignments, start and close times, paths taken,<br/>compare to definition and to S, find where reviews wait longest by time in activity)
    W->>GW: aml_fetch(workflow map definitions M and S)
    GW->>SB: data/aml/maps.jsonl
    W->>GW: aml_fetch(workflow processes instantiated from M in range, activities, assignments, timestamps, paths)
    GW->>SB: data/aml/runs.jsonl (paged)
    W->>SB: python_run(reconstruct each run's path, compare to the defined paths,<br/>compute time per activity, distribution, rework loops)
    SB-->>W: 140 runs, 23 took a path not in the definition, median and tail times per activity
    W->>MA: submit_report(deviations with example runs, bottleneck activities with computed durations,<br/>coverage: 140 runs, 12 still in progress, 4 with missing timestamps)
    MA-->>U: answer
```

The answer:

- States the comparison precisely: which steps of M are not in S, which runs left the defined paths and how often, and the activities with the longest time in activity, with the computed medians and tails **[R]** (every example run cited).
- Separates two different findings that are easy to conflate: deviation from the process definition (structural), and slowness within the definition (bottleneck).
- **[A]** "Time in an activity is measured from activation to closure; waiting for assignment is included."
- **[C]** "140 runs you can access; 12 still in progress are excluded from durations; 4 runs had missing timestamps."
- Causal honesty: "Activity A accounts for the largest share of elapsed time", not "A is why the process is slow".

## 5. A question with no item anchor that becomes a background full-text job

> "Have we ever had issues with corrosion on coastal installations?"

No part, document, or change record is named. The agent first tries the cheap anchors, and only proposes a scan of file contents if they are not enough.

```mermaid
sequenceDiagram
    actor U as User
    participant App as Chat App
    participant MA as Main Agent
    participant GW as Tool Gateway
    participant BJ as Background Jobs
    participant FS as File Service

    U->>App: question
    App->>MA: message
    MA->>GW: aml_query(search names, titles, descriptions across likely item types for "corros*" and "coastal")
    GW-->>MA: 3 problem reports, 1 document [R]
    MA->>GW: aml_query(search file names for related terms)
    GW-->>MA: 2 files [R]
    Note over MA: Metadata search found a little. File contents may hold more.<br/>A full-text scan is long-running: propose, do not start.
    MA->>BJ: start_background_job(proposal: full-text scan of documents of types T1, T2 for terms,<br/>estimated files: about 4,800, runs as you, batched and throttled)
    BJ-->>App: proposal card (scope, estimate, "runs with your access, you can leave")
    MA-->>App: interim answer: the 3 problem reports and 2 files found so far [R]<br/>"This covers item names, titles, descriptions and file names only." [C]
    App-->>U: interim answer + proposal
    U->>App: Confirm
    App->>BJ: confirm(job)
    BJ->>GW: obtain job-scoped renewable credential for this user (held by the Tool Gateway)
    loop batches, throttled as the user
        BJ->>GW: list files in scope (AML, paged)
        BJ->>FS: for each file: re-check access as user, stored text by file id or extract and store
        FS-->>BJ: text or unreadable/denied
        BJ->>BJ: match terms, collect passages with file id and page [R]
        BJ-->>App: progress (files scanned / total, matches so far)
    end
    BJ->>GW: drop job credential
    BJ->>MA: results (matched files and passages, counts, unreadable, denied)
    MA-->>App: final answer with citations, coverage
    App-->>U: answer (also visible if the user returns later)
```

Points to note:

- **Nothing runs without confirmation.** The proposal states what will be scanned, a rough size, that it runs with the user's own access, and that it may take a while. The user can decline and keep the interim answer.
- **Interim answer first.** The user gets what the cheap search found immediately, with coverage that says exactly what was and was not searched.
- **Job-scoped credential.** The job runs as the requesting user with a renewable credential that is scoped to this job, held by the Tool Gateway, and dropped when it ends. There is no privileged account. See [security-and-permissions.md](security-and-permissions.md#background-jobs-job-scoped-renewable-credentials).
- **Stored text makes later scans cheaper.** Extracted text is stored per immutable file id, so a later scan by anyone skips extraction for files already processed. Access is still re-checked through Aras, as that user, for every file on every use.
- **[C]** "Scanned 4,812 files you can access in document types T1 and T2. 640 had no text layer (scanned) and were not searched. 0 were denied. 14 files mention corrosion in a coastal context."
- **Leave and return.** Progress and results are attached to the session, which is saved in the Database; the user can close the browser and find the answer when they come back.

## Where each product rule shows up

| Rule | Walk-through 1 | 2 | 3 | 4 | 5 |
| --- | --- | --- | --- | --- | --- |
| Ask before expensive work [D] | not needed (cheap) | asks for "late" | asks what "so often" covers | asks which maps | proposal before scan |
| Disclose assumptions [A] | meaning of "open" | quarter, late | "so often" window | time-in-activity | search scope |
| Report coverage [C] | count, access-limited | records, unreadable, denied, unlinked | revisions without evidence | in-progress, missing timestamps | searched vs not searched |
| Cite everything [R] | records, knowledge entry | records, file pages | records, comments, file pages | runs, maps | records, file pages |
| Numbers computed in code | count from rows | Sandbox | Sandbox | Sandbox | job counts |
| Association, not causation | n/a | "appear most often on" | "documented reasons point to" | "largest share of time" | n/a |
| Learning signal | correction if any | explicit teaching | explicit teaching | none (the user's confirmation of the uncertain entry is evaluated by the Learner) | none |

## Open questions

- **Proposed:** the interim answer in walk-through 5 is shown before the background job is confirmed, so the user always has something. The exact wording and layout of the proposal card are specified in [chat-app.md](../specs/chat-app.md) and [background-jobs.md](../specs/background-jobs.md).
- **Proposed:** when a background job finishes, its results are composed into an answer by the Main Agent in the same conversation (rather than shown as raw matches). This keeps citation, coverage, and causal-honesty rules in one place.
- Per-question file budgets and relevance ordering of which files to read first are deliberately not tuned yet; they will be set once the system runs against real installations.
