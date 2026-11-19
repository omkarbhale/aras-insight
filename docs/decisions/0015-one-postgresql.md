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
| An embedded database per instance | Does not support multiple service instances or the customer's existing database operations. |
| A database other than PostgreSQL | PostgreSQL is widely run in-house, has strong text and JSON support, and is often already operated by the customer's teams. |

## Consequences

- One thing to back up, monitor, and secure.
- Text search over stored extracted text can use the database's own capabilities if needed.
- Background job coordination and progress use the same database (proposed in [deployment and operations](../architecture/deployment-and-operations.md)).
- If scale requires it later, a concern can be split out; the specs keep persistence behind clear interfaces.

## Related

[Persistence](../specs/persistence.md).
