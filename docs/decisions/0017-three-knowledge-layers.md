# 0017. Three knowledge layers

- Status: Accepted
- Scope: MVP

## Context

The agent needs three different kinds of knowledge, with different owners, lifetimes, and sharing rules: how to work (how to write AML, how to use the tools and files, what to persist), what this installation is and means (its configuration and the business meaning of its terms), and how this particular user likes to work.

## Decision

Knowledge is split into three layers, served by the Knowledge Service and loaded by index and description, not by vectors:

1. **Product Skills** - shipped with the product and versioned with it: how to write AML, how to use the tools and files, what knowledge to persist and where, live or batched over transcripts.
2. **Instance Knowledge** - about this Aras installation: **Discovered** (from configuration, including model-inferred descriptions such as "this process starts from a problem report, which fills fields A, B, C") and **Learned** (meaning formed in use, such as what the business means by "delayed"). Shared organisation-wide.
3. **User Preferences** - per user or role, in their own personal layer, separate from shared knowledge.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| One undifferentiated memory | Mixes shipped guidance (which must change only with product versions) with instance facts (which change with configuration) and personal preferences (which must not be shared). |
| No learned layer; discovery only | Configuration does not say what the business means by "delayed" or "released"; users would repeat definitions forever. |
| Vector-indexed knowledge | Not needed: index and description loading is enough at MVP volumes (see [ADR 0012](0012-no-vector-index-graph-first-retrieval.md)). |

## Consequences

- Each layer has its own lifecycle: skills change with upgrades; discovered knowledge with configuration; learned knowledge by learning and invalidation; preferences by the user.
- Shared learned knowledge must contain no data values ([ADR 0018](0018-learned-knowledge-depends-only-on-configuration.md)).
- Every layer is citable: answers cite discovered and learned entries with their confidence ([ADR 0029](0029-three-kinds-of-citation.md)).
- The catalog of shipped skills is in [docs/skills/README.md](../skills/README.md).

## Related

[Knowledge and learning](../specs/knowledge-and-learning.md).
