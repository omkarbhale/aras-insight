# 0053. Core capabilities: parallel workers, audit records, assumption statements, live learning, and preferences

- Status: Accepted
- Scope: MVP

## Context

The product sorts every capability into core (not fit to pilot without it), quality of life (expected in the first release, but the product still works if one slips) and not supported. Five capabilities were in the design without a stated category.

## Decision

Core:

- Several Analysis Workers in parallel for multi-part questions. The worker limit is configurable, so setting it to one gives sequential behaviour if it must slip.
- Audit records of every query run on a user's behalf, and usage records of model calls.
- Stating an unconfirmed Learned entry as an assumption, and asking which meaning the user intends when an entry is contested.
- Live learning during a conversation, alongside batched learning.

Quality of life:

- The per-user preferences layer.

The [scope](../product/scope.md) document lists them accordingly.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Parallel workers as quality of life | The shared question budget, one Sandbox per conversation and progress per worker assume them. |
| Audit and usage records as quality of life | Without them there is no security-review evidence and no ROI measurement. |
| Assumption statements as quality of life | Learning without disclosed assumptions was rejected ([ADR 0020](0020-organisation-wide-learnings-with-disclosed-assumptions.md)), and risk mitigations rely on them. |
| Batched learning alone as core | It cannot apply a definition within the same conversation ([ADR 0021](0021-learn-from-signals-and-merge.md)). |
| Preferences as core | The layer least tied to the safety design; the product works without it. |

## Consequences

- The four core capabilities must work before a pilot starts.
- Preferences may slip without blocking it.

## Related

[Scope](../product/scope.md), [roadmap](../product/roadmap.md), [ADR 0010](0010-main-agent-plus-analysis-workers.md), [ADR 0039](0039-audit-and-usage-records.md), [ADR 0042](0042-assumption-disclosure-and-independent-confirmation.md), [ADR 0043](0043-contested-knowledge.md), [ADR 0045](0045-personal-preferences-layer.md).
