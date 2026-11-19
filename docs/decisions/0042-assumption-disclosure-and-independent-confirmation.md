# 0042. Assumptions are disclosed to every other user until independently confirmed

- Status: Accepted
- Scope: MVP

## Context

A taught definition is organisation-wide at once ([ADR 0020](0020-organisation-wide-learnings-with-disclosed-assumptions.md)). Until someone other than the teacher confirms it, the agent says so when it uses the entry, so that others can object. Several details had to be fixed.

## Decision

- The teacher is not told; every other reader is. The usage note is computed per reader. Workers receive every unconfirmed, uncertain or contested Learned entry and the Main Agent filters per reader.
- A confirmed Learned entry carries `confirmation_status`: `unconfirmed` or `independently_confirmed`. The four states are unchanged. One independent confirmation ends the announcement.
- `knowledge.single_user_deployment` (default false) makes new entries start independently confirmed. If switched off later, entries confirmed only by that rule become unconfirmed again, so each records that a rule supplied its confirmation.
- The Learner, on the background slot, judges the user's next message: carrying on confirms, an objection makes the entry contested, anything else records nothing. Leaving is not confirmation. There are no accept or reject buttons. "Correct this" marks the entry uncertain.
- An unconfirmed entry counts as a usable definition: the agent proceeds and discloses. It asks only when none exists.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Tell the teacher too | The disclosure exists so that others can object. |
| Infer single-user mode from user count | Behaviour would depend on scale. |
| Keep rule-confirmed entries confirmed | The exemption's reason, nobody else to object, is gone. |
| Boolean flag, count or fifth state | A fifth state changes the four decided states. |
| Buttons, or any follow-up confirms | A rule cannot tell an objection from carrying on. |
| Ask first for a teacher-only entry | Every taught definition becomes a question. |

## Consequences

- Each disclosed assumption costs one background model call.
- An entry stays unconfirmed while replies are unclear or users leave.

## Related

[Knowledge and learning](../specs/knowledge-and-learning.md), [Main Agent](../specs/main-agent.md), [ADR 0043](0043-contested-knowledge.md), [ADR 0044](0044-learner-operations-and-signals.md).
