# 0047. A final answer check on every answer

- Status: Accepted
- Scope: MVP

## Context

Citations are the first control on hallucination, and the Main Agent checks an answer before showing it. Open points: whether a model-based pass runs on every answer and on which slot, whether the Main Agent may do arithmetic, and whether the worker's causal-wording check blocks a report.

## Decision

- Structural checks run on every answer: markers valid in this conversation, coverage block present, no number from a truncated result.
- A model-based check also runs on every answer and lists uncited factual sentences and causal claims. `main_agent.answer_check_enabled` (default true) switches it off. It runs on the background slot by default, configurable to the main slot, and live checks take priority over batched work there.
- One repair step follows; remaining invalid markers are removed and remaining problems carry an "unverified statement" note.
- The Main Agent does no arithmetic. Every number, percentage and rounded value comes from a tool result or worker report, so numbers can be matched mechanically. Formatting is allowed.
- Every answer reports coverage; a one-record lookup gets one short clause.
- The worker's causal-wording check is advisory (prompt plus a light wording check). The Main Agent's check is the enforced layer. A causal claim stays only when a cited source states the cause.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Structural checks only | Misses uncited sentences and causal wording. |
| Model check only for some answers | Favours latency over assurance before the pilot measures cost. |
| Run the check on the main slot | It may be the costly model. |
| Small arithmetic allowed | The size is undefined and cannot be checked. |
| Exempt one-record lookups from coverage | The clause costs nothing. |
| Block a report on the worker's check | Two enforcement points. |

## Consequences

- Each answer costs an extra model call and some latency.
- The background slot must serve live checks ahead of batched work ([ADR 0051](0051-model-slot-requirements-and-budgets.md)).

## Related

[Main Agent](../specs/main-agent.md), [Model configuration](../specs/model-configuration.md), [ADR 0027](0027-causal-honesty-and-coverage-reporting.md), [ADR 0029](0029-three-kinds-of-citation.md).
