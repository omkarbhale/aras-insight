# Skill: delegating-analysis

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Main Agent |
| When | Always |
| Related | [analysis job brief](../prompts/analysis-job-brief.md), [background-job confirmation wording](../prompts/background-job-confirmation.md), [Analysis Worker and Sandbox](../specs/analysis-worker-and-sandbox.md), [background jobs](../specs/background-jobs.md) |

**Why this skill exists.** The Main Agent has three ways to answer: directly through the Tool Gateway, by starting one or more Analysis Workers, or by proposing a background job. Answering a simple lookup with a worker wastes time; answering an aggregation in the conversation produces numbers the model did not compute. This skill fixes when each path is used, how work is split, and how worker reports are checked before they reach the user.

The text below is the skill as the agent receives it.

---

## Delegating analysis

### Choose the path

**Answer directly** with `aml_query`, `file_read`, `schema_describe`, and knowledge tools when the answer is a lookup:

- One record, or a short list of records, and their properties.
- A relationship walk of one or two steps from a known item.
- Reading one or a few files, or a section of one.
- A count or total that the tool result itself reports.

**Start an Analysis Worker** with `start_analysis` when the answer needs computation or volume:

- Aggregation: counts by group, rankings, shares, distributions, durations, trends over time.
- Process questions: reconstructing workflow or lifecycle runs, comparing them with the definition, finding bottlenecks.
- Many files: reading and searching more than a few files, or correlating file contents with records.
- Any number you would otherwise have to compute yourself from more than a few rows.

**Propose a background job** with `start_background_job` when the work is too large for a conversation turn, most often a full-text scan across the contents of files that the item graph cannot narrow. A background job runs only after the user confirms it. Use the background-job confirmation wording.

You never compute aggregates yourself from rows in your context. If a number requires arithmetic over more than a few values, it comes from a worker.

### Before starting a worker

1. Resolve deciding terms (clarifying-definitions). A worker cannot ask the user anything.
2. Identify the item types, properties, relationships, and states the work needs, using `schema_describe` and knowledge. Confirm names; do not pass guesses to a worker.
3. If you are unsure how large the population is, run a cheap bounded query first to size it.

### Splitting work

- A multi-part question whose parts are independent ("which parts appear most often on late change orders, and what the linked reports say went wrong with them") can run as parallel workers, one per part, when the second part does not need the first part's result. When it does, run them in sequence and pass the first report's findings into the second brief.
- Do not split one population across several workers to get around a row budget. Budgets apply per question.
- Prefer one well-scoped worker over several small ones. Start parallel workers only when the parts are genuinely separate.

### Writing the brief

Fill the analysis job brief completely. A good brief:

- States the question the worker must answer in one or two sentences, in data terms.
- Gives every definition, with its source, and the scope (period, item types, current or all revisions).
- Names the confirmed schema elements to use.
- Lists the knowledge entries the worker may rely on, by id, and the skills it needs.
- States the expected output: which numbers, which ranking, which table, how many examples to cite.
- States limits as instructions to the worker: how many files it should read at most (this is your instruction in the brief, not a system budget; the system budgets queries, rows, bytes, and time), and what to do if the population is larger than expected.

### Reading the report

When a report comes back, check it before using it:

- Does it answer the question that was asked, with the definitions that were given?
- Are its numbers computed (it says how) and do they reconcile (parts add up to the total it states)?
- Does every finding carry citations?
- Does it report coverage and assumptions?
- Does it claim causation? If so, restate as association (causal-honesty).

If a report is incomplete or inconsistent, start a corrected worker with a brief that names the problem, at most once. If it is still unusable, tell the user what could and could not be established.

### Telling the user

When you start a worker or several, tell the user in one line what you are doing and that they will see progress and can leave and return. Do not narrate tool calls.
