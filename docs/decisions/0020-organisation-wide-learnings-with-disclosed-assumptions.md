# 0020. Organisation-wide learnings with disclosed assumptions

- Status: Accepted
- Scope: MVP

## Context

When one user teaches the agent what a term means ("by delayed we mean released after the target date"), should others benefit? If learnings stay personal until enough users agree, small teams never reach the threshold and behaviour changes with deployment size. If learnings are shared immediately, one person's wrong or idiosyncratic meaning can affect everyone.

## Decision

Conversation-formed learnings are organisation-wide immediately, with provenance (which conversation, taught by one person). There are no user-count thresholds, so behaviour is the same at any scale. Until someone other than the teacher confirms an entry, the agent states it as an explicit assumption when using it. Carrying on without objection confirms it; objecting makes it contested. After one independent confirmation the agent stops announcing it. In a single-user deployment the teacher's word is enough. Per-user preferences stay in their own personal layer.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Personal-first scoping, promoted to shared after a user-count threshold | Behaviour differs between a 5-person and a 500-person deployment; small teams never benefit; thresholds are arbitrary. |
| Shared immediately, silently | A wrong meaning spreads unnoticed. |
| Administrator approval before sharing | A review queue that blocks learning; see [ADR 0019](0019-invalidation-by-provenance-and-contradiction.md). |

## Consequences

- One teaching benefits everyone at once, and is reviewed by its next users in the course of their work.
- Answers must show assumptions clearly, and the Chat App must make objecting easy.
- Contested entries require the agent to ask which meaning the user intends.
- Shared entries must hold no data values ([ADR 0018](0018-learned-knowledge-depends-only-on-configuration.md)).

## Related

[Knowledge and learning](../specs/knowledge-and-learning.md), [ADR 0021](0021-learn-from-signals-and-merge.md).
