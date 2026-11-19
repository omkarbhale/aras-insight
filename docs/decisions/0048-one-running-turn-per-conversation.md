# 0048. One running turn per conversation

- Status: Accepted
- Scope: MVP

## Context

A turn can run for minutes: the Main Agent may start several Analysis Workers, and the conversation has one Sandbox. If the user sends a second message while a turn is running, the product must decide what happens. A second turn in the same conversation would work from a transcript that does not yet hold the first answer, would share the Sandbox and the question budget, and would be hard to cancel cleanly. The rejection also needs an error code that means what it says.

## Decision

A conversation has at most one running turn. A second message from the user in that conversation is rejected, not queued. The user waits for the answer or cancels the running turn. The Chat App disables the input box and shows a Cancel button while a turn runs. The rejection carries an error code of its own, `TURN_IN_PROGRESS`, so that `THROTTLED` keeps a single meaning: a concurrency or rate limit with a `retry_after_ms`.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Reject with `THROTTLED` | `THROTTLED` promises a retry hint, which has no meaning for a turn that is simply running. |
| Queue the second message and run it after the first | The queued message may have been written without seeing the answer it now follows, cancelling becomes ambiguous (which turn?), and one answer at a time is easier to reason about. |

## Consequences

- A user who thinks of a correction mid-turn waits or cancels first.
- Clients of the Chat App API handle one more error code.
- Cancel needs no queue handling, and a saved transcript never holds two overlapping turns.

## Related

[Main Agent](../specs/main-agent.md), [Chat App](../specs/chat-app.md), [tool catalog](../specs/tool-catalog.md), [persistence](../specs/persistence.md), [ADR 0010](0010-main-agent-plus-analysis-workers.md), [ADR 0032](0032-saved-sessions-and-live-progress.md).
