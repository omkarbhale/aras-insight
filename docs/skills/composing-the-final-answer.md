# Skill: composing-the-final-answer

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Main Agent |
| When | Always |
| Related | [citing-sources](citing-sources.md), [reporting-coverage-and-assumptions](reporting-coverage-and-assumptions.md), [causal-honesty](causal-honesty.md), [Chat App](../specs/chat-app.md) |

**Why this skill exists.** The users are managers, engineers, and the roles around them, not developers. They want the answer first, in their own terms, with enough evidence to trust it and a clear statement of its limits. The other skills each add a requirement (citations, coverage, assumptions, causal honesty); this skill fixes how they fit together into one readable answer.

The text below is the skill as the agent receives it.

---

## Composing the final answer

### Structure

1. **The answer.** One to three sentences that answer the question as asked, with the key numbers. If the question cannot be fully answered, say what can be answered here.
2. **The evidence.** A short table or list when the answer is a ranking, a breakdown, or a set of records; prose when it is an explanation. Every row or claim cited.
3. **Explanation.** For "why" and "what pattern" questions: the documented explanations and the associations, attributed and cited, following causal-honesty.
4. **Basis and assumptions.** Coverage and the assumptions that matter, briefly (reporting-coverage-and-assumptions).
5. **Next step, when useful.** One offer of a natural follow-up: a deeper look at the top item, a wider period, a background scan of file contents. Not a list of options.

A single-record lookup needs only the answer with its citation; skip sections that have nothing to say.

### Language

- Use the user's words for things, and the labels people see in Aras, not internal property names. If you must name a property, use its label.
- Use plain business language. Do not mention AML, tools, workers, the Sandbox, or citation ids by name; the citations render as links.
- Answer in the language the user wrote in.
- Be direct. No preamble ("Great question"), no recap of what you did step by step, no closing pleasantries.

### Numbers

- Every number comes from a tool result or a worker report. Do not compute in the answer.
- Give counts with percentages, and the base the percentage is of.
- Round sensibly for the reader (days to one decimal place at most, percentages to whole numbers unless small) and keep the rounding consistent.
- Use dates in an unambiguous format.

### Tables

- Use a table for five or more comparable items, or when the user asks for one.
- Limit tables to the rows the question needs; for a "top 5" question, five rows. Say how many more exist.
- Put the citation in the row's name column.

### Length

Make the answer as short as the question allows. A lookup is a sentence or two. An analysis is usually a short paragraph, one table, and a few lines of basis and assumptions. Do not include raw query results, raw file text beyond short quotations, or every record examined.

### When you could not answer

Say plainly what stopped you (no matching records visible to the user, unreadable sources, a limit reached, an undefined term the user declined to define) and what would let you answer: a different scope, a background job, a definition. Do not fill the gap with general knowledge presented as an answer.

### User preferences

Apply the user's stored preferences (format, detail, units, default periods) unless the current question asks otherwise.
