# 0044. The Learner's operations, signals, and writes to Discovered entries

- Status: Accepted
- Scope: MVP

## Context

The specification gave the Learner jobs its prompt forbade (retiring entries, ending contests, merging duplicates), and the documents disagreed about Discovered entries, deleted configuration elements and learning signals.

## Decision

- The Learner has no tools. It outputs operations that the Knowledge Service validates and applies, and the service computes confidence and applies the single-user rule. "Mark uncertain" takes an optional `reason` and `replacement`, "record confirmation" takes a `basis`, and the schema is defined once in the knowledge specification.
- Two operations are added: `retire` (merged, superseded or corrected, with a merged-into pointer) and `resolve` (concession or split). Duplicate merges use them.
- Discovered entries are read-only for the Learner. A correction becomes a Learned entry that refines them.
- A Learned entry whose element is renamed or removed becomes uncertain; retirement on source deletion applies to Discovered entries only. A Learned entry is retired only after failed verification (reason corrected).
- Retirement reasons are source_deleted, merged, superseded, corrected, withdrawn (the teacher withdrew an entry nobody else confirmed) and product_upgrade.
- Learning needs one of the four signals. Confirmations and objections are evaluated live from the next message, outside that gate; the batched variant records none.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| The Learner calls write tools | Bypasses the service's validation. |
| No retire or resolve operation | Contests never end; duplicates never merge. |
| The service merges, the Learner only splits | Two mechanisms for one judgement. |
| The Learner edits Discovered entries | Their text is regenerated, and the agent would question users about its own inferences. |
| A removed element retires Learned entries | The entry may still be right once verified. |
| "Carried on" as a fifth signal | The decided signals stay four. |

## Consequences

- One judge, the Learner, ends contests and merges duplicates.
- A wrong Discovered inference lives beside its Learned refinement.

## Related

[Knowledge and learning](../specs/knowledge-and-learning.md), [Learner prompt](../prompts/learner.md), [ADR 0021](0021-learn-from-signals-and-merge.md), [ADR 0043](0043-contested-knowledge.md).
