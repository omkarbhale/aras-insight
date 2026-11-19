# Prompt: Contested-knowledge clarification wording

| Field | Value |
|---|---|
| Runs on | Model slot **main** (the Main Agent writes the message from this template) |
| Used by | Main Agent, when a question depends on a knowledge entry in the contested state |
| Related | [uncertain-and-contested-knowledge](../skills/uncertain-and-contested-knowledge.md), [clarifying-definitions](../skills/clarifying-definitions.md), [Learner](learner.md) |

## Purpose

Learned knowledge becomes organisation-wide as soon as one person teaches it. When a second person disagrees, the entry becomes contested and keeps both meanings. From then on, whenever a question depends on that entry, the agent does not choose a meaning for the user: it asks which one they intend. This wording keeps that question short, neutral, and easy to answer, so that a real disagreement in the organisation surfaces without slowing the user down more than one reply.

## Design notes

- **Neutral and anonymous.** The message says that colleagues use the term in different ways. It does not name who taught which meaning: conversations are private, and the point is the meaning, not the person.
- **Always asked, whatever the cost.** Unlike an undefined term on a cheap lookup, a contested term is always clarified before answering, because either reading is known to be someone's real meaning.
- **One question.** If the question also has an undefined term, both are combined into this one question.
- **Meanings in business terms.** Each option is stated in the user's language, with the data it relies on, as the knowledge entry describes it.

## Inputs and placeholders

| Placeholder | Source | Description |
|---|---|---|
| `{{term}}` | The contested entry | The term as the user wrote it |
| `{{meaning_a}}`, `{{meaning_b}}` | The contested entry's competing meanings | Each in one sentence, in business terms. More options are listed the same way if the entry holds more than two meanings |
| `{{effect}}` | Main Agent | Optional: one sentence on how the choice changes the answer, when the Main Agent can say so without running the analysis (for example "the second counts from creation, so durations will be longer") |
| `{{other_term}}`, `{{suggested_definition}}` | Main Agent | Optional: an undefined term in the same question, and the definition the Main Agent suggests for it |
| `{{chosen_meaning}}` | The user's reply | The meaning the user chose or described |

## Full text: clarification

````text
Before I answer: colleagues here use "{{term}}" in two different ways.

1. {{meaning_a}}
2. {{meaning_b}}

{{effect}}

Which do you mean? You can reply 1 or 2, or describe what you mean.
````

When the question also has an undefined term, the Main Agent appends one line:

````text
Also, for "{{other_term}}", I suggest {{suggested_definition}}; tell me if you'd rather use something else.
````

## Full text: after the user answers

The Main Agent proceeds and states the choice once in the answer's assumptions:

````text
Using "{{term}}" to mean {{chosen_meaning}}, as you chose.
````

If the user describes a third meaning, the Main Agent uses it, states it the same way, and reports it with `record_learning`; the Learner adds it to the contested entry.

The Main Agent reports the user's choice with `record_learning` so the Learner can record it. Choosing does not resolve the disagreement for the organisation; the entry stays contested until a later teaching or a configuration change settles it.

## Open questions

1. Proposed: who taught each meaning is never shown to other users. Whether an administrator-facing view of contested entries should show provenance is a later question and belongs with developer and administrator features, which are deferred.
2. Proposed: a user's own choice between contested meanings may be remembered as their preference, so the same user is not asked again (see the open questions in the [Learner](learner.md)).
