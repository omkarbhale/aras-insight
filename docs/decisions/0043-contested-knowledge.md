# 0043. Contested knowledge: ask once per conversation, and how a contest ends

- Status: Accepted
- Scope: MVP

## Context

When users disagree about an entry's meaning it becomes contested ([ADR 0019](0019-invalidation-by-provenance-and-contradiction.md)), and the agent asks which meaning the user intends. Several details had to be fixed.

## Decision

- The agent asks which meaning the user intends, whatever the cost of the work, once per conversation. The reply stays in the transcript, no preference is stored, and the entry stays contested for everyone.
- A contest ends only by concession (one side gives way; the entry returns to confirmed with the agreed content) or split (both meanings hold in different contexts; the entry is retired as superseded and replaced by two scoped entries), judged by the Learner in a later conversation. A user's choice or a configuration change does not end it.
- If a referenced element is renamed or removed, the entry stays contested, with the stale reference flagged in its reason.
- The user's choice is a confirmation carrying a `contradiction_id` that names the meaning.
- Verification questions and contested clarifications are disclosed items on the turn. The Learner evaluates the next message for them, including a third meaning the user describes, so the Main Agent makes no state-changing calls.
- The clarification takes the number of meanings as a parameter; the Chat App shows one button per meaning plus free text.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Ask every time | Repeats a question the transcript answers. |
| Store the choice as a preference | Goes stale when the entry is split. |
| End by configuration change or administrator action | No usable criterion; administrator features are out. |
| Make it uncertain, or let removal decide | Hides the contest, or settles it while users disagree. |
| Main Agent state-changing calls | One Learner rule serves live and batched learning. |

## Consequences

- Each new conversation asks again.
- A contest lasts until the Learner judges a concession or a split.

## Related

[Knowledge and learning](../specs/knowledge-and-learning.md), [clarification prompt](../prompts/contested-knowledge-clarification.md), [ADR 0042](0042-assumption-disclosure-and-independent-confirmation.md), [ADR 0044](0044-learner-operations-and-signals.md).
