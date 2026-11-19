# Skill: persisting-knowledge

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Learner (always); Main Agent (always, to decide when to call `record_learning`) |
| When | Always |
| Related | [Learner prompt](../prompts/learner.md), [knowledge and learning](../specs/knowledge-and-learning.md), [uncertain-and-contested-knowledge](uncertain-and-contested-knowledge.md), aras-knowledge-mapping (domain skill, see the [catalog](README.md)) |

**Why this skill exists.** Instance knowledge is what makes the product better at one customer's Aras over time without anyone configuring it: what "delayed" means here, which property the teams actually use for a target date, how the design review process is really run. The same mechanism could fill shared knowledge with business data, personal information, noise, and duplicates. This skill is the strong initial instruction that keeps learned knowledge useful and clean: learn how, never what; learn only from conversations that carry a signal; merge rather than append; and put each thing in the right layer.

Neither the tools nor the Knowledge Service scan entries for data values, and scheduled automated checks for them are deferred; until they exist, this skill and the [Learner prompt](../prompts/learner.md) are the only control, so every proposal must pass the test below before it is written.

The text below is the skill as the agent receives it.

---

## Persisting knowledge

Shared knowledge is read by every user of this installation. Write into it only what is true of the installation's configuration and its use, never what is true of its data.

### How, never what

A learned entry describes **how** to find or interpret information. It never records **what** the data contains.

Allowed (how):

- What a term means, as a rule over configuration: "In this organisation, a 'delayed' change order is one whose Released date is later than its target date property."
- Which configuration element a phrase refers to: "'The design review process' refers to the workflow map used by the document review item type."
- How a process is run, stated in terms of its configuration: "The second review activity is optional for document change types; the workflow has a path that skips it."
- Where an explanation is usually recorded: "Root causes for this item type are recorded in the linked analysis document, in a section headed Root Cause."
- Which of two similar properties the organisation relies on.

Forbidden (what):

- Any identifier, name, or number of a business record: part numbers, document numbers, change numbers, item names.
- Any count, total, rate, duration, date, or ranking observed in the data: "most delays come from supplier parts", "there were 40 changes last quarter".
- Any excerpt or paraphrase of file contents or record contents beyond the generic structure of a document type.
- Names or identifying details of people. A role or identity configured in Aras may be named only when the entry is about the process configuration, for example which role a workflow assigns.
- Anything a user said about a specific case, customer, supplier, or project, and facts a user states about themselves (their own product line, team, or projects). Use such facts in the conversation; do not store them.

If a statement mixes both, keep only the how. If nothing remains, do not persist it.

Test every entry before writing it: if the data changed tomorrow but the configuration did not, would the entry still be true? If not, it contains data, and it must not be written.

### Where it goes

| Destination | What | Visible to |
|---|---|---|
| Learned (instance knowledge) | Meanings of terms, mappings to configuration, how processes are used, where information is recorded | Every user of the installation, with provenance |
| User Preferences | How one user wants answers: format, level of detail, default periods, units, and their personal working definitions when they say so. Nothing else | That user only |
| Nowhere | Data values, facts a user states about themselves, one-off scope choices for a single question, anything about a specific case | No one |

Discovered knowledge is written by Discovery from the configuration. It is for reading: do not write into it, and do not mark it uncertain or contested. If a conversation shows that a Discovered inference is wrong, write a Learned entry that refines it and maps to the same configuration elements.

When a user says how they personally want something ("always show me working days"), it is a preference. When they say what a term means only for themselves ("when I say late I mean past the need date"), it is a personal working definition, which is also a preference. When they say what something means for the organisation ("here, 'late' means past the need date"), it is a learning. When it is unclear, treat a definition as a learning and a presentation choice as a preference.

A personal working definition and an organisation-wide entry for the same term can both exist. The user's own definition wins for that user's own questions, and the answer states it as an assumption; a contested term still asks (clarifying-definitions).

### Learn only from signal-bearing conversations

Persist knowledge only when the conversation contains at least one of these signals:

1. **Correction.** The user said an answer or an interpretation was wrong and gave the right one.
2. **Retried question.** The user rephrased or re-asked because the first answer missed, and a later attempt succeeded. The learning is what made it succeed.
3. **Empty result.** A query returned nothing, and a different mapping then found the records. The learning is the mapping that worked. The Tool Gateway records every empty result itself; the learning is written only when a different mapping fixed it.
4. **Explicit teaching.** The user stated how something works or what a term means, or answered a clarifying question with a definition.

A conversation without a signal produces no learning, however long or interesting it is. Successful answers that used existing knowledge are not new learnings. Confirming or objecting to a disclosed assumption is not a fifth signal: the product evaluates the user's next message on a separate path, right after the assumption is disclosed (see Provenance and confirmation below), and a transcript read later never records a confirmation.

### Merge rather than append

Before writing, look for existing entries about the same term or the same configuration element.

- **Same meaning, more detail:** merge into the existing entry. Refine its wording, add any new configuration elements, and add the new conversation to its provenance. Do not create a second entry.
- **Same teacher, changed meaning:** update the entry; the teacher has corrected themselves.
- **Different person, conflicting meaning:** do not overwrite. Mark the entry contested and record both meanings with their provenance.
- **A correction on an answer that cited the entry:** mark the entry uncertain and record the correction as its proposed replacement.
- **Unrelated term that happens to share a word:** write a separate entry and make each description specific enough to tell them apart.
- **Duplicates that already exist:** two entries about the same term or element are merged into one, and the other is retired as merged, with a pointer to the entry that stays.

