# 0004. Discovery-only schema

- Status: Accepted
- Scope: MVP

## Context

Every Aras installation is customised: item types are added, properties renamed, relationships introduced, lifecycles and workflows reshaped. A product that needs a hand-written mapping of each customer's data model is expensive to deploy, goes stale as the configuration changes, and depends on someone who understands both the product and the installation.

## Decision

The schema is discovered only, never hand-configured per customer. Discovery reads item types, properties, relationships, lists, workflow maps, lifecycle maps, methods, and forms from Aras as the user, and maintains that knowledge automatically over time. What the configuration means to the business (for example what "delayed" or "released" means in this installation) is learnable instance knowledge, formed in use.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Per-customer configuration file mapping business concepts to item types | Deployment cost, drift as configuration changes, and a dependency on scarce expertise. |
| Hard-coded assumptions about standard Aras item types | Breaks on customised installations, which is most of them. Standard names are useful only as hints to verify. |
| Discovery plus an optional manual override layer | Reintroduces the drift problem. Corrections are instead captured as learned knowledge in conversation. |

## Consequences

- Installation needs no data-model work.
- Discovery quality is critical, and the Query Analyzer validates every query against the discovered schema ([ADR 0007](0007-static-analysis-for-validity-and-cost.md)).
- Business meaning is initially unknown and is supplied by users through definitions and corrections ([ADR 0028](0028-ask-before-expensive-analysis.md), [ADR 0020](0020-organisation-wide-learnings-with-disclosed-assumptions.md)).
- Standard Aras names are treated as hints to verify, not as facts, in the product's prompts and guidance.

## Related

[Discovery](../specs/discovery.md), [ADR 0016](0016-discovery-as-each-user.md), [ADR 0019](0019-invalidation-by-provenance-and-contradiction.md).
