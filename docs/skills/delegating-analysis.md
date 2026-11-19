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
- Reading up to 5 files, or a section of one.
- A count or total that the tool result itself reports.

**Start an Analysis Worker** with `start_analysis` when the answer needs computation or volume:

- Aggregation: counts by group, rankings, shares, distributions, durations, trends over time.
- Process questions: reconstructing workflow or lifecycle runs, comparing them with the definition, finding bottlenecks.
- Many files: reading and searching more than 5 files, or correlating file contents with records.
- Any number that needs a calculation: a sum, a difference, a share, an average, a rounded figure.

**Propose a background job** with `start_background_job` when the work is too large for a conversation turn, most often a full-text scan across the contents of files that the item graph cannot narrow. A background job runs only after the user confirms it. You do not write the confirmation message: the Chat App builds a card from your proposal, with Confirm and Decline buttons, and shows the numbers from the tool result as they are. Your `title` becomes the card's heading and your `explanation` its sentence, so write the explanation as one plain sentence about what the scan will look for and what the user will get. In your own reply, give only the interim answer (what you could establish without the scan); the card carries the proposal itself.

You do no arithmetic yourself. Every number in your answer comes from a tool result or a worker report; anything that needs a calculation, even a single percentage, comes from a worker.

### Before starting a worker

1. Resolve deciding terms (clarifying-definitions). A worker cannot ask the user anything.
2. Identify the item types, properties, relationships, and states the work needs, using `schema_describe` and knowledge. Confirm names; do not pass guesses to a worker. A worker has no schema or knowledge tools: it can work only with what you name, so name every item type the work touches and every knowledge entry it may rely on. If you leave something out, the worker can only report that it is stuck.
3. If you are unsure how large the population is, run a cheap bounded query first to size it.

### Splitting work

- A multi-part question whose parts are independent ("which parts appear most often on late change orders, and what the linked reports say went wrong with them") can run as parallel workers, one per part, when the second part does not need the first part's result. When it does, run them in sequence and pass the first report's findings into the second brief.
- Do not split one population across several workers to get around a row budget. Budgets apply per question.
- Prefer one well-scoped worker over several small ones. Start parallel workers only when the parts are genuinely separate.

### Writing the brief

You give a worker its brief through the inputs of `start_analysis`; the service renders them into the analysis job brief. Fill every part that applies. A good brief:

- States the question the worker must answer in one or two sentences, in data terms.
- Gives every definition with its source: `user` (the user gave it in this conversation), `knowledge` (a knowledge entry defines it, including a Discovered suggestion, which is a knowledge entry), or `assumption` (your own choice). Gives the scope: period with its time zone, item types, current or all revisions, and inclusions and exclusions.
- For a multi-part question, says what the other parts cover and passes on any earlier findings this job needs, with their citation ids.
- Names the confirmed schema elements to use. The service builds the worker's schema extract from the item types you name.
- Names the starting records, when the job must begin from specific records, by citation id (every record a worker sees carries one).
- Lists the knowledge entries the worker may rely on, by id, and any domain-specific skills the job needs. You do not name uncertain-and-contested-knowledge: the service attaches it whenever an entry is not independently confirmed, or is uncertain or contested.
- States the expected output: which numbers, which ranking, which table, how many examples to cite, and how to present each figure (decimals, percentages), because you will state the figures as they come back.
- States limits as instructions to the worker: how many files it should read at most (this is your instruction in the brief, not a system budget; the system budgets queries, rows, bytes, and time), and what to do if the population is larger than expected. A worker never narrows the scope on its own: when it reaches a limit it stops, analyses what it has, and reports the result as partial (see working-within-limits).
- Adds notes for anything else the worker must know, such as a data quirk established earlier in the conversation.

Inputs you leave out get the service's defaults; for example the worker's budget is an even share of what remains, split among the workers you start in the same step.

### Reading the report

When a report comes back, check it before using it:

- Does it answer the question that was asked, with the definitions that were given?
- Are its numbers computed (it says how) and do they reconcile (parts add up to the total it states)?
- Does every finding carry citations?
- Does it report coverage and assumptions?
- Does it claim causation? If so, restate as association (causal-honesty).

If a report is incomplete or inconsistent, start a corrected worker with a brief that names the problem, at most once. If it is still unusable, tell the user what could and could not be established.

If a report is `partial` because the worker reached a limit, the report says exactly what was not covered. Whether to run a narrower follow-up is your decision, because narrowing changes the question the brief set: either start one worker with a narrower scope and tell the user what the narrower question covers, or tell the user what was and was not covered and offer the narrower or wider run.

A report may carry candidate learnings, and may say that a knowledge entry looks wrong given the data. Neither is one of the four learning signals. Do not call `record_learning` for them, and do not treat a worker's doubt as a reason to mark an entry uncertain. They stay in the stored report; the batched Learner may use them only when the conversation also carries a signal. If the doubt affects your answer, say so in the answer's assumptions.

### Telling the user

When you start a worker or several, tell the user in one line what you are doing and that they will see progress and can leave and return. Do not narrate tool calls.
