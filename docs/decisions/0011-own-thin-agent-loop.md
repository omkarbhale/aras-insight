# 0011. Our own thin agent loop instead of a framework

- Status: Accepted
- Scope: MVP

## Context

Agent frameworks offer loops, tool calling, memory, and multi-agent orchestration. This product's hardest requirements are at the tool boundary: every Aras access as the user through one gateway, credentials never in the Sandbox, citations only from a registry, configurable endpoints per slot. Customers also expect to point the product at their own model endpoints and to switch models without code changes.

## Decision

The agent loop is our own thin loop; no agent framework is used. A plain model-client library (OpenAI-compatible, plus native clients where useful) keeps endpoints configurable.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| A general agent framework | Adds abstractions over exactly the parts the product must control precisely (tool execution, context assembly, credentials). Framework upgrades can change behaviour. Endpoint support is tied to the framework's integrations. |
| A provider's hosted agent runtime | Ties the product to one provider and moves the loop outside the customer's deployment. |

## Consequences

- Full control over context assembly, tool dispatch, retries, and persistence of each step (sessions are saved during inference, [ADR 0032](0032-saved-sessions-and-live-progress.md)).
- Any endpoint speaking the OpenAI-compatible protocol, or with a native client, can be used ([ADR 0026](0026-three-model-slots.md)).
- The team owns the loop code, which must stay small.
- The implementation language is not decided by this record.

## Related

[Main Agent](../specs/main-agent.md), [Model configuration](../specs/model-configuration.md).
