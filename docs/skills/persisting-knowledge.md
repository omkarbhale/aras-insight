# Skill: persisting-knowledge

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Learner (always); Main Agent (always, to decide when to call `record_learning`) |
| When | Always |
| Related | [Learner prompt](../prompts/learner.md), [knowledge and learning](../specs/knowledge-and-learning.md), [uncertain-and-contested-knowledge](uncertain-and-contested-knowledge.md), aras-knowledge-mapping (domain skill, see the [catalog](README.md)) |

**Why this skill exists.** Instance knowledge is what makes the product better at one customer's Aras over time without anyone configuring it: what "delayed" means here, which property the teams actually use for a target date, how the markup process is really run. The same mechanism could fill shared knowledge with business data, personal information, noise, and duplicates. This skill is the strong initial instruction that keeps learned knowledge useful and clean: learn how, never what; learn only from conversations that carry a signal; merge rather than append; and put each thing in the right layer.

Scheduled automated checks for data values in learned knowledge are deferred; until they exist, this skill and the [Learner prompt](../prompts/learner.md) are the control.

The text below is the skill as the agent receives it.

---

## Persisting knowledge

Shared knowledge is read by every user of this installation. Write into it only what is true of the installation's configuration and its use, never what is true of its data.

### How, never what

A learned entry describes **how** to find or interpret information. It never records **what** the data contains.

Allowed (how):

- What a term means, as a rule over configuration: "In this organisation, a 'delayed' change order is one whose Released date is later than its target date property."
- Which configuration element a phrase refers to: "'The markup process' refers to the workflow map used by the document review item type."
- How a process is used in practice: "Teams skip the second review activity for minor document changes; the workflow allows it through an optional path."
- Where an explanation is usually recorded: "Root causes for this item type are recorded in the linked analysis document, in a section headed Root Cause."
- Which of two similar properties the organisation relies on.

Forbidden (what):

- Any identifier, name, or number of a business record: part numbers, document numbers, change numbers, item names.
- Any count, total, rate, duration, date, or ranking observed in the data: "most delays come from supplier parts", "there were 40 changes last quarter".
- Any excerpt or paraphrase of file contents or record contents beyond the generic structure of a document type.
- Names or identifying details of people. A role or identity configured in Aras may be named only when the entry is about the process configuration, for example which role a workflow assigns.
- Anything a user said about a specific case, customer, supplier, or project.

If a statement mixes both, keep only the how. If nothing remains, do not persist it.

Test every entry before writing it: if the data changed tomorrow but the configuration did not, would the entry still be true? If not, it contains data, and it must not be written.

### Where it goes

| Destination | What | Visible to |
|---|---|---|
| Learned (instance knowledge) | Meanings of terms, mappings to configuration, how processes are used, where information is recorded | Every user of the installation, with provenance |
| User Preferences | How one user wants answers: format, level of detail, default periods, units, their personal working definitions when they say so | That user only |
| Nowhere | Data values, one-off scope choices for a single question, anything about a specific case | No one |

Discovered knowledge is written by Discovery from the configuration. Do not write into it. If a conversation shows that a Discovered inference is wrong, write a Learned entry that refines it and maps to the same configuration elements.

When a user says how they personally want something ("always show me working days"), it is a preference. When they say what something means for the organisation ("here, 'late' means past the need date"), it is a learning. When it is unclear, treat a definition as a learning and a presentation choice as a preference.

### Learn only from signal-bearing conversations

Persist knowledge only when the conversation contains at least one of these signals:

1. **Correction.** The user said an answer or an interpretation was wrong and gave the right one.
2. **Retried question.** The user rephrased or re-asked because the first answer missed, and a later attempt succeeded. The learning is what made it succeed.
3. **Empty result.** A query returned nothing, and a different mapping then found the records. The learning is the mapping that worked.
4. **Explicit teaching.** The user stated how something works or what a term means, or answered a clarifying question with a definition.

A conversation without a signal produces no learning, however long or interesting it is. Successful answers that used existing knowledge are not new learnings; at most they are confirmations (see below).

### Merge rather than append

Before writing, look for existing entries about the same term or the same configuration element.

- **Same meaning, more detail:** merge into the existing entry. Refine its wording, add any new configuration elements, and add the new conversation to its provenance. Do not create a second entry.
- **Same teacher, changed meaning:** update the entry; the teacher has corrected themselves.
- **Different person, conflicting meaning:** do not overwrite. Mark the entry contested and record both meanings with their provenance.
- **A correction on an answer that cited the entry:** mark the entry uncertain and record the correction as its proposed replacement.
- **Unrelated term that happens to share a word:** write a separate entry and make each description specific enough to tell them apart.

One term or one configuration element should have one entry. Many near-duplicate entries make the index useless.

### Map every entry to configuration

Every learned entry lists the configuration elements it depends on (item types, properties, list values, lifecycle states, workflow activities), each by its stable id and current modification stamp. This is what lets the product mark the entry uncertain when one of them is renamed or removed. An entry that cannot be mapped to any configuration element is usually data or opinion; reconsider it.

### Provenance and confirmation

Every entry records which conversation it came from and who taught it.

- A new entry taught by one person is used organisation-wide at once, but is stated as an explicit assumption whenever it is used, until someone other than the teacher confirms it.
- A different user who is shown the assumption and carries on without objecting confirms it. A user who objects makes it contested.
- After one independent confirmation, the entry is used without announcement.
- In a single-user deployment, the teacher's word is enough.

### Writing the entry

- A one-line description that a model can match against a question: name the term or element and the context.
- A statement of one to five sentences, in plain language, in terms of the configuration.
- No hedging words unless the uncertainty is the point; the state and confidence fields carry uncertainty.

### For the Main Agent: when to call record_learning

Call `record_learning` during the conversation when:

- The user corrects you, teaches you, or answers a clarifying question with a definition.
- A query returned nothing and a different mapping worked.
- The user objects to an assumption that came from a learned entry (this makes the entry contested).
- A different user carried on without objection after you stated an unconfirmed entry as an assumption (this is a confirmation).
- A query built from a learned entry failed Query Analyzer validation (this makes the entry uncertain).
- You verified an uncertain entry against the current schema and it still holds (this returns it to confirmed).

Pass the how, in configuration terms, and the ids of any entries involved. Never pass data values. The Learner decides whether and how to write; you do not need to tell the user each time, but if the user explicitly taught you something, acknowledge briefly that it will be remembered for the organisation.
