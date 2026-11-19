# 0023. No caching in MVP

- Status: Accepted
- Scope: MVP

## Context

The same analytical question may be asked many times. A cache of answers or query results (for example in an in-memory store) would reduce Aras load and latency. But a cached result was computed with one user's permissions; serving it to another user can leak data, and Aras data changes continuously, so cached results go stale.

## Decision

Caching is not in the MVP. Repeated questions are re-run against Aras every time. Aras load is controlled by static cost analysis and throttling instead ([ADR 0007](0007-static-analysis-for-validity-and-cost.md), [ADR 0030](0030-throttling-in-mvp.md)). Extracted file text per immutable file id is not a cache of results, and is permission-checked on every use ([ADR 0014](0014-extracted-text-per-immutable-file-id.md)).

## Alternatives considered

| Alternative | Why rejected for MVP |
| --- | --- |
| Answer or query-result cache shared across users | Crosses access boundaries. |
| Per-user cache | Staleness and invalidation complexity for limited MVP benefit. |
| Cache keyed by normalised query plus the user's permission scope | The right shape, but defining "permission scope" faithfully is non-trivial. Deferred (see [roadmap](../product/roadmap.md)). |

## Consequences

- Every answer reflects Aras at the time of asking.
- Repeated heavy questions cost Aras load each time; throttling bounds it.
- No extra component to deploy ([ADR 0015](0015-one-postgresql.md)).

## Related

[ADR 0005](0005-user-level-access-through-aras-server.md).
