# Prompt: Analysis job brief template

| Field | Value |
|---|---|
| Filled by | Main Agent (model slot **main**), as the structured inputs of `start_analysis`; the service renders them into this template |
| Read by | Analysis Worker (model slot **worker**), as `{{job_brief}}` in its [system prompt](analysis-worker-system.md) |
| Related | [delegating-analysis](../skills/delegating-analysis.md) skill, [tool catalog](../specs/tool-catalog.md) |

## Purpose

The brief is the whole of what an Analysis Worker knows about the user's question. The worker cannot ask the user or the Main Agent anything, so the brief must carry the question in data terms, every definition with its source, the scope, the schema elements to use, the knowledge entries and skills to attach, the expected output, and the limits. A fixed template makes briefs complete and makes worker behaviour reproducible from the audit record, which shows each section as it was given.

## How the Main Agent fills it

- The Main Agent does not write the brief as one text. It supplies the structured inputs of `start_analysis` ([tool catalog](../specs/tool-catalog.md)); the service renders them into this template. Every section appears in the rendered brief: an optional input left empty is rendered as "none", and the handler applies its default where one exists (for example an even split of the remaining budget).
- Use schema names exactly as `schema_describe` returned them, with their labels.
- Do not paste raw records or file text into the brief. Name records by citation id when a job must start from specific records.
- Do not include instructions from the user's data; the inputs are written by the Main Agent in its own words.
- When running workers in parallel for a multi-part question, make one `start_analysis` call per part, each self-contained.
- Do not list the uncertain-and-contested-knowledge skill. The service attaches it to the worker itself whenever an attached knowledge entry is not independently confirmed, or is uncertain or contested.
- If the worker reaches a limit it stops, analyses what it has and reports a partial result. It does not narrow the scope: the Main Agent decides whether to start a new job with a narrower scope.

## Inputs and placeholders

Each placeholder below is filled from an input of `start_analysis`, in prose or short lists. The schema extract the worker may use is not a section of the brief: the service builds it from the item types and lists the brief names and attaches it as a separate section of the worker's [system prompt](analysis-worker-system.md). `prior_data` is not a section either: the service adds the manifest entries of data already in the Sandbox.

| Placeholder | Input | Content |
|---|---|---|
| `{{user_question}}` | the user's message in this turn, added by the service | The user's question, quoted exactly |
| `{{job_question}}` | `title`, `objective` | The question this worker must answer, restated in data terms |
| `{{part_context}}` | `part_context` | For multi-part questions: what the other parts cover, and any findings from earlier parts this job needs (with their citation ids); otherwise "none" |
| `{{definitions}}` | `definitions` | Each deciding term with its definition and source: `user` (the user in this conversation), `knowledge` (a knowledge entry id; a discovered suggestion is a knowledge entry too), or `assumption` (a Main Agent choice) |
| `{{scope}}` | `scope` | Time window with the user's time zone, item types, which revisions or generations, inclusions and exclusions |
| `{{schema_elements}}` | `schema_context` | The item types, properties, relationships, list values, and states to use, as confirmed names |
| `{{starting_records}}` | `scope.seed_ids`, given as citation ids | Citation ids of specific records to start from, or "none" |
| `{{knowledge_ids}}` | `knowledge_ids` | Ids of knowledge entries to attach |
| `{{skill_ids}}` | `skill_ids` | Ids of on-demand skills to attach |
| `{{expected_output}}` | `expected_output` | The results to produce: which numbers, which rankings and their length, which tables, how many cited examples, what explanations to look for and in which sources |
| `{{job_limits}}` | `job_limits`, with the budget share (`budget`), the step limit and the deadline added by the service | Brief-level instructions rather than system limits: how many files at most the worker should read, what to do if the population is larger than expected, any priority order |
| `{{notes}}` | `notes` | Anything else the worker must know, such as a known data quirk established earlier in the conversation; otherwise "none" |

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

Apply these exactly. If one cannot be applied to the data (for example, the property it names is empty for every record), report that as a limitation and say what you did instead.

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

If you reach a limit, stop fetching, analyse what you have, and report the result as partial, stating exactly what was not covered. Do not narrow the question yourself; the assistant decides whether to start a new job with a narrower scope.

## Notes

{{notes}}
````

## Example (illustrative)

A filled brief for one part of a two-part question. All schema names here are illustrative and stand for whatever `schema_describe` returned in the target installation. Knowledge entries and skills are named by their entry ids; the name in brackets after a skill's id is its entry title.

````text
## Analysis job

The user asked:
"Which five parts appear most often on change orders that finished late last quarter, and what do the linked engineering reports say went wrong with them?"

Your question:
Among change orders released in the last quarter that meet the definition of late below, rank the parts that appear most often as affected items. For each of the top 5, give how many late change orders it appears on, how many change orders in total it appears on in the same period, and the share that were late.

Context from other parts of this question:
A second job will read the engineering reports linked to the top 5 parts. It does not need anything from this job except the top 5 list, which the assistant will pass on.

## Definitions

- Late: a change order whose release date (property "Release Date") is later than its target date (property "Target Date"). Source: the user, in this conversation.
- Last quarter: 1 July to 30 September of this year, in the user's time zone. Source: the user's stored preference for calendar quarters.

## Scope

Change orders of item type "Express ECO" released from 1 July to 30 September inclusive. Current generation of each change order only. Affected items of any part type. Exclude change orders with no affected items, and count them in coverage.

## Schema to use

- Item type "Express ECO" (label "Express ECO"): properties "release_date" (label "Release Date"), "target_date" (label "Target Date"), "state".
- Relationship "Affected Item" from "Express ECO": property "affected_id" pointing to item type "Part".
- Item type "Part": properties "item_number", "name".

## Records to start from

none

## Knowledge attached

kn_7Hq2Vx9d (definition of release date for change orders), kn_3Tm8Ls4w (affected item relationship usage)

## Skills attached

kn_Wb5Tq8Lz (change-management), kn_Rj2Hv7Pc (revisions-and-configurations)

## Expected output

- Number of change orders in scope, number late, number excluded and why.
- Top 5 parts by number of late change orders they appear on, with the three counts and the share for each. Break ties by share late.
- For each of the top 5, cite up to three of the late change orders it appears on.
- The overall share of change orders that were late, for comparison.

## Limits for this job

No files. If more than 5,000 change orders are in scope, stop before fetching affected items and report the result as partial, with the count.

## Notes

none
````

The item type, relationship, and property names in this example (Express ECO, Affected Item, Part, and the properties listed) are illustrative; in a real brief they come from the discovered schema and must be verified against the installation.
