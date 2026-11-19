# 0026. Three model slots

- Status: Accepted
- Scope: MVP

## Context

Customers often have internal model infrastructure, want to use the newest available models, and need to switch models without changes to the product. Different work has different needs: the conversation needs a strong model; analysis workers write code and read evidence; background work (the Learner's steps, evaluation of disclosed assumptions, Discovery's model-inferred entries, and the check of each answer before it is shown) can use a cheaper or internal model.

## Decision

The Model Router exposes three configurable model slots: main (Main Agent), worker (Analysis Workers), background (the Learner's steps, evaluation of disclosed assumptions, Discovery's model-inferred entries, and by default the check of each answer, which can be switched to the main slot). Each slot points to an endpoint: OpenAI-compatible endpoints; the MVP ships no native client, and one is added when a customer's provider needs it. There is no embedding model in the MVP. The product assumes reasonably competent models.

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
- Live answer checks take priority over batched work on the background slot ([ADR 0047](0047-final-answer-check.md), [ADR 0051](0051-model-slot-requirements-and-budgets.md)).

## Related

[Model configuration](../specs/model-configuration.md), [ADR 0011](0011-own-thin-agent-loop.md), [ADR 0047](0047-final-answer-check.md), [ADR 0051](0051-model-slot-requirements-and-budgets.md).
