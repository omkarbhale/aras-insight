# 0006. The model writes AML directly

- Status: Accepted
- Scope: MVP

## Context

To read Aras, something must produce AML, Aras's XML query language. Options range from letting the model write AML, to having the model fill in a constrained query form our code compiles, to having it write SQL that is translated. The product assumes a reasonably competent model. AML is expressive (nested relationships, conditions, paging, select lists) and documented, and capable models can write it, especially with guidance and feedback.

## Decision

The model writes AML directly. Our code does not generate AML on the model's behalf; it analyses what the model wrote ([ADR 0007](0007-static-analysis-for-validity-and-cost.md)) and returns diagnostics for a retry. A shipped product skill teaches how to write AML well (see [skills](../skills/README.md)). An SQL path remains open only if it preserves Aras permissions, for example through translation into AML.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Structured query form compiled by our code (the model fills fields; our code emits AML) | Every AML capability the form lacks becomes a product limitation. The form becomes a second query language to design, document, and keep in step with Aras. A competent model can write the real language, and static analysis catches its mistakes. |
| SQL-to-AML translation (the model writes SQL, which is translated) | Models know SQL well, but Aras's data model is not relational in the way SQL assumes (generations, relationships as items, server events). A faithful translator is a significant project, and anything it cannot translate is lost. Kept open as a possibility only if it preserves permissions. |
| SQL against the database | Bypasses Aras permissions ([ADR 0005](0005-user-level-access-through-aras-server.md)). |

## Consequences

- No expressiveness ceiling beyond AML itself.
- Model mistakes are expected and handled by static analysis and retry, not by restricting the language.
- Quality depends on the model and on the AML product skill; the product targets capable models ([ADR 0026](0026-three-model-slots.md)).
- Every query is an auditable artefact in the language Aras administrators already know.
- The SQL path is tied to the roadmap item for an analytics path that does not load production. It is reconsidered under that item's condition: production load nears agreed limits despite throttling, or analytical questions regularly hit row budgets, and a way to preserve user-level access is demonstrated (see [roadmap](../product/roadmap.md)).

## Related

[Query Analyzer](../specs/query-analyzer.md), [tool catalog](../specs/tool-catalog.md).
