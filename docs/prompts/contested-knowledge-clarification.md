# Prompt: Contested-knowledge clarification wording

| Field | Value |
|---|---|
| Runs on | Model slot **main** (the Main Agent writes the message from this template) |
| Used by | Main Agent, when a question depends on a knowledge entry in the contested state |
| Rendered with | One button per competing meaning, plus a free-text reply, in the [Chat App](../specs/chat-app.md) |
| Related | [uncertain-and-contested-knowledge](../skills/uncertain-and-contested-knowledge.md), [clarifying-definitions](../skills/clarifying-definitions.md), [Learner](learner.md), [ADR 0043](../decisions/0043-contested-knowledge.md) |

## Purpose

Learned knowledge becomes organisation-wide as soon as one person teaches it. When a second person disagrees, the entry becomes contested and keeps both meanings. From then on, whenever a question depends on that entry, the agent does not choose a meaning for the user: it asks which one they intend, once per conversation. This wording keeps that question short, neutral, and easy to answer, so that a real disagreement in the organisation surfaces without slowing the user down more than one reply.

## Design notes

- **Neutral and anonymous.** The message says that colleagues use the term in different ways. It never names who taught which meaning: conversations are private, and the point is the meaning, not the person. A deployment can turn on `knowledge.show_teacher_names` to show teachers' names where provenance is shown; the template itself still names no one, and the MVP has no administrator view of contested entries.
- **Always asked, whatever the cost, once per conversation.** Unlike an undefined term on a cheap lookup, a contested term is always clarified before answering, because either reading is known to be someone's real meaning. The question is asked once per conversation: the user's reply stays in the transcript, so the agent uses the meaning they chose for the rest of the conversation. A new conversation asks again. The choice is not stored as a personal preference.
- **One question.** If the question also has an undefined term, both are combined into this one question.
- **Meanings in business terms.** Each option is stated in the user's language, with the data it relies on, as the knowledge entry describes it.
- **However many meanings there are.** The message lists every competing meaning the entry holds, whether two or more. The Chat App shows one button per meaning, and the user may instead describe a meaning in words.

## Inputs and placeholders

| Placeholder | Source | Description |
|---|---|---|
| `{{term}}` | The contested entry | The term as the user wrote it |
| `{{meanings}}` | The contested entry's competing meanings | One numbered line per meaning, each in one sentence, in business terms |
| `{{effect}}` | Main Agent | Optional: one sentence on how the choice changes the answer, when the Main Agent can say so without running the analysis (for example "the second counts from creation, so durations will be longer"). When empty, its line is left out |
| `{{stale_note}}` | The contested entry | Optional: one sentence when a configuration element one of the meanings relies on has been renamed or removed (for example "One of these relies on a property that no longer exists in the configuration."). When empty, its line is left out |
| `{{other_term}}`, `{{suggested_definition}}` | Main Agent | Optional: an undefined term in the same question, and the definition the Main Agent suggests for it. When there is no other term, the appended line is not written |
| `{{chosen_meaning}}` | The user's reply | The meaning the user chose or described |

## Full text: clarification

The Main Agent writes this message, leaving out any line whose placeholder is empty. The message is never filled with "none".

````text
Before I answer: colleagues here use "{{term}}" in different ways.

{{meanings}}

{{effect}}

{{stale_note}}

Which do you mean? Choose one, or describe what you mean in your own words.
````

When the question also has an undefined term, the Main Agent appends one line:

````text
Also, for "{{other_term}}", I suggest {{suggested_definition}}; tell me if you'd rather use something else.
````

## Full text: after the user answers

The Main Agent proceeds and states the choice once, in the closing assumptions of the answer:

````text
Using "{{term}}" to mean {{chosen_meaning}}, as you chose.
````

If the user describes a further meaning, the Main Agent uses it and states it the same way.

The Main Agent does not report the user's choice with `record_learning`. The product evaluates the user's reply with the evaluation variant of the [Learner](learner.md) and records the choice as a confirmation of that meaning on the contested entry, and any further meaning the user describes as an added meaning (see [Main Agent](../specs/main-agent.md#57-contested-knowledge)). Choosing does not resolve the disagreement for the organisation. The entry stays contested until the Learner resolves it in a later conversation: one side concedes and the entry returns to confirmed with the agreed meaning, or both meanings are legitimate in different contexts and the entry is split into two scoped entries. A configuration change does not end a contest either: if an element a meaning relies on is renamed or removed, the entry stays contested and the stale reference is noted in its reason, which the clarification question then mentions.
