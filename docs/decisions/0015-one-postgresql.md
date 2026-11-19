# 0015. One PostgreSQL database

- Status: Accepted
- Scope: MVP

## Context

The product stores sessions and transcripts, three knowledge layers, extracted file text, job state, and audit and usage records. Each could justify a specialised store (a document store, a search index, a queue, a cache). Every additional store is another thing a self-hosted customer must install, secure, back up, and upgrade.

## Decision

One database only: a single PostgreSQL instance holds sessions and transcripts, knowledge, extracted file text, job state, and audit and usage records.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Separate stores per concern (for example a cache, a queue, a vector store, a document store) | Operational burden on the customer; none is needed at MVP scale. Several are excluded by other decisions (no cache, no vector index). |
| An embedded database per instance | Ties state to one host, so a standby instance cannot take over, and bypasses the customer's existing database operations. |
| A database other than PostgreSQL | PostgreSQL is widely run in-house, has strong text and JSON support, and is often already operated by the customer's teams. |

## Consequences

- One thing to back up, monitor, and secure.
- Stored extracted text is searched by code in the MVP, not by a database index; PostgreSQL's text-search features remain available if a later decision introduces one. The roadmap's revisit condition is repeat scans dominating Aras load, and any index would keep the per-use access check on every hit.
- Background job scheduling, claiming, retry and progress use the same database; there is no message broker or queue product ([ADR 0037](0037-single-instance-mvp-and-optional-gateway-process.md)).
- The MVP runs one active service instance, so the database holds no ownership or lease data for turns and jobs.
- If scale requires it later, a concern can be split out; the specs keep persistence behind clear interfaces.

## Related

[Persistence](../specs/persistence.md), [ADR 0037](0037-single-instance-mvp-and-optional-gateway-process.md).
