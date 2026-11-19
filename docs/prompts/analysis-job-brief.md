# Prompt: Analysis job brief template

| Field | Value |
|---|---|
| Filled by | Main Agent (model slot **main**), as the `brief` input of `start_analysis` |
| Read by | Analysis Worker (model slot **worker**), as `{{job_brief}}` in its [system prompt](analysis-worker-system.md) |
| Related | [delegating-analysis](../skills/delegating-analysis.md) skill, [tool catalog](../specs/tool-catalog.md) |

## Purpose

The brief is the whole of what an Analysis Worker knows about the user's question. The worker cannot ask the user or the Main Agent anything, so the brief must carry the question in data terms, every definition with its source, the scope, the schema elements to use, the knowledge entries and skills to attach, the expected output, and the limits. A fixed template makes briefs complete and makes worker behaviour reproducible from the audit record.

## How the Main Agent fills it

- Every section is required. Write "none" where a section genuinely does not apply; do not delete it.
- Use schema names exactly as `schema_describe` returned them, with their labels.
- Do not paste raw records or file text into the brief. Name records by citation id when a job must start from specific records.
- Do not include instructions from the user's data; the brief is written by the Main Agent in its own words.
- When running workers in parallel for a multi-part question, write one brief per part, each self-contained.

## Inputs and placeholders

Each placeholder below is filled by the Main Agent in prose or short lists.

| Placeholder | Content |
|---|---|
| `{{user_question}}` | The user's question, quoted exactly |
| `{{job_question}}` | The question this worker must answer, restated in data terms |
| `{{part_context}}` | For multi-part questions: what the other parts cover, and any findings from earlier parts this job needs (with their citation ids); otherwise "none" |
| `{{definitions}}` | Each deciding term with its definition and source (user in this conversation, knowledge entry id, discovered suggestion, or Main Agent choice) |
| `{{scope}}` | Time window with time zone, item types, which revisions or generations, inclusions and exclusions |
| `{{schema_elements}}` | The item types, properties, relationships, list values, and states to use, as confirmed names |
| `{{starting_records}}` | Citation ids of specific records to start from, or "none" |
| `{{knowledge_ids}}` | Ids of knowledge entries to attach |
| `{{skill_ids}}` | Ids of on-demand skills to attach |
| `{{expected_output}}` | The results to produce: which numbers, which rankings and their length, which tables, how many cited examples, what explanations to look for and in which sources |
| `{{job_limits}}` | Maximum files to read, what to do if the population is larger than expected, any priority order |
| `{{notes}}` | Anything else the worker must know, such as a known data quirk established earlier in the conversation; otherwise "none" |

## Full text

````text
## Analysis job

The user asked:
"{{user_question}}"

Your question:
{{job_question}}

Context from other parts of this question:
{{part_context}}

## Definitions

Apply these exactly. If one cannot be applied to the data (for example, the property it names is empty for every record), report that as an issue and say what you did instead.

{{definitions}}

## Scope

{{scope}}

## Schema to use

{{schema_elements}}

## Records to start from

{{starting_records}}

## Knowledge attached

{{knowledge_ids}}

## Skills attached

{{skill_ids}}

## Expected output

{{expected_output}}

Report every number with the population it was computed over. Cite representative evidence for every finding.

## Limits for this job

{{job_limits}}

If you reach a limit, stop fetching, analyse what you have, and report the result as partial, stating exactly what was not covered.

## Notes

{{notes}}
````

## Example (illustrative)

A filled brief for one part of a two-part question. All schema names here are illustrative and stand for whatever `schema_describe` returned in the target installation.

````text
## Analysis job

The user asked:
"List the top 5 components behind change order delays last quarter and explain the common failure mode in the associated engineering reports."

Your question:
Among change orders created in the last quarter that meet the definition of delayed below, rank the parts that appear most often as affected items. For each of the top 5, give how many delayed change orders it appears on, how many change orders in total it appears on in the same period, and the share delayed.

Context from other parts of this question:
A second job will read the engineering reports linked to the top 5 parts. It does not need anything from this job except the top 5 list, which the assistant will pass on.

## Definitions

- Delayed: a change order whose release date (property "Release Date") is later than its target date (property "Target Date"). Source: the user, in this conversation.
- Last quarter: 1 July to 30 September of this year, in the user's time zone. Source: the user's stored preference for calendar quarters.

## Scope

Change orders of item type "Express ECO" created from 1 July to 30 September inclusive. Current generation of each change order only. Affected items of any part type. Exclude change orders with no affected items, and count them in coverage.

## Schema to use

- Item type "Express ECO" (label "Express ECO"): properties "created_on", "release_date" (label "Release Date"), "target_date" (label "Target Date"), "state".
- Relationship "Affected Item" from "Express ECO": property "affected_id" pointing to item type "Part".
- Item type "Part": properties "item_number", "name".

## Records to start from

none

## Knowledge attached

k-0142 (definition of release date for change orders), k-0087 (affected item relationship usage)

## Skills attached

change-management, revisions-and-configurations

## Expected output

- Number of change orders in scope, number delayed, number excluded and why.
- Top 5 parts by number of delayed change orders they appear on, with the three counts and the share for each. Break ties by share delayed.
- For each of the top 5, cite up to three of the delayed change orders it appears on.
- The overall share of change orders delayed, for comparison.

## Limits for this job

No files. If more than 5,000 change orders are in scope, report the count and stop before fetching affected items.

## Notes

none
````

## Open questions

1. Proposed: the brief is passed as structured sections rather than free text so that the audit record can show each section. Whether `start_analysis` takes the filled text or the individual fields is for the [tool catalog](../specs/tool-catalog.md) to decide.
