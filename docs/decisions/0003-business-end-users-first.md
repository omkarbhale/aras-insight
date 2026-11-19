# 0003. Business end users first

- Status: Accepted
- Scope: MVP

## Context

Aras is used by many roles: engineers, change analysts, quality and manufacturing staff, managers, and executives, as well as the developers and administrators who configure it. These groups need very different things. Developers want to inspect methods, item type definitions, and server events; business users want answers about their products and processes without learning AML or the data model.

## Decision

The MVP targets business end users across all roles (managers, executives, engineers, and the roles around them). Features aimed at Aras developers and administrators are not part of the MVP.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Developers and administrators first | Smaller audience, and a group already able to answer their own questions with Aras tooling. The larger unmet need is people who cannot write AML. |
| Both at once | Splits design attention: answer style, guidance, and safeguards differ for the two groups. |

## Consequences

- Answers are in plain language, with citations as links into Aras rather than raw AML.
- The product must hide the data model: users never need to know item type or property names. This makes discovery and learned definitions essential ([ADR 0004](0004-discovery-only-schema.md), [ADR 0017](0017-three-knowledge-layers.md)).
- Missing definitions are resolved with the user in their own vocabulary ([ADR 0028](0028-ask-before-expensive-analysis.md)).
- Developer and administrator features are deferred (see [roadmap](../product/roadmap.md)).

## Related

[Personas and use cases](../product/personas-and-use-cases.md).
