# 0039. Audit and usage records: redacted query text by default, append-only, citation counters, no identity groups

- Status: Accepted
- Scope: MVP

## Context

Audit records show what each person looked for. AML criteria can contain literal values such as a part number, and the text outlives the conversation. The records must hold up in a security review, and usage metrics must survive the retention purge, without recording more than needed.

## Decision

- **Query text.** `audit.store_aml_text` takes `redacted` (default), `full` or `none`. Redacted replaces literal criteria values with placeholders and keeps the query shape and fingerprint; `none` keeps only the fingerprint. Rejected-query events follow it. It governs the audit record only: stored tool calls belong to the private transcript and follow conversation retention.
- **Append-only.** Database permissions enforce it: the service role can insert but not update or delete, and a separate purge role deletes only past retention. The customer provisions both.
- **No identity groups.** Usage is reported per user and in total; the product does not read or record users' Aras identity groups.
- **Citation follow-through.** Each citation row carries an opened count and a first-opened time. There is no per-click event.
- **Usage summaries.** A table of counts only (period, metric, dimensions, value) is refreshed periodically (default every 60 minutes) and kept longer than its source rows, so metrics survive the purge.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Full query text by default | Holds more Aras values for a year; selectable instead. |
| Application-level append-only enforcement | A security review will ask for enforcement in the database. |
| Apply the setting to stored tool calls too | The model's own messages still hold the text. |
| Record group membership at sign-in | Complicates sign-in and schema; customers can join per-user exports to their own lists. |
| A per-click `citation.opened` event | Logs what each person looked at. |
| Compute metrics on request only | Metrics could not survive the purge. |

## Consequences

- Redacted audit is less useful for forensic review unless a customer selects `full`.
- Metrics by role are not available from the product.
- Holding less query text means shortening conversation retention.

## Related

[Audit and progress](../specs/audit-and-progress.md), [Persistence](../specs/persistence.md#audit_events), [Security and permissions](../architecture/security-and-permissions.md#audit).
