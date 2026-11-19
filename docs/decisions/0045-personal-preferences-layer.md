# 0045. Preferences are personal and per user only

- Status: Accepted
- Scope: MVP

## Context

Per-user preferences are a knowledge layer apart from shared knowledge ([ADR 0017](0017-three-knowledge-layers.md)). Open points: whether roles share preferences, what a preference may hold, and which meaning wins when a user's personal working definition and a Learned entry cover the same term.

## Decision

- A preference belongs to one user, is visible only to that user, is never used for anyone else and is never promoted into shared knowledge. Role-level preferences are not part of the MVP.
- A preference holds style, format, level of detail, default periods, units, and personal working definitions. It holds no facts the user states about themselves or their cases (for example a product line) and never copied query results. A private fact can still be stated in the conversation.
- For the user's own question, a personal working definition wins over a Learned entry for the same term. The agent looks first at the user's earlier words in the conversation, then the user's preferences, then Learned knowledge, then a suggestion from Discovery. The answer states the preference as an assumption. A contested term still asks.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Per user plus per role, or a role layer | A role preference is shared beyond one person, and nobody can author it without administrator features. |
| The Learned entry wins | Overrides the user's own meaning for their own question. |
| Ask whenever both exist | Turns a personal habit into a recurring question. |
| Preferences that hold facts about the user | Puts data-like values into storage, against "how, never what". |

## Consequences

- Two users may see different answers, and each answer says why.
- Team conventions are shared through Learned entries, not preferences.
- The layer is quality of life ([ADR 0053](0053-core-capabilities.md)).

## Related

[Knowledge and learning](../specs/knowledge-and-learning.md), [clarifying definitions](../skills/clarifying-definitions.md), [persisting knowledge](../skills/persisting-knowledge.md), [ADR 0017](0017-three-knowledge-layers.md), [ADR 0020](0020-organisation-wide-learnings-with-disclosed-assumptions.md).
