# 0052. ROI metrics come from the product's own records

- Status: Accepted
- Scope: MVP and beyond

## Context

The business case lists return-on-investment metrics, and the audit and progress specification defines usage metrics. The two differed, although the business case says each metric is computed from records the product already keeps. A customer running a pilot must obtain the figures without collecting data by hand, and estimated time saved needs a manual baseline that differs per customer.

## Decision

- The audit and progress specification is the single source of metric definitions. The value and ROI document refers to it.
- Metrics the business case wants but the product did not define (repeat use, analysis depth, clarification rate, cost per answer) are added there as derived queries over tables the product already keeps. The correction-rate source is stated once: submitted corrections plus the Learner's correction signals.
- Cost per answer is tokens from `model_calls` times the customer's own rates, so the product stores no price.
- Estimated time saved is the answers in each answer-mix class (direct lookup, analysis, background-job answer) times a minutes value per class that the customer configures (`metrics.time_saved_minutes`), filled in from the customer's baseline timing. It needs one customer input and no per-answer collection.
- Figures come from the usage report and database queries, with no hand-collected data apart from the customer's baseline.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| The business case defines the metrics | Two owners drift, and records would grow to match the list. |
| Shrink the ROI list to what is recorded | Loses repeat use, analysis depth and cost per answer. |
| Manual effort per question type minus measured time | Needs a baseline per question type and per-answer collection. |
| Configured minutes with no baseline | An arbitrary figure. |

## Consequences

- Time saved is only as good as the customer's baseline.
- Derived metrics need operator queries, not a dashboard.
- Nothing is sent to the makers of the product ([ADR 0041](0041-no-telemetry-to-the-makers.md)).

## Related

[Audit and progress](../specs/audit-and-progress.md), [value and ROI](../business/value-and-roi.md), [ADR 0039](0039-audit-and-usage-records.md), [ADR 0040](0040-operator-reports-without-administrator-endpoints.md).
