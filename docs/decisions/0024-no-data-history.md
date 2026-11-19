# 0024. No history of Aras data

- Status: Accepted
- Scope: MVP, at least for now

## Context

Some trend questions need history Aras itself does not keep (for example, values of a property over time where Aras keeps only the current one). The product could record changes from installation day onward to make such trends possible.

## Decision

The product keeps no history of Aras data, at least for now. Trends come from the data the agent can access in Aras: typically item generations, workflow and lifecycle run histories, and timestamps, as far as the installation retains them (to verify per version).

## Alternatives considered

| Alternative | Why rejected for now |
| --- | --- |
| Record change history from installation day | A store of business data, with permission, retention, and storage implications, and a privileged or per-user collector to populate it. Conflicts with the simplicity of user-level access. Deferred (see [roadmap](../product/roadmap.md)). |

## Consequences

- Trend questions are limited to what Aras retains; answers say so in coverage when history is missing.
- The Database holds no business-data history; its only copies of Aras content are extracted file text, the excerpts and record names kept with citations, the text of queries (redacted by default in audit records, and in full in the stored tool calls of transcripts), short passage snippets kept for background jobs, progress lines of every conversation, and what appears in transcripts. Each has its own retention (see [Persistence](../specs/persistence.md) and [ADR 0039](0039-audit-and-usage-records.md)).
- The roadmap names what would prompt reconsidering: users regularly ask for trends over data that Aras overwrites or does not keep, and a permission-faithful storage design exists. That is an indicative condition, not a commitment, and would need a new record.

## Related

[ADR 0018](0018-learned-knowledge-depends-only-on-configuration.md), [ADR 0023](0023-no-caching.md), [ADR 0039](0039-audit-and-usage-records.md).
