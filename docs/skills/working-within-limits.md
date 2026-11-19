# Skill: working-within-limits

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Main Agent, Analysis Worker |
| When | Always |
| Related | [Query Analyzer](../specs/query-analyzer.md), [Tool Gateway and Throttle](../specs/tool-gateway.md), [tool catalog](../specs/tool-catalog.md) |

**Why this skill exists.** Every query the agents issue runs on the customer's production Aras server as the signed-in user. The Query Analyzer checks each query for validity and cost, and the Throttle enforces per-user and system-wide limits and a row budget per question. These controls protect the people doing day-to-day work in Aras. They only work if the agents respond to them correctly: fix an invalid query rather than repeating it, narrow an expensive one, wait when throttled, and report what a limit left uncovered rather than working around it.

The text below is the skill as the agent receives it.

---

## Working within limits

The data tools protect a shared production system. Their limits are not obstacles to get around; they are part of the job.

### Query Analyzer diagnostics

A query tool may return diagnostics instead of results. Each diagnostic names a problem and where it is.

**Validity diagnostics** mean the query refers to something that does not exist or uses it wrongly: an unknown item type, property, or relationship; an unknown list value or state; a value of the wrong type; a field that would be silently ignored.

- Read the diagnostic, then check the schema with `schema_describe` (Main Agent) or the schema extract in your brief (Analysis Worker).
- Correct the query and retry. Never resend the same query unchanged.
- If a knowledge entry told you to use the name that failed, the entry may be stale. Mention this in your answer or report so that it can be marked uncertain, and do not rely on the entry for the rest of the question.

**Cost diagnostics** mean the query would be expensive: it selects every property, has no row limit or paging, nests too deeply, or uses a broad wildcard.

- Select only the properties you need.
- Add a row limit and page through results.
- Replace deep nesting with a second query over the ids from the first.
- Narrow wildcards, or add another condition (a date range, a type, a state).

Proposed: after three failed attempts at the same query, stop and report what you were trying to retrieve and the last diagnostic. Do not keep varying the query blindly.

### Row budgets

Each question has a row budget. Tool results tell you how much has been used.

- Plan for the budget. Size the population with a cheap query before fetching everything.
- If the population exceeds the budget, narrow the scope (a shorter period, one product line) and say so, or, for the Main Agent, ask the user whether to narrow or to run a background job.
- Never split work across extra queries, extra workers, or extra conversations to exceed a budget.
- When the budget stops the work, report exactly what was and was not covered.

### Throttling

A tool may report that it is throttled: too many queries are running for this user or for the whole system.

- Wait as the tool result advises, then retry. Do not retry immediately in a loop.
- Reduce concurrency: issue queries one at a time rather than in parallel.
- If throttling persists beyond the time the tool advises, report that the work could not be completed now and what was covered so far.

### Timeouts

A query that times out was too large or too slow. Treat it as a cost diagnostic: narrow it, page it, or split it by a natural key (for example by month), within the row budget.

### Access denied

A tool may report that the user cannot access a record or file. This is a fact about the user's permissions, not an error to work around.

- Do not try another route to the same record or file.
- Do not ask the user for other credentials, and do not suggest that someone with more access could run the question for them unless they ask how access works.
- Report it in coverage.

### Read-only

The product is read-only. Never attempt to create, change, delete, lock, promote, or otherwise act on anything in Aras. The tools will refuse such requests; do not try to phrase one differently. If the user asks for a change, say that this product only reads and explain where in Aras they could make the change, if you know.
