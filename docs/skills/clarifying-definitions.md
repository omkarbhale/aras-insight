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

In this order:

1. The user defined it earlier in this conversation. Use that.
2. A Learned knowledge entry defines it. Use `knowledge_index` and `knowledge_read`. If the entry is confirmed, use it and cite it. If it is unconfirmed, uncertain, or contested, follow uncertain-and-contested-knowledge.
3. The user's preferences define it. Use that.
4. Discovered knowledge suggests one, for example a target date property on the item type, or a lifecycle state whose name matches. This is a suggestion, not a definition.

### Step 3: decide whether the work is expensive

The work is expensive if any of these is true:

- It needs an Analysis Worker, or several.
- It needs a background job.
- It needs to read more than a handful of files.
- It needs to fetch a large share of an item type, or the Query Analyzer reports a high cost.

Otherwise it is cheap.

### Step 4: ask or proceed

**Expensive work with a deciding term that has no confirmed definition: ask one quick question before starting.**

- Ask exactly one question. If several terms are undefined, combine them into that one question.
- Offer a concrete suggested definition, built from Discovered or Learned knowledge, phrased in the user's terms and naming the data it relies on. Offer one or two alternatives when they are plausible.
- Make it easy to answer: the user should be able to reply "yes" or pick an option.
- Say briefly why you are asking: the analysis will take a while and the definition decides the result.

Example: "Before I start, this will scan about three months of change orders and their reports. How should I treat 'delayed'? I suggest: a change order that reached Released after its target completion date. Alternatively: one that spent more than 30 days in review. Reply with one, or describe your own."

**Cheap work, or a term with a confirmed definition: proceed.** State the assumption in the answer and invite correction in a short clause: "I took 'open' to mean any state before Released; tell me if you meant something else."

**A contested term: ask, whatever the cost.** Use the contested-knowledge clarification wording.

### Step 5: after the user answers

- Use their definition exactly, and repeat it in the answer's assumptions.
- If the user gave a definition that is not yet known, or corrected one that is, propose it as a learning with `record_learning`. A definition is "how" knowledge: what records a term refers to, in terms of the configuration. Do not include any data values.
- If the user declines to define it ("just use your judgement"), use your suggested definition and state it as an assumption.

### Do not

- Do not ask more than one clarifying question before starting work, unless the user's answer itself introduced a new ambiguity that would change the work.
- Do not ask about terms that do not change the result.
- Do not ask permission to do cheap work. Do it.
- Do not present a Discovered suggestion as the organisation's definition.
