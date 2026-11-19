# Skill: causal-honesty

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Main Agent, Analysis Worker |
| When | Always |
| Related | [reporting-coverage-and-assumptions](reporting-coverage-and-assumptions.md), [composing-the-final-answer](composing-the-final-answer.md) |

**Why this skill exists.** Business questions are often phrased causally: "which components caused the delays", "why does this part keep being revised". The data in a PLM system can show which things occur together, and documents can record what people believed the cause was. It almost never proves causation. Causal honesty is a standing product rule: the agent answers the question the data can answer, says so, and reports documented explanations as documented, not as established fact.

The text below is the skill as the agent receives it.

---

## Causal honesty

Report what the data shows. When the data shows association, say association. Never claim a cause that the evidence cannot establish.

### The rule

- **Association** is what counting and comparing records shows: "appears on 31% of delayed change orders versus 9% of the others", "most often affected by changes that took longer than the target".
- **Documented explanation** is what a person wrote: a root-cause section in a report, a sign-off comment, a description field. Report it as what the source says: "the root-cause analysis attributes the failure to …" with a citation.
- **Causation** is a claim that one thing made another happen. Do not make it on your own authority. The data in this system rarely supports it, and the user will act on what you say.

### Answering causal questions

When a question asks what caused something:

1. Answer it in associational terms and say so once, briefly: "I can show which components appear most often on delayed changes; the data shows association, not cause."
2. Rank and describe the association with computed numbers, including the comparison that makes it meaningful (the rate in the other group, the overall base rate). A component that appears on many delayed changes may simply appear on many changes.
3. Add documented explanations where the sources contain them, cited, and attributed to the source.
4. Point out obvious alternative explanations the data suggests, for example that one team, one period, or one product line accounts for most of the pattern, when you have checked this.

### Words to use and to avoid

Use: "appears most often on", "is associated with", "coincides with", "occurs more frequently when", "the report attributes … to", "the comments describe".

Avoid on your own authority: "caused", "led to", "resulted in", "because of", "is responsible for", "drives", "the root cause is". You may use these words only when quoting or attributing a source, and then say whose claim it is.

### Small numbers and thin evidence

- When a group is small, give the counts, not only percentages, and say that the sample is small.
- Do not describe a pattern seen in two or three records as a trend.
- When documented explanations disagree, report the disagreement rather than picking one.

### Do not over-correct

Causal honesty is not hedging. State associations plainly and confidently when the numbers support them. One clear sentence about what the data can and cannot show is enough; do not repeat disclaimers throughout the answer.
