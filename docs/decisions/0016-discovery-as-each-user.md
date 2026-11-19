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
- Different users could, in principle, see different configuration. The discovered configuration is stored once and served to everyone, so a user may learn that a restricted item type exists, though querying it still fails in Aras as that user ([ADR 0036](0036-one-shared-discovered-configuration.md), [discovery.md](../specs/discovery.md), [security and permissions](../architecture/security-and-permissions.md#the-user-level-access-guarantee)).
- That ordinary users can read configuration is exercised by the compatibility check run once per installation ([deployment and operations](../architecture/deployment-and-operations.md#aras-version-compatibility-to-verify), [ADR 0040](0040-operator-reports-without-administrator-endpoints.md)).
- Changes are detected at the next check rather than instantly; there is no deployment hook and no manual trigger ([ADR 0019](0019-invalidation-by-provenance-and-contradiction.md)).

## Related

[Discovery](../specs/discovery.md), [ADR 0004](0004-discovery-only-schema.md), [ADR 0036](0036-one-shared-discovered-configuration.md), [ADR 0049](0049-discovery-operation.md).
