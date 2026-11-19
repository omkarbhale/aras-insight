# 0030. Throttling in MVP

- Status: Accepted
- Scope: MVP

## Context

All queries run against the production Aras server as users ([ADR 0005](0005-user-level-access-through-aras-server.md)). An analytical question, a multi-worker analysis, or a full-text scan can issue many queries and downloads. Interactive Aras users must not see latency or locking because of the product.

## Decision

Throttling is in the MVP. The Throttle, inside the Tool Gateway, enforces per-user and system-wide limits on concurrent and total queries, row budgets per question, and timeouts, using Query Analyzer cost estimates. Background scans are batched and throttled. Refusals and budget stops are reported back to the agent and appear in coverage.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Rely on Aras's own protections | Not designed to distinguish product load from interactive load. |
| Throttle only by request rate | A single unbounded query can be worse than many small ones; cost estimates matter. |

## Consequences

- Limits must be sized per installation and monitored. The defaults are starting points for a pilot of up to 25 active users, replaced with measurements from the installation ([deployment and operations](../architecture/deployment-and-operations.md#sizing-considerations)).
- Some answers will be partial under load, and say so.
- Read queries are not expected to take item locks in Aras. That is one of the documented manual checks that the customer's Aras owners confirm per version ([ADR 0040](0040-operator-reports-without-administrator-endpoints.md)); the main concern is load, which this bounds.

## Related

[ADR 0007](0007-static-analysis-for-validity-and-cost.md), [Tool Gateway](../specs/tool-gateway.md).
