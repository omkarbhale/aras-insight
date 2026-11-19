# 0007. Static analysis of queries for validity and cost

- Status: Accepted
- Scope: MVP

## Context

When a model writes AML ([ADR 0006](0006-model-writes-aml-directly.md)), two kinds of error matter. Validity errors: the query names an item type, property, relationship, list value, or state that does not exist, or uses a value of the wrong type. Aras often ignores unknown fields silently, so a wrong query can return plausible but wrong results, which is a hallucination that looks like data. Cost errors: a query selects everything, has no row limit or paging, nests deeply, or uses broad wildcards, and loads the production Aras server.

## Decision

Every AML query passes through the Query Analyzer, inside the Tool Gateway, before it runs. It checks the query against the discovered schema for validity (unknown item types, properties, relationships, list or state values, value types, and fields Aras would ignore) and for cost (select-all, no row limit or paging, deep nesting, broad wildcards). Diagnostics are returned to the model for a retry. Cost estimates feed the Throttle ([ADR 0030](0030-throttling-in-mvp.md)).

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Run the query and let the model notice empty or odd results | Silent ignoring of unknown fields means the model often cannot notice. Wrong results cost a round trip to Aras and load. |
| Restrict the model to a constrained query form | Rejected in [ADR 0006](0006-model-writes-aml-directly.md). |
| Validate only, leave cost to Aras | The point is to stop expensive queries before they reach production. |

## Consequences

- A class of hallucination (querying fields that do not exist) is caught before execution.
- The Query Analyzer depends on fresh discovered knowledge; a stale schema produces false diagnostics, which is one of the signals that marks learned knowledge uncertain ([ADR 0019](0019-invalidation-by-provenance-and-contradiction.md)).
- The Query Analyzer is also where read-only is enforced: non-read actions are rejected ([ADR 0001](0001-read-only.md)).
- Diagnostics must be written for a model to act on: precise, naming the closest valid alternatives.

## Related

[Query Analyzer](../specs/query-analyzer.md), [Tool Gateway](../specs/tool-gateway.md).
