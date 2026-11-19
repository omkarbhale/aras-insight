# 0001. Read-only

- Status: Accepted
- Scope: MVP and foreseeable versions

## Context

Aras Innovator holds the authoritative record of a company's products: parts, documents, changes, and their approvals. Business users want to ask questions of that record. Any tool that can also change it must answer harder questions: who approved the change, how it is audited, how a model's mistake is undone, and whether the tool now needs edit permissions that a security team must review. Those questions slow adoption and raise the stakes of every model error.

## Decision

Aras Insight is read-only. It never creates, edits, deletes, promotes, locks, claims, or otherwise changes anything in Aras. The Tool Gateway admits only read actions, the Query Analyzer rejects any non-read action before execution, and no agent tool exposes a write.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Read and write, with user approval of each write | Moves the product from "assistant that answers" to "actor in the change process", with approval, audit, and rollback requirements that would dominate the MVP. The value demonstrated by answering questions does not depend on writing. |
| Writes limited to low-risk actions (for example comments) | Still requires a write path, its permission review, and a definition of "low risk" that differs per customer. |

## Consequences

- A model error can produce a wrong answer, never a wrong change. Wrong answers are mitigated by citations and coverage.
- Security review is simpler: the product needs no more than the user's existing read access.
- Requests such as "create a change request for this" are out of scope. Write actions are listed as deferred, "later, if ever" (see [roadmap](../product/roadmap.md)).
- Defence in depth: even if the gateway's read-only check were bypassed, the user's own Aras permissions still apply.
- The roadmap names what would prompt reconsidering writes: a strong, specific customer need, together with an approval and audit design acceptable to the customer's Aras owners. That is an indicative condition, not a commitment, and a change of direction would need a new record that supersedes this one.

## Related

[ADR 0005](0005-user-level-access-through-aras-server.md), [ADR 0007](0007-static-analysis-for-validity-and-cost.md), [security and permissions](../architecture/security-and-permissions.md).
