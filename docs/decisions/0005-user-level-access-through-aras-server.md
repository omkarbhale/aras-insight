# 0005. User-level access through the Aras server only

- Status: Accepted
- Scope: MVP. The no-hybrid and no-administrator-aggregates stance is standing; the no-read-replica part holds for the MVP only (see [roadmap](../product/roadmap.md)).

## Context

Analytical questions ("which parts appear most often on change orders that finished late last quarter") touch many records. There is a pull toward giving the product broad access for speed and completeness: a reporting copy, an administrator-granted view of everything, or direct database reads. Each of those makes the product, rather than Aras, the place where access control is decided. Aras permissions are rich (identities, permissions per item and state, classification, ownership and team rules) and can be extended by custom server methods that run when items are read; re-implementing them faithfully outside Aras is error-prone, and any gap leaks data.

## Decision

Access is user-level only. Every data access in the MVP goes through the Aras server, as the signed-in user, through the Tool Gateway. There is no privileged service account, no hybrid model, no administrator-granted organisation-wide aggregates, and no read replica.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Hybrid analytical copy: a periodically refreshed copy of Aras data for analytics, with per-user filtering | Requires re-implementing Aras permissions, including custom read-time server logic, over the copy. Introduces a store of business data and its retention questions. Adds a second source of truth that can disagree with Aras. |
| Administrator-granted organisation-wide aggregates (for example "anyone may see counts across all records") | An aggregate can disclose information about records a user cannot open, especially at small counts. It makes the customer's administrators responsible for a new disclosure policy, and it is a different product from "ask your Aras". |
| Read replica of the Aras database in the MVP | Bypasses the Aras server and therefore its permission model and server events. A replica is useful to keep heavy analytics off production, but only if user-level access stays faithful, which is hard. Deferred, not rejected forever (see [roadmap](../product/roadmap.md)). |

## Consequences

- Aras enforces its own permissions on every read; the product never re-implements them.
- Answers depend on who asks. Two users can get different counts for the same question, and every answer's coverage says it is based on records the user can access ([ADR 0027](0027-causal-honesty-and-coverage-reporting.md)).
- All analytical load lands on the production Aras server. This is controlled by static cost analysis, throttling, and asking before expensive work ([ADR 0007](0007-static-analysis-for-validity-and-cost.md), [ADR 0030](0030-throttling-in-mvp.md), [ADR 0028](0028-ask-before-expensive-analysis.md)).
- Long-running work needs the user's own credential for its duration ([ADR 0013](0013-user-confirmed-background-full-text-scan.md)).
- No shared cache is possible without permission scoping ([ADR 0023](0023-no-caching.md)).

## Related

[Security and permissions](../architecture/security-and-permissions.md), [Tool Gateway](../specs/tool-gateway.md).
