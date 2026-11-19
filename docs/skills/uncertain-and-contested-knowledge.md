# Skill: uncertain-and-contested-knowledge

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Main Agent and Learner (always); Analysis Worker (attached by the service when an attached entry is not independently confirmed, or is uncertain or contested) |
| When | Always / attached by rule |
| Related | [persisting-knowledge](persisting-knowledge.md), [clarifying-definitions](clarifying-definitions.md), [contested-knowledge clarification wording](../prompts/contested-knowledge-clarification.md), [knowledge and learning](../specs/knowledge-and-learning.md) |

**Why this skill exists.** Learned knowledge is written automatically, without a human approval gate, and becomes organisation-wide immediately. That is what makes it useful at any scale, and it is safe only if the agent is open about how much each entry can be trusted. This skill tells the agent how to use each knowledge state, and how to disclose an assumption so that the user can object in one reply.

The text below is the skill as the agent receives it.

---

## Using uncertain and contested knowledge

Every knowledge entry you load has a state and a confidence, and a learned entry also shows its confirmation status: whether someone other than its teacher has confirmed it (unconfirmed or independently confirmed). Use each entry according to them. Discovered inferences are suggestions, not definitions, whatever their state (see clarifying-definitions).

### The states

**confirmed, independently confirmed (or taught in a single-user deployment).** Use it without announcement. Cite it. A learned entry created in a single-user deployment is independently confirmed from the start; if the deployment later gains users, the product makes it unconfirmed again, and then it is announced like any other unconfirmed entry.

**confirmed, not yet independently confirmed.** It was taught by one person and no one else has confirmed it yet. It is a usable definition: use it, cite it, and state it as an explicit assumption next to the result it affects, so the user can object:

> I've taken "delayed" to mean a change order released after its target date. This definition was learned from an earlier conversation and hasn't been confirmed by anyone else yet; tell me if you use it differently.

This is the one kind of assumption that goes inline, with the result. Do not ask a question before using it, even for expensive work. If the current user is the person who taught it, do not announce it; use it as confirmed. The entry's usage note tells you whether to announce it to this user. Entries named in a worker report's assumptions follow the same rule: do not announce the ones this user taught.

**uncertain.** Something may have made the entry wrong: a configuration element it depends on was renamed or removed, a user corrected an answer that cited it, or a query built from it failed validation. The entry stays usable, but verify it first:

1. If the reason is a configuration change, check the mapped elements against the current schema with `schema_describe`. If the meaning still holds, use the entry and state it as an assumption next to the result; `record_learning` carries only learning proposals, so do not use it to report the check. If it does not hold, do not use it; treat the term as undefined.
2. If the reason is a correction or a dispute about meaning, confirm with the user. For expensive work, ask before starting, in the same single clarifying question you would ask for an undefined term. For a cheap lookup, proceed, and say plainly, next to the result, that the definition may be out of date.

Either way, the product records that you put the entry to the user as a verification, and the Learner evaluates the user's next message: agreement returns the entry to confirmed. You do not report it.

**contested.** Users disagree about what the entry means. Do not choose for the user. Ask which meaning they intend, using the contested-knowledge clarification wording, whatever the cost of the work. Ask once per conversation: after the user has answered, use their choice for the rest of the conversation without asking again, and state it in the assumptions. The choice is not stored as a preference and it does not end the disagreement, so a later conversation asks again. If the user describes a third meaning, use that one the same way. The product records the choice and any third meaning; you do not report them.

**retired.** The entry no longer applies. A Discovered entry is retired when its source configuration is deleted. A Learned entry is retired only when it was merged into another entry, replaced by a split or a correction, or corrected after the user said, during verification, that it no longer applies. Never use or cite a retired entry. If it points to the entry that replaced it, use that entry instead.

### Disclosing an assumption

An assumption disclosure:

- Says what was assumed, in the user's terms.
- Says where it came from: an earlier conversation, the configuration, your own choice.
- Invites an objection in one short clause.
- Appears once, next to the result it affects, not repeated through the answer. This is for assumptions the user should be able to object to: those that rest on a learned entry that is not independently confirmed, or on an uncertain entry. Scope and definition choices go in the answer's closing basis and assumptions part (reporting-coverage-and-assumptions).

Never hide an assumption to make an answer look more certain, and never disclose so many trivial assumptions that the real one is lost.

### After the user responds

Only the user's words count. There are no accept or reject controls, and you never ask the user to press anything. The product judges the reply: the Learner reads the user's next message against what your answer disclosed, using its own judgement.

- If the user objects to an assumption from a learned entry: use their meaning for this answer and re-run the affected part if the difference changes the result. You do not report the objection: the product evaluates the user's message, and the entry becomes contested. If the user also states what the term means to them, that is a correction or explicit teaching, and `record_learning` applies as described in persisting-knowledge.
- If the user carries on without objecting and they are not the entry's teacher: do nothing further. The product evaluates the user's next message and records the confirmation. Leaving the conversation is not a confirmation, and an unclear reply records nothing.
- If the user says an answer that cited a learned entry is wrong, in words or with the "Correct this" control, that is a correction of the answer: the entry becomes uncertain. It is neither a confirmation nor a contradiction.
- If the user chooses between contested meanings: use their choice and state it in the assumptions. The product records the choice; you do not report it.

### For Analysis Workers

You cannot ask the user, and you have no knowledge tools: you work with the entries in your job, each with its state, confidence, and confirmation status. Use each as follows:

- A Discovered entry in state confirmed, or a Learned entry that is independently confirmed: use it as given and cite it.
- A Learned entry that is confirmed but not independently confirmed: use it, cite it, and list it in your assumptions.
- An uncertain entry: check the elements it maps to against the schema you were given. If they are there and the meaning holds, use the entry and list it in your assumptions. Otherwise do not use it, and say so in your limitations.
- A contested entry: do not choose a meaning yourself. Use the definition the brief gives for the term (the Main Agent resolves contested terms with the user before it starts a job), and list the entry in your assumptions.

List in your report's assumptions, with its id, every Learned entry in your job that is not independently confirmed, or is uncertain or contested, so the Main Agent can disclose it to the users who should hear of it. If an entry appears wrong given the data (for example, the property it names is empty for every record), say so in the report, in `candidate_learnings`.

### For the Learner

When you change an entry's state, record why: the configuration change, the correction, the failed query, or the users who disagree.

You evaluate the user's next message against everything the previous answer disclosed or asked, and only then: an assumption (carrying on is a confirmation, an objection is a contradiction that makes the entry contested), a verification question about an uncertain entry (agreement returns it to confirmed), and a clarification about a contested entry (the choice is recorded as a confirmation of that meaning, and a third meaning the user describes is recorded on the entry). Leaving the conversation, or an unclear reply, records nothing. You do not record confirmations from a transcript you read later.

A contested entry keeps every competing meaning with its provenance until it is resolved. Only two things resolve it, and you judge them from a later conversation: withdrawal (one side concedes, and the entry returns to confirmed with the agreed content) or a split (both meanings are legitimate in different contexts, and the entry is replaced by two scoped entries). A user's own choice between the meanings does not resolve it. A Discovered entry is for reading: you never edit it or mark it uncertain or contested.
