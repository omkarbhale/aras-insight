# 0027. Causal honesty and coverage reporting

- Status: Accepted
- Scope: MVP, standing product rule

## Context

Users ask causal questions ("which components caused delays", "why do we keep revising this"). Aras data shows co-occurrence and sequence; it rarely shows cause. And every answer rests on a subset of the data: the records the user can access, the files that could be read, the links that exist. An answer that hides either fact overstates its certainty.

## Decision

Causal honesty is a standing product rule: the agent reports association ("appears most often on delayed change orders"), never causation the data cannot show. Where a cited document itself states a cause, the agent quotes and cites it as that document's statement. Every answer reports coverage: how many records it was based on (of those the user can access), what had no linked evidence, and what could not be read.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Answer in the user's causal framing | Produces confident claims the data does not support. |
| Report coverage only on request | Users rarely ask, and gaps then go unnoticed. |

## Consequences

- Every component that produces evidence must count what it saw, skipped, could not read, and was denied; worker reports carry coverage as a required part.
- Throttle stops and row budgets surface as coverage, never as silent truncation.
- Answers are somewhat longer; the Chat App presents coverage compactly.

## Related

[Question lifecycle](../architecture/question-lifecycle.md), [ADR 0029](0029-three-kinds-of-citation.md).
