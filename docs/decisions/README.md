# Decision records

Each record below captures one significant architectural or product decision for Aras Insight: the context that forced a choice, what was decided, the alternatives that were considered, why each was rejected, and the consequences the product accepts. They exist so that a reviewer can challenge a decision on its merits, and so that a future change of direction starts from the original reasoning rather than from memory.

All records here have status **Accepted** for the MVP. A record is never edited to reverse its decision: a new record supersedes it and both are kept. Items deliberately left out of the MVP, and what would trigger revisiting them, are in the [roadmap](../product/roadmap.md). How the decisions fit together is in the [architecture overview](../architecture/overview.md).

## Index

### Product stance

| # | Decision |
| --- | --- |
| 0001 | [Read-only](0001-read-only.md) |
| 0002 | [Self-hosted](0002-self-hosted.md) |
| 0003 | [Business end users first](0003-business-end-users-first.md) |
| 0031 | [Sources limited to Aras and what it connects to](0031-sources-limited-to-aras.md) |

### Data access and query safety

| # | Decision |
| --- | --- |
| 0005 | [User-level access through the Aras server only](0005-user-level-access-through-aras-server.md) |
| 0006 | [The model writes AML directly](0006-model-writes-aml-directly.md) |
| 0007 | [Static analysis of queries for validity and cost](0007-static-analysis-for-validity-and-cost.md) |
| 0030 | [Throttling in MVP](0030-throttling-in-mvp.md) |
| 0023 | [No caching in MVP](0023-no-caching.md) |
| 0024 | [No history of Aras data](0024-no-data-history.md) |

### Analysis and agents

| # | Decision |
| --- | --- |
| 0008 | [Sandboxed Python analysis instead of an aggregation engine](0008-sandboxed-python-analysis.md) |
| 0009 | [Sealed Sandbox, with the worker loop outside it](0009-worker-loop-outside-the-sandbox.md) |
| 0010 | [Main Agent plus Analysis Workers](0010-main-agent-plus-analysis-workers.md) |
| 0011 | [Our own thin agent loop instead of a framework](0011-own-thin-agent-loop.md) |
| 0026 | [Three model slots](0026-three-model-slots.md) |
| 0032 | [Saved sessions and live progress](0032-saved-sessions-and-live-progress.md) |

### Retrieval and files

| # | Decision |
| --- | --- |
| 0012 | [No vector index or search-engine reuse in MVP; graph-first retrieval](0012-no-vector-index-graph-first-retrieval.md) |
| 0013 | [A user-confirmed background full-text scan](0013-user-confirmed-background-full-text-scan.md) |
| 0014 | [Extracted text stored per immutable file id](0014-extracted-text-per-immutable-file-id.md) |
| 0025 | [OCR deferred; MVP readable formats](0025-ocr-deferred.md) |

### Knowledge and learning

| # | Decision |
| --- | --- |
| 0004 | [Discovery-only schema](0004-discovery-only-schema.md) |
| 0016 | [Discovery runs as each user](0016-discovery-as-each-user.md) |
| 0017 | [Three knowledge layers](0017-three-knowledge-layers.md) |
| 0018 | [Learned knowledge depends only on configuration](0018-learned-knowledge-depends-only-on-configuration.md) |
| 0019 | [Invalidation by provenance and re-derivation, plus contradiction](0019-invalidation-by-provenance-and-contradiction.md) |
| 0020 | [Organisation-wide learnings with disclosed assumptions](0020-organisation-wide-learnings-with-disclosed-assumptions.md) |
| 0021 | [Learn only from signal-bearing conversations; merge rather than append](0021-learn-from-signals-and-merge.md) |
| 0022 | [Prompt-based how-never-what enforcement in MVP](0022-prompt-based-how-never-what-enforcement.md) |

### Answer quality

| # | Decision |
| --- | --- |
| 0027 | [Causal honesty and coverage reporting](0027-causal-honesty-and-coverage-reporting.md) |
| 0028 | [Ask before expensive analysis; disclose assumptions on cheap work](0028-ask-before-expensive-analysis.md) |
| 0029 | [Three kinds of citation](0029-three-kinds-of-citation.md) |

### Persistence

| # | Decision |
| --- | --- |
| 0015 | [One PostgreSQL database](0015-one-postgresql.md) |

## Record format

Every record has the same sections:

- **Status and scope** - Accepted, and whether it holds for the MVP only or beyond.
- **Context** - the forces that made a decision necessary.
- **Decision** - what was decided, stated so it can be checked against the implementation.
- **Alternatives considered** - each serious alternative, and why it was rejected (or deferred).
- **Consequences** - what the product accepts as a result, including costs.
- **Related** - other records and the documents that implement the decision.

New records take the next free number and the same sections. Numbers are never reused.
