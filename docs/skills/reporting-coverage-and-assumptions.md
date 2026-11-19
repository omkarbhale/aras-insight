# Skill: reporting-coverage-and-assumptions

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Main Agent, Analysis Worker |
| When | Always |
| Related | [citing-sources](citing-sources.md), [uncertain-and-contested-knowledge](uncertain-and-contested-knowledge.md), [composing-the-final-answer](composing-the-final-answer.md) |

**Why this skill exists.** An answer is only as trustworthy as the user's understanding of what it was based on. Every query runs as the signed-in user, files may be unreadable, evidence may be missing for some records, and a budget may stop a scan early. A number without its basis invites the user to over-trust it. This skill makes coverage and assumptions a required part of every answer, not an optional caveat.

The text below is the skill as the agent receives it.

---

## Reporting coverage and assumptions

Every answer states what it was based on and what it assumed. The user should never have to ask "out of how many?" or "what did you take that word to mean?".

### Coverage: what the answer is based on

Report these, in plain language, for every answer that rests on more than a single record:

1. **Basis.** How many records the answer was computed from, and what they were ("142 change orders created between 1 July and 30 September").
2. **Visibility.** That the records are those the user can access. Say it once, simply: "based on the records you have access to". Do not speculate about records the user cannot see, and never estimate how many there might be.
3. **Missing evidence.** Records for which the evidence the question needed was absent: no linked files, no comments, an empty field, no workflow history. Give the count and, when short, the list with citations.
4. **Unreadable sources.** Files that could not be read and why: a scanned document with no text layer, a format this release does not read, a file too large to process, access denied. Give the count and name the files when there are few.
5. **Limits reached.** Anything not examined because a budget, a row limit, or a time limit stopped the work. Say what was not covered and whether a larger job (for example a background scan) could cover it.
6. **Read-time filtering.** If Discovered knowledge says an item type's results may be filtered or altered when read, mention that the results reflect what the system returned to the user.

For a single-record lookup, coverage is implicit; mention it only if something was missing or unreadable.

### Assumptions: what the answer took for granted

State every assumption that could change the answer if it were wrong:

- **Definitions** of terms the question used ("delayed", "late", "recurring", "critical"), with the source of the definition: the user in this conversation, a Learned entry (cite it), a Discovered suggestion, or your own choice.
- **Scope** choices: the time window, which item types were included, whether all revisions or only current ones were counted, how duplicates were treated.
- **Unconfirmed or uncertain knowledge** you relied on. Follow uncertain-and-contested-knowledge for the wording.
- **Interpretation of the data**, for example treating a specific date property as the completion date.

State an assumption once, near the result it affects. Do not bury assumptions in a long list of generic disclaimers; list only those that are real and specific.

### Wording

- Lead with the result, then coverage, then assumptions. Keep coverage to a short paragraph or a few bullets.
- Use numbers, not vague words: "38 of 142 had no linked report" rather than "some had no report".
- Do not apologise and do not hedge the whole answer because part of it is incomplete. Report the gap precisely and let the user decide.
- Never present coverage you did not measure. If you did not count something, do not estimate it.

### For Analysis Workers

Your report has separate coverage and assumptions fields. Fill them with counts computed in the Sandbox, not estimates. The Main Agent will carry them into the answer.
