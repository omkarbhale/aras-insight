# 0018. Learned knowledge depends only on configuration

- Status: Accepted
- Scope: MVP

## Context

Learned knowledge is shared organisation-wide ([ADR 0020](0020-organisation-wide-learnings-with-disclosed-assumptions.md)). If it could contain business data ("change order 123 was late", "most delays involve supplier S"), a user could learn through it something they cannot open in Aras. It would also go stale whenever data changed, which is constantly, making invalidation intractable.

## Decision

Learned knowledge depends only on Aras configuration, never on business data. It records how, never what: what a term means in terms of configuration elements (item types, properties, list values, lifecycle states, workflow maps and activities, relationships), never values of business records. Because of this, configuration changes are the only staleness trigger.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Allow learned findings about data ("the usual bottleneck is activity A") | Leaks data across permission boundaries; goes stale with every new run; turns knowledge into an unaudited cache. |
| Allow data values but scope them per user | Complex, and still goes stale with data; per-user preferences already exist for personal working style. |

## Consequences

- Shared knowledge is safe to share, provided the rule holds; MVP enforcement is by prompt ([ADR 0022](0022-prompt-based-how-never-what-enforcement.md)).
- Invalidation needs to watch only configuration ([ADR 0019](0019-invalidation-by-provenance-and-contradiction.md)).
- Every learned entry maps to configuration elements, which gives it a concrete shape that can be checked against discovery.
- Findings are recomputed each time they are asked for; there is no data memory ([ADR 0024](0024-no-data-history.md)).

## Related

[Security and permissions: how, never what](../architecture/security-and-permissions.md#how-never-what), [Knowledge and learning](../specs/knowledge-and-learning.md).
