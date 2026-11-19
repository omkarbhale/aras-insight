# 0029. Three kinds of citation

- Status: Accepted
- Scope: MVP

## Context

The main defence against hallucination in synthesis is letting users verify every claim. That requires citations to the exact source, and a guarantee that cited sources were actually read.

## Decision

Answers cite their sources, including learned and discovered knowledge. There are three kinds of citation: an Aras data record (type, id, and a link), a file (with page or section), and a knowledge entry (discovered or learned, with its confidence). The Citation Registry records every record, file passage, and knowledge entry an answer may cite; agents may cite only registered ids. A user correction on an answer that cited a learned entry marks that entry uncertain.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Free-text references written by the model | Can be invented; cannot be resolved reliably to a link. |
| Citing only data records | Leaves file evidence and the agent's own knowledge unverifiable. |

## Consequences

- Every tool that returns evidence must register it and return citation ids.
- Corrections can be traced to the knowledge that caused them.
- The Chat App renders citations as links into Aras and as passage views.

## Related

[Tool Gateway](../specs/tool-gateway.md), [ADR 0019](0019-invalidation-by-provenance-and-contradiction.md).
