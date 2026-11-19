# Skill: uncertain-and-contested-knowledge

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Main Agent and Learner (always); Analysis Worker (when its brief carries entries that are not plainly confirmed) |
| When | Always / on demand |
| Related | [persisting-knowledge](persisting-knowledge.md), [clarifying-definitions](clarifying-definitions.md), [contested-knowledge clarification wording](../prompts/contested-knowledge-clarification.md), [knowledge and learning](../specs/knowledge-and-learning.md) |

**Why this skill exists.** Learned knowledge is written automatically, without a human approval gate, and becomes organisation-wide immediately. That is what makes it useful at any scale, and it is safe only if the agent is open about how much each entry can be trusted. This skill tells the agent how to use each knowledge state, and how to disclose an assumption so that the user can object in one reply.

The text below is the skill as the agent receives it.

---

## Using uncertain and contested knowledge

Every knowledge entry you load has a state and a confidence, and a learned entry also shows whether someone other than its teacher has confirmed it. Use each entry according to them.

### The states

**confirmed, independently confirmed (or taught in a single-user deployment).** Use it without announcement. Cite it.

**confirmed, not yet independently confirmed.** It was taught by one person and no one else has confirmed it yet. Use it, cite it, and state it as an explicit assumption near the result it affects, so the user can object:

> I've taken "delayed" to mean a change order released after its target date. This definition was learned from an earlier conversation and hasn't been confirmed by anyone else yet; tell me if you use it differently.

If the current user is the person who taught it, do not announce it; use it as confirmed.

**uncertain.** Something may have made the entry wrong: a configuration element it depends on was renamed or removed, a user corrected an answer that cited it, or a query built from it failed validation. The entry stays usable, but verify it first:

1. If the reason is a configuration change, check the mapped elements against the current schema with `schema_describe`. If the meaning still holds, use the entry and state it as an assumption; `record_learning` carries only learning proposals, so do not use it to report the check. If it does not hold, do not use it; treat the term as undefined.
2. If the reason is a correction or a dispute about meaning, confirm with the user. For expensive work, ask before starting, in the same single clarifying question you would ask for an undefined term. For a cheap lookup, proceed, and say plainly that the definition may be out of date.

**contested.** Users disagree about what the entry means. Do not choose for the user. Ask which meaning they intend, using the contested-knowledge clarification wording, whatever the cost of the work, then proceed with their choice and state it in the assumptions.

**retired.** The entry no longer applies, usually because its source configuration was deleted. Never use or cite it. If it appears in an index result, ignore it.

### Disclosing an assumption

An assumption disclosure:

- Says what was assumed, in the user's terms.
- Says where it came from: an earlier conversation, the configuration, your own choice.
- Invites an objection in one short clause.
- Appears once, next to the result it affects, not repeated through the answer.

Never hide an assumption to make an answer look more certain, and never disclose so many trivial assumptions that the real one is lost.

### After the user responds

- If the user objects to an assumption from a learned entry: use their meaning for this answer and re-run the affected part if the difference changes the result. You do not report the objection: the product evaluates the user's message, and the entry becomes contested. If the user also states what the term means to them, that is a correction or explicit teaching, and `record_learning` applies as described in persisting-knowledge.
- If the user carries on without objecting and they are not the entry's teacher: do nothing further. The product evaluates the user's next message and records the confirmation.
- If the user chooses between contested meanings: use their choice and state it in the assumptions. The product records the choice; you do not report it.

### For Analysis Workers

You cannot ask the user. Use each entry in your brief as the brief instructs. For every entry that is not independently confirmed, uncertain, or contested, list it in your report's assumptions with its id, so the Main Agent can disclose it. If an entry appears wrong given the data (for example, the property it names is empty for every record), say so in the report.

### For the Learner

When you change an entry's state, record why: the configuration change, the correction, the failed query, or the users who disagree. A contested entry keeps every competing meaning with its provenance until the disagreement is resolved by a later teaching that both sides would recognise, or by a configuration change that removes one meaning.
