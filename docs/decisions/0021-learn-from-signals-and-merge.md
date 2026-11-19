# 0021. Learn only from signal-bearing conversations; merge rather than append

- Status: Accepted
- Scope: MVP

## Context

If the Learner processed every conversation and appended whatever it found, the knowledge base would grow quickly with duplicates and near-duplicates, loading by index and description would degrade, and contradictory entries would multiply.

## Decision

The Learner learns only from conversations that carry a signal: a correction, a retried question, an empty result, or explicit teaching. It merges into an existing entry about the same term or configuration element rather than appending a duplicate. Learning runs live during a conversation (`record_learning`) or in batch over a session's transcript after an inactivity threshold.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Learn from every conversation | Volume and noise; most conversations teach nothing new. |
| Append-only entries with periodic clean-up | Duplicates degrade retrieval between clean-ups; clean-up is itself a hard merge problem. |
| Live learning only | Misses signals that are clear only in hindsight over the whole transcript. |
| Batch learning only | Loses the chance to apply a definition within the same conversation. |

## Consequences

- Knowledge volume grows with genuine new meaning, not with usage.
- Each entry is keyed by the term or configuration element it describes, so merge targets can be found.
- The background model slot handles batched learning and the other Learner steps; live calls take priority over batched work on that slot ([ADR 0026](0026-three-model-slots.md), [ADR 0051](0051-model-slot-requirements-and-budgets.md)).
- Live and batched learning are both core capabilities: live learning lets a definition apply later in the same conversation and for others at once ([ADR 0053](0053-core-capabilities.md)).
- Ageing out unused entries is deferred (see [roadmap](../product/roadmap.md)).

## Related

[Knowledge and learning](../specs/knowledge-and-learning.md), [Background Jobs](../specs/background-jobs.md), [ADR 0044](0044-learner-operations-and-signals.md), [ADR 0053](0053-core-capabilities.md).
