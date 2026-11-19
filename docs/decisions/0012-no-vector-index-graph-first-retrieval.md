# 0012. No vector index or search-engine reuse in MVP; graph-first retrieval

- Status: Accepted
- Scope: MVP

## Context

Many answers are not in metadata rows: the reason for a change is in a long text field, a sign-off comment, or a root-cause report in the vault. A common approach is to vector-index all files, or to reuse the full-text search engine an Aras installation may already have. A vector index of vault content is a second copy of the data that must respect per-user permissions, be kept in sync, and needs an embedding model. An existing search engine's index has its own permission behaviour and coverage, and depends on how each customer configured it.

## Decision

The model is the semantic layer. There is no vector index and no reuse of the installation's search engine in the MVP. Retrieval is graph-first: narrow through the Aras item graph (part to revisions to change records to linked documents), fetch the reached files through Aras as the user, and search within them with code. "Why" sources are tried in a fixed order set by product guidance: long text fields on items, then workflow sign-off comments (a first-class source), then linked files read with code. If vector search is ever used, it is for files inside Aras, not for the agent's own learned material, and the model narrows which files to search first.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Vector index over all vault files | Permission-faithful retrieval is hard; index sync; an embedding model to host or send data to; a large store of business content. The graph usually narrows to a small set of relevant files anyway. |
| Reuse the installation's existing search engine | Behaviour, coverage, and permission handling vary per installation; ties the product to a component not every customer runs. |
| Vector index over learned and discovered knowledge | Not needed: the agent's own knowledge is loaded by index and description, and vector search, if ever used, is reserved for files inside Aras. See the [roadmap](../product/roadmap.md) for the revisit trigger. |

## Consequences

- No embedding model is needed ([ADR 0026](0026-three-model-slots.md)).
- Every file read is a fresh, permission-checked read through Aras.
- Questions with no item anchor need a different path: metadata search, then a user-confirmed full-text scan ([ADR 0013](0013-user-confirmed-background-full-text-scan.md)).
- Per-question file budgets and relevance ordering will be tuned once the system runs.

## Open questions

- The roadmap lists a possible later vector index for learned and discovered knowledge, to be revisited only if the knowledge base outgrows index-based loading (see [roadmap](../product/roadmap.md)). That would revisit the statement above that vector search is for files, not for the agent's own material, and would need a new decision record.

## Related

[File Service](../specs/file-service.md), [question lifecycle: why-question](../architecture/question-lifecycle.md#3-why-question-correlating-revision-history-with-root-cause-documents-and-sign-off-comments).
