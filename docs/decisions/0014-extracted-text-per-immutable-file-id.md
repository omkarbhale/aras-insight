# 0014. Extracted text stored per immutable file id

- Status: Accepted
- Scope: MVP

## Context

Reading a vault file means downloading it through Aras and extracting its text, which is the slow and load-heavy part of answering from files. The same files are read repeatedly across questions and users. Caching is otherwise not in the MVP ([ADR 0023](0023-no-caching.md)), because cached query results raise permission-scoping problems. In Aras, a file item's content is immutable: new content produces a new file id.

## Decision

The File Service stores extracted text keyed by Aras file id. Because Aras files are immutable, stored text for a file id never goes stale. Access is re-checked through Aras, as the requesting user, on each use; stored text is never served on the strength of another user's earlier access.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| No storage; re-download and re-extract every time | Repeated load on Aras and the vault, and slow background scans. |
| Store text keyed by document item or path | Document items are versioned and mutable; a key by file id is the only one that cannot go stale. |
| Store text with an access-control list copied from Aras | Re-implements Aras permissions and drifts; a live check is simpler and correct. |

## Consequences

- Repeated reads of the same file are cheap after the first, for any user who has access.
- The Database holds a copy of vault text; retention is configurable and the copy is never served without a fresh access check ([security and permissions](../architecture/security-and-permissions.md#data-at-rest)).
- The immutability assumption must be verified per Aras version ([deployment and operations](../architecture/deployment-and-operations.md#aras-version-compatibility-to-verify)).
- This is not a query cache: it stores file contents, not answers or results.

## Related

[File Service](../specs/file-service.md), [ADR 0015](0015-one-postgresql.md).
