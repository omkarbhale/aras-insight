# 0036. One shared discovered configuration for all users

- Status: Accepted
- Scope: MVP

## Context

Discovery runs as each user ([ADR 0016](0016-discovery-as-each-user.md)), and ordinary Aras users normally have read access to configuration. An installation can still show different users different configuration, for example an item type visible only to some identities. The product must either store the discovered configuration once and serve it to everyone, or limit what each user is served. Instance knowledge is shared by all users ([ADR 0017](0017-three-knowledge-layers.md)), and the Query Analyzer validates against one schema.

## Decision

The discovered configuration is stored once per deployment and served to every user. There is no per-user filtering and no per-user copy. Discovery reads configuration only; it does not read Aras identities or permission definitions, because Aras applies permissions when a query runs.

If an installation hides some configuration from some users, a user can learn through `schema_describe` that a restricted item type exists. Querying it still fails in Aras as that user, so only the existence of the configuration is exposed, never data. An item the triggering user cannot see is not treated as deleted.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Per-user filtering: record which users have seen each item | New tables, a more complex Query Analyzer and shared knowledge, to hide only the existence of configuration. |
| A separate discovered copy per user | Multiplies Discovery work and storage; shared knowledge could no longer refer to one schema. |
| Read permission definitions in Discovery | Puts access decisions in the product instead of Aras and reopens the per-user question. |

## Consequences

- One schema serves the Query Analyzer, `schema_describe` and shared knowledge.
- The existence exposure is a stated limitation in the security documentation.
- The design relies on every Aras user having configuration read access, which the compatibility check exercises.

## Related

[Discovery](../specs/discovery.md), [Security and permissions](../architecture/security-and-permissions.md#the-user-level-access-guarantee), [Persistence](../specs/persistence.md), [ADR 0004](0004-discovery-only-schema.md), [ADR 0016](0016-discovery-as-each-user.md), [ADR 0017](0017-three-knowledge-layers.md).
