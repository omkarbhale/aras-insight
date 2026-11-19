# Skill: clarifying-definitions

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Main Agent |
| When | Always |
| Related | [delegating-analysis](delegating-analysis.md), [uncertain-and-contested-knowledge](uncertain-and-contested-knowledge.md), [persisting-knowledge](persisting-knowledge.md), [contested-knowledge clarification wording](../prompts/contested-knowledge-clarification.md) |

**Why this skill exists.** Many business questions turn on a word that the data does not define: "delayed", "late", "stuck", "recurring", "critical", "last quarter" in a company with a fiscal calendar. Guessing wrong on a cheap lookup costs a follow-up message. Guessing wrong on a long analysis wastes minutes of the user's time and real load on the Aras server. The product rule is therefore asymmetric: for expensive analysis, ask one quick question first, offering a suggested definition; for cheap lookups, proceed and state the assumption.

The text below is the skill as the agent receives it.

---

## Clarifying definitions

Before you answer, find the terms in the question whose meaning decides the result. Then decide whether to ask or to proceed.

### Step 1: find the deciding terms

A deciding term is a word or phrase where two reasonable readings would produce different records, different numbers, or a different ranking. Typical examples: a status word ("delayed", "open", "stuck", "released"), a frequency word ("recurring", "frequent"), a judgement word ("critical", "problematic"), an ambiguous period ("last quarter", "this year"), an ambiguous population ("our parts", "the design review process").

Words that the schema defines unambiguously are not deciding terms.

### Step 2: look for an existing definition

Check all of these. Then use the first one, in this order, that gives a usable definition, unless a Learned entry for the term is contested (see 3):

1. The user defined it earlier in this conversation. Use that.
2. The user's own preferences define it (a personal working definition). Use it for this user, even when an organisation-wide Learned entry also defines the term, and state it as an assumption in the answer ("I used your working definition of 'late'").
3. A Learned knowledge entry defines it. Use `knowledge_index` and `knowledge_read`. A confirmed entry is a usable definition whether or not anyone other than its teacher has confirmed it; follow uncertain-and-contested-knowledge for how to use and disclose it. An uncertain entry is verified first. A contested entry is never a usable definition: it sends you to the question in Step 4, even when the user has a personal working definition of the term.
4. Discovered knowledge suggests one, for example a target date property on the item type, or a lifecycle state whose name matches. This is a suggestion, not a definition, whatever the entry's state: the state says whether the entry still matches the configuration, and its confidence says how far to rely on it. Discovered knowledge may also hold an ambiguity entry for the term: a point where the configuration allows several readings, written as a question. Use it only when the rule in Step 4 tells you to ask, as material for your one question.

A term has a **usable definition** when 1, 2, or 3 gives one. A term that has only a Discovered suggestion, or nothing, has no usable definition.

### Step 3: decide whether the work is expensive

The work is expensive if any of these is true:

- It needs an Analysis Worker, or several.
- It needs a background job.
- It needs to read more than 5 files.
- It needs a query that the Query Analyzer classes as heavy cost (for example fetching a large share of an item type).

Otherwise it is cheap.

### Step 4: ask or proceed

**Expensive work with a deciding term that has no usable definition: ask one quick question before starting.**

- Ask exactly one question. If several terms are undefined, combine them into that one question.
- Offer a concrete suggested definition, built from Discovered or Learned knowledge, phrased in the user's terms and naming the data it relies on. Offer one or two alternatives when they are plausible; an ambiguity entry from Discovery is a good source for them.
- Make it easy to answer: the user should be able to reply "yes" or pick an option.
- Say briefly why you are asking: the analysis will take a while and the definition decides the result.

Example: "Before I start, this will scan about three months of change orders and their reports. How should I treat 'delayed'? I suggest: a change order that reached Released after its target completion date. Alternatively: one that spent more than 30 days in review. Reply with one, or describe your own."

**Cheap work, or a term with a usable definition: proceed.** Do not ask. A Learned entry that only its teacher has confirmed is a usable definition: use it and disclose it as uncertain-and-contested-knowledge describes, instead of asking. State other assumptions about definitions in the answer's basis and assumptions, with a short invitation to correct them: "I took 'open' to mean any state before Released; tell me if you meant something else."

**A contested term: ask, whatever the cost.** Use the contested-knowledge clarification wording. Ask once per conversation: when the user has chosen, use their choice for the rest of the conversation without asking again.

### Step 5: after the user answers

- Use their definition exactly, and repeat it in the answer's assumptions.
- If the user gave a definition that is not yet known, or corrected one that is, propose it as a learning with `record_learning`. A definition is "how" knowledge: what records a term refers to, in terms of the configuration. Do not include any data values. If the user said the definition is only how they use the word, propose it as a personal preference (`scope` personal); otherwise it is an organisation-wide learning (see persisting-knowledge).
- If the user declines to define it ("just use your judgement"), use your suggested definition and state it as an assumption.

### Do not

- Do not ask more than one clarifying question before starting work, unless the user's answer itself introduced a new ambiguity that would change the work.
- Do not ask about terms that do not change the result.
- Do not ask permission to do cheap work. Do it.
- Do not present a Discovered suggestion as the organisation's definition.
- Do not treat a Learned entry as missing because only its teacher has confirmed it.
