# Skill: citing-sources

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Main Agent, Analysis Worker |
| When | Always |
| Related | [Citation Registry in the Tool Gateway](../specs/tool-gateway.md), [tool catalog](../specs/tool-catalog.md), [composing-the-final-answer](composing-the-final-answer.md) |

**Why this skill exists.** Citations are the product's first control against hallucination: a user can check any statement in an answer by opening the record, the file passage, or the knowledge entry it came from. Citations only work if they are complete and if they come from the tools rather than from the model, so this skill makes both rules explicit. The Tool Gateway's Citation Registry records every source a tool returned in the conversation; a citation id that the registry did not issue is rejected when the answer is rendered.

The text below is the skill as the agent receives it.

---

## Citing sources

Every factual statement you make about the user's data must be traceable to a source the user can open. You cite sources with citation ids. You never invent a citation id, and you never cite from memory.

### The three kinds of source

1. **Aras record.** An item in Aras: its type, its id, and a link that opens it. Tools that read records (`aml_query`, and `aml_fetch` for workers) return a citation id for each record they return.
2. **File passage.** A part of a file stored in Aras: the file, and the page, sheet, slide, or section the statement rests on. `file_read` and `file_fetch` return citation ids per segment of the file (a page, sheet, slide, or section), not per passage; cite the segment that holds the statement. A phrase search with `file_read` returns each matching passage with its own id, and an Analysis Worker can ask for a finer citation of a passage in its report.
3. **Knowledge entry.** A Discovered or Learned knowledge entry you relied on to interpret the question or the data, for example the definition of a term or the meaning of a workflow step. `knowledge_read` returns a citation id for each entry, and `schema_describe` results also issue knowledge citations for the configuration elements they describe. A knowledge citation carries the entry's state, its confirmation status (learned entries only), and its confidence. The product computes the confidence as a label: **high** for a mechanically derived Discovered entry in state confirmed, and for a Learned entry that is confirmed and independently confirmed; **medium** for a model-inferred Discovered entry in state confirmed, and for a Learned entry that is confirmed but not yet independently confirmed; **low** for any entry that is uncertain or contested. Citing knowledge tells the user that part of the answer depends on what the product has learned, not only on the data.

### Citation sets

A figure such as "412 change orders finished late" rests on hundreds of records, and listing every record is impossible. A **citation set** is one id (it starts with `cst_`) that stands for a whole group of record citations. It is not a fourth kind of source: it is a grouping of Aras record citations, and the user opens it as a paged list of the member records, each of which opens in Aras.

- Set ids are issued by the product, never invented by you. Each `aml_fetch` returns the set id covering every record in the file it wrote, and a worker can register narrower sets in its report (see python-analysis-in-sandbox).
- A computed number over many records cites the set that covers the records counted, plus a few representative record citations (the records behind a short ranking, the examples you show). Write a set like any citation: `[cite:cst_...]`.
- A set id that appears in a worker report is valid. When you use a finding that rests on a set, carry the set id into your answer unchanged.
- Cite a set only for the records it holds. Never invent a set id, and never cite a set as support for a statement about one record that is not a member.

### How to cite

- Place the citation immediately after the statement it supports, in the form `[cite:<citation id>]`. Several sources for one statement: `[cite:<id1>][cite:<id2>]`.
- Cite at the level of the claim. A sentence that combines two facts from two sources carries both citations.
- A computed number (a count, a duration, a share, a ranking) cites the knowledge entries that defined what was counted, and refers to the evidence it was computed from. When the evidence is many records, cite the citation set that covers them plus a few representative records (the records that make up a short list, the top items of a ranking, or the examples you show), and state the total the number was computed over. Do not paste hundreds of citations.
- When you quote or paraphrase a file, cite the segment it comes from (the page, sheet, slide, or section), not only the file.
- When an answer relies on a definition, cite the knowledge entry that holds it. If the definition came from the user in this conversation, say so instead ("as you defined it above").
- An Analysis Worker report carries its own citation ids. When you use a finding from a report, carry its citations into your answer unchanged.
- If you are an Analysis Worker, write `[cite:<id>]` markers in your report's `summary` and in each finding's `statement`, as in any answer. In the structured fields (`support`, and the `basis` of a computed number) give plain citation ids, or citation set ids, with no marker around them.

### What must not be cited, and what must be cited

- Do not cite your general knowledge of PLM or of Aras as if it were a source. If a statement rests only on general knowledge (for example how change processes usually work), say that it is general background, not a finding about the user's data.
- Do not cite a record you did not read in this conversation. If you remember that a record exists from earlier context but no tool returned it, read it again or leave the statement out.
- Every statement about a specific record, file, person's action, date, or value in the user's data must have a citation. If you cannot cite it, do not state it.

### When a source is missing

- If you looked for evidence and found none, say what you looked for and where. That is a coverage statement, not a citation (see reporting-coverage-and-assumptions).
- If a source you cited earlier in the conversation is no longer accessible (a tool reports that access was denied or the record no longer exists), stop relying on it and say so.

### Check before you finish

Before you return an answer or report, read it once for citations:

- Every sentence that states a fact about the data has a citation, or is clearly a computed result with its basis stated.
- Every citation id came from a tool result in this conversation or from a worker report. The product checks every marker before an answer is shown: a marker whose id no tool issued is removed, and its sentence is shown to the user as unverified.
- No citation supports something it does not say. A record that shows a change was created on a date does not support a claim about why it was created.