One term or one configuration element should have one entry. Many near-duplicate entries make the index useless.

### Ending entries and contests

Retiring a Learned entry, resolving a contest, splitting an entry, and merging duplicates are the Learner's judgement. The Main Agent never does them.

- **Retire** an entry with one of three reasons, naming the entry that replaces it where there is one: *merged* (a duplicate folded into another entry), *superseded* (replaced by a split, or by a new entry), or *corrected* (a correction left nothing valid, or the user said, when the entry was verified with them, that it no longer applies). A renamed or removed configuration element never retires a Learned entry; it makes the entry uncertain. Deleting the source configuration retires Discovered entries, which are not yours to change.
- **Resolve** a contested entry in one of two ways, judged from a later conversation: *withdrawal* (one side concedes; the entry returns to confirmed with the agreed content) or *split* (both meanings are legitimate in different contexts; the entry is retired as superseded and replaced by two entries with distinct titles and scopes, each keeping the disagreement as provenance). Nothing else ends a contest. A user's own choice between the meanings does not end it.

### Map every entry to configuration

Every organisation-wide learned entry lists the configuration elements it depends on (item types, properties, list values, lifecycle states, workflow activities). The proposal names each element by its `element_kind` and `name`, plus its `parent` where it has one (for example the item type that owns a property), and `record_learning` requires at least one for an organisation-wide entry. The Learner resolves each element against the discovered schema and records its stable id and current modification stamp; the proposer does not pass ids or stamps. This is what lets the product mark the entry uncertain when one of them is renamed or removed. An organisation-wide entry that cannot be mapped to any configuration element is usually data or opinion; reconsider it.

A personal preference needs no configuration element: a presentation habit ("show durations in working days") refers to none. A personal working definition names the elements it refers to, as any definition does.

### Provenance and confirmation

Every entry records which conversation it came from and who taught it.

- A new entry taught by one person is used organisation-wide at once. Every user other than the teacher is told that it is an explicit assumption whenever it is used, until someone other than the teacher confirms it. The teacher is not told: they know their own wording.
- A different user who is shown the assumption and carries on without objecting confirms it. A user who objects makes it contested.
- After one independent confirmation, the entry is used without announcement.
- In a single-user deployment, the teacher's word is enough: the deployment setting makes new entries independently confirmed from the start, so you are not told to announce them. If the deployment later gains users, the product makes entries whose only confirmation came from that rule unconfirmed again, and they are announced to users other than the teacher like any other unconfirmed entry.

The product, not the Main Agent, judges whether a reply confirms or objects: the Learner evaluates the user's next message against the assumptions the answer disclosed. There are no accept or reject controls for the user to press; the user's reply in words is the only input, and leaving the conversation confirms nothing.

### Writing the entry

- A one-line description that a model can match against a question: name the term or element and the context. Use the words users use for the term; the Learner derives the entry's aliases (other words for the same thing) from your wording, so you do not supply them.
- A statement of one to five sentences, in plain language, in terms of the configuration. (This limit is for Learned entries. Discovered entries, which outline whole processes, may run to six sentences.)
- No hedging words unless the uncertainty is the point; the state and confidence fields carry uncertainty.

### For the Main Agent: when to call record_learning

`record_learning` carries one thing: a proposal for a learned entry or a user preference. Every call names the signal that justifies it, one of `correction`, `retried_question`, `empty_result`, or `explicit_teaching`; a call without one is rejected.

Call `record_learning` during the conversation when:

- The user corrects you (`correction`).
- The user rephrased or re-asked because your first answer missed, and a later attempt succeeded (`retried_question`).
- A query returned nothing and a different mapping worked (`empty_result`). The Tool Gateway has already recorded the empty result; your call adds the learning once the different mapping has found the records.
- The user teaches you how something works, or answers a clarifying question with a definition (`explicit_teaching`).

Do not use it to report anything else. The product handles these without a call from you:

- The Learner evaluates the user's next message against everything your answer put to the user: the assumptions you disclosed, the questions you asked to verify an uncertain entry, and the clarifications you asked about contested entries. Carrying on without objecting confirms an assumption, and an objection makes the entry contested. Agreement to a verification question returns the uncertain entry to confirmed. The user's choice between contested meanings, and any third meaning they describe, is recorded on the entry.
- When a query built from a learned entry fails Query Analyzer validation, the Tool Gateway reports it and the Knowledge Service marks the entry uncertain.
- A worker's candidate learnings, and a worker's note that an entry looks wrong, are not signals. They stay in the stored report.
- Retiring, resolving, splitting, and merging entries are the Learner's work.

Do not use `record_learning` to report that you checked an uncertain entry against the current schema; state the entry as an assumption in your answer instead.

Pass the how, in configuration terms: a title, a one-line description, the body, and the configuration elements it refers to, each by element kind, name, and parent where it has one. Pass `scope` personal for a user preference. Do not pass aliases, ids, or modification stamps; the Learner derives the aliases and resolves the elements. If the proposal corrects an existing entry, pass that entry's id as `supersedes`. Never pass data values.

The Learner decides whether and how to write, and the call's result tells you what happened. If a call comes back rejected (for example for no signal, unresolved configuration references, or malformed fields), you may correct it and send it once more. Do not say anything about remembering until you have the result. If the user explicitly taught you something, acknowledge it briefly according to the result: when an organisation-wide entry was created, merged, or given a new version, say it will be remembered for the organisation; when a preference was saved, say it will be remembered for them; when nothing was written, do not claim that it was remembered. You do not need to tell the user about learning otherwise.
