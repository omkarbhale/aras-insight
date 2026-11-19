# 0026. Three model slots

- Status: Accepted
- Scope: MVP

## Context

Customers often have internal model infrastructure, want to use the newest available models, and need to switch models without changes to the product. Different work has different needs: the conversation needs a strong model; analysis workers write code and read evidence; background work (batched learning, job summaries) can use a cheaper or internal model.

## Decision

The Model Router exposes three configurable model slots: main (Main Agent), worker (Analysis Workers), background (Learner batches and Background Jobs). Each slot points to an endpoint: OpenAI-compatible endpoints plus native clients where useful. There is no embedding model in the MVP. The product assumes reasonably competent models.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| A single model for everything | Forces the cost of the strongest model on background work, or the weakness of a cheap one on the conversation. |
| Hard-coding one provider | Fails customers with internal endpoints or provider policies. |
| A slot per agent role in detail (planner, coder, reader, ...) | More configuration than customers need; three slots cover the real cost and capability split. |
| An embedding slot | No vector index in MVP ([ADR 0012](0012-no-vector-index-graph-first-retrieval.md)). |

## Consequences

- Plug-and-play endpoint configuration: switching a model is a configuration change.
- Data exposure is governed per slot by the customer's endpoint choice ([security and permissions](../architecture/security-and-permissions.md#data-exposure-to-the-configured-model-endpoints)).
- Prompts and skills must work across capable models, not be tuned to one.

## Related

[Model configuration](../specs/model-configuration.md), [ADR 0011](0011-own-thin-agent-loop.md).
