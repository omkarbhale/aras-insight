# 0050. Background full-text scans: proposal, decision flow, partial results, and completion

- Status: Accepted
- Scope: MVP

## Context

A full-text scan reads many files as the user, so the user confirms it first ([ADR 0013](0013-user-confirmed-background-full-text-scan.md)). The flow around that confirmation was open.

## Decision

- The agent gives an interim answer from the cheap search (names, titles, descriptions, file names) together with the proposal.
- The Chat App builds the proposal card from the structured proposal; the tool's `title` and `explanation` are its heading and sentence. Buttons are Confirm and Decline, then Cancel while the job runs.
- Confirm, Decline and Cancel only change state. They start no Main Agent turn and cost no model call.
- On completion Background Jobs starts a Main Agent turn that composes a cited answer with coverage. If the user is away, the answer waits, with a notification on return.
- Cancelling keeps the matches found so far and produces a partial answer through the same path, with coverage naming the files not searched.
- An analysis that exceeds its budget ends as a partial report and is not converted into a background job.
- Search terms are literal strings (case sensitivity, whole word, any or all, proximity window). Regular expressions are not accepted.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| The Main Agent writes the card | A model would rewrite numbers and scope. |
| Every decision starts a turn | Model cost with nothing new to say. |
| Post a summary only | The user returns to nothing finished. |
| Show raw matches | Citation, coverage and causal rules would need a second home. |
| Discard matches on cancel | Breaks the promise that cancelling keeps what was found. |
| Regular expressions | Pathological patterns threaten availability. |

## Consequences

- Model cost is spent on completion even if the user is away, bounded by the per-turn token budget.
- Several literal variants replace a pattern.
- Declining the scan leaves the user with the interim answer.

## Related

[Background Jobs](../specs/background-jobs.md), [Chat App](../specs/chat-app.md), [question lifecycle](../architecture/question-lifecycle.md), [ADR 0013](0013-user-confirmed-background-full-text-scan.md), [ADR 0051](0051-model-slot-requirements-and-budgets.md).
