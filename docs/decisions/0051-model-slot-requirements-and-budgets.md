# 0051. Model slot requirements, startup checks, clients, and token budgets

- Status: Accepted
- Scope: MVP

## Context

The Model Router has three slots ([ADR 0026](0026-three-model-slots.md)). Customers need to know what each slot's model must support, how a wrong model is caught, which clients ship, and how usage is bounded without the product storing prices.

## Decision

- Tool calling with structured arguments is required on main and worker. Structured JSON output is required on all slots, and streaming on main. The minimum context window is 128,000 tokens for main and worker and 32,000 for background, taken from configuration.
- The startup check tests what each slot needs: a tool call with a structured argument for main and worker, valid JSON output for background.
- Only the OpenAI-compatible client ships. A native client is added when a customer's provider needs one.
- Provider-specific content is dropped when a conversation changes model.
- Each slot has one primary endpoint, with fallbacks used only on failure and no load balancing. Retries (3 attempts), queue wait (60 seconds) and concurrency (16, 16, 2) have defaults and keys.
- The one budget is per turn: `models.turn_token_budget` (default 3,000,000) counts input plus output tokens of the Main Agent and its workers, taken from `model_calls`. There is no daily budget, and background calls are outside it.
- Every Learner and re-derivation step runs on the background slot, with live calls ahead of batched work. Discovery is the only model caller for Discovered entries.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Tool calling on every slot | Excludes the cheap models the background slot is for. |
| Native clients in the MVP | The first customer's provider decides which is worth building. |
| Keep provider-specific content | No other model can use it. |
| Load balancing | A customer can put a balancer in front. |
| Daily budgets | Customers pay their own endpoints; the Throttle bounds Aras load. |
| Main slot for interactive Learner steps | Spends the costly model on background work. |

## Consequences

- A model lacking a capability fails at startup, not mid-conversation.
- A 128,000-token minimum excludes small-context models.

## Related

[Model configuration](../specs/model-configuration.md), [Discovery](../specs/discovery.md), [ADR 0026](0026-three-model-slots.md), [ADR 0047](0047-final-answer-check.md).
