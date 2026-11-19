# 0022. Prompt-based how-never-what enforcement in MVP

- Status: Accepted
- Scope: MVP

## Context

Shared learned knowledge must hold no business data values ([ADR 0018](0018-learned-knowledge-depends-only-on-configuration.md)). Enforcing that perfectly would need a check that understands whether a string is a data value or a configuration description, which is itself a model judgement.

## Decision

In the MVP, keeping learned knowledge free of data values relies on a strong initial prompt for the Learner and for agents proposing learnings, together with the shipped product skill on what to persist and the structural shape of learned entries (each maps to configuration elements). Conversations stored on the self-hosted server are treated as private.

## Alternatives considered

| Alternative | Why rejected for MVP |
| --- | --- |
| A mechanical check on every write (pattern matching against known data values, or a second model judging each entry) | Pattern matching cannot know which strings are data; a second model per write adds cost and its own error rate. Worth adding once real entries exist to test it on. Deferred as scheduled checks, possibly with small inexpensive models, that flag entries where data slipped in (see [roadmap](../product/roadmap.md)). |
| Human review of every entry | Rejected in [ADR 0019](0019-invalidation-by-provenance-and-contradiction.md). |

## Consequences

- The control is probabilistic, and the documentation says so ([security and permissions](../architecture/security-and-permissions.md#how-never-what)).
- The prompt and skill texts are critical artefacts ([docs/prompts/README.md](../prompts/README.md), [docs/skills/README.md](../skills/README.md)).
- Provenance on every entry means a leaked value can be traced and removed.

## Related

[ADR 0017](0017-three-knowledge-layers.md), [Knowledge and learning](../specs/knowledge-and-learning.md).
