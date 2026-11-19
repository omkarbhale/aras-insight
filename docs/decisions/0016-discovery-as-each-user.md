# 0016. Discovery runs as each user

- Status: Accepted
- Scope: MVP

## Context

Discovery reads the Aras configuration: item types, properties, relationships, lists, workflow maps, lifecycle maps, methods, forms. It could run with a privileged account on a schedule, but that would reintroduce the privileged identity rejected in [ADR 0005](0005-user-level-access-through-aras-server.md). In Aras, ordinary users typically already have read access to configuration, since the client needs it to render forms and relationships.

## Decision

Discovery runs as each user, with that user's credentials, through the Tool Gateway. It detects changes cheaply via configuration modification stamps and regenerates the discovered layer when something changed.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Discovery with an administrator account | Introduces a privileged identity; see [ADR 0005](0005-user-level-access-through-aras-server.md). |
| One-time discovery at installation | Goes stale; configuration changes continuously in many installations. |
| Full re-discovery on every question | Unnecessary Aras load; modification stamps make change detection cheap. |

## Consequences

- No new privileged identity.
- Different users could, in principle, see different configuration. How the discovered layer handles this is owned by [discovery.md](../specs/discovery.md); a proposal is in [security and permissions](../architecture/security-and-permissions.md#the-user-level-access-guarantee).
- That configuration is readable by ordinary users must be verified per Aras version ([deployment and operations](../architecture/deployment-and-operations.md#aras-version-compatibility-to-verify)).
- Changes are detected at the next check rather than instantly; there is no deployment hook ([ADR 0019](0019-invalidation-by-provenance-and-contradiction.md)).

## Related

[Discovery](../specs/discovery.md), [ADR 0004](0004-discovery-only-schema.md).
