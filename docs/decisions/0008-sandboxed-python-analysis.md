# 0008. Sandboxed Python analysis instead of an aggregation engine

- Status: Accepted
- Scope: MVP

## Context

Analytical questions need grouping, counting, durations, rankings, joins across item types, time windows, and reading of text. Models are unreliable at arithmetic over many rows, and loading thousands of rows into a model's context is costly and lossy. Something must do the computation.

## Decision

An Analysis Worker agent runs with instructions and a Python environment in a Sandbox. The Tool Gateway saves AML results and file text as files in the Sandbox; the worker writes and runs Python over them and reports results back. There is no bespoke aggregation engine. Numbers in answers are computed in code.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| A bespoke aggregation engine (a query language for group-by, time series, joins over Aras data) | A large component with its own language, which still would not cover text analysis or unanticipated question shapes. Every missing operator is a product limitation. |
| Let the model compute from rows in its context | Unreliable arithmetic, context limits, and cost. |
| Push aggregation into Aras (server methods or reports) | Requires installing code on the customer's Aras server and admin rights; conflicts with read-only and user-level access. |

## Consequences

- Any computation expressible in Python is available, including text processing over fetched files.
- Model-written code must be contained: the Sandbox has no network and no credentials ([ADR 0009](0009-worker-loop-outside-the-sandbox.md)).
- Data must be fetched into the Sandbox, which means bounded fetches through the Tool Gateway with throttling and row budgets.
- Computed numbers are traceable to the code that produced them and the files it read.

## Related

[Analysis Worker and Sandbox](../specs/analysis-worker-and-sandbox.md), [ADR 0010](0010-main-agent-plus-analysis-workers.md).
