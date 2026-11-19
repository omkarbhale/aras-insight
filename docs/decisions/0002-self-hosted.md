# 0002. Self-hosted

- Status: Accepted
- Scope: MVP

## Context

Aras data is product intellectual property: designs, suppliers, quality problems, change history. Many Aras customers run Aras inside their own network and are unwilling to let a third-party service hold credentials to it or keep copies of its content. They also often have their own model infrastructure and want to use it.

## Decision

Aras Insight is deployed and operated by the customer, inside their own network, next to their Aras installation. The makers of the product do not operate it and receive no data from it. Data leaves the deployment only to the model endpoints the customer configures.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Hosted service operated by the product's makers | Requires holding user tokens for the customer's Aras outside their network, and storing transcripts and extracted file text outside it. A barrier to adoption for the intended customers, and a large security and compliance burden for the makers. |
| Hybrid: hosted control plane, on-premise data plane | Adds a second deployment surface and a network dependency for little MVP benefit. Can be revisited if operating at scale makes it worthwhile. |

## Consequences

- The customer controls where every piece of data lives, including transcripts and extracted file text.
- The product must be easy to install, configure, upgrade, and monitor by the customer's own staff (see [deployment and operations](../architecture/deployment-and-operations.md)).
- Model endpoints must be configurable, including internal ones ([ADR 0026](0026-three-model-slots.md)).
- The product sends no automatic telemetry to its makers, and there is no opt-in channel ([ADR 0041](0041-no-telemetry-to-the-makers.md)). Product improvement relies on pilots and on what customers choose to send deliberately, for example usage figures they export themselves.

## Related

[Deployment and operations](../architecture/deployment-and-operations.md), [ADR 0015](0015-one-postgresql.md), [ADR 0041](0041-no-telemetry-to-the-makers.md).
