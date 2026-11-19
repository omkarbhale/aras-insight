# 0046. Analysis job contract: structured inputs, knowledge only through the brief, citation sets

- Status: Accepted
- Scope: MVP

## Context

Workers cannot ask the user or look anything up, so the tool, the brief and the worker's own tools must line up. A count such as "412 change orders finished late" rests on hundreds of records that the user must be able to check.

## Decision

- `start_analysis` takes structured inputs. The tool catalog is their single source and the brief is the service's rendering of them, so the audit record shows each section. Starting records are citation ids, a definition's source is `user`, `knowledge` or `assumption`, and the service builds the schema extract from the item types the brief names.
- The brief is the worker's whole basis: no `schema_describe` or `knowledge_read`. A worker that needs more says so, and the Main Agent decides on a new job.
- The service attaches the uncertain-and-contested-knowledge skill when an attached entry is unconfirmed, uncertain or contested.
- At a limit the worker stops fetching, analyses what it has and reports partial results with exact coverage. Narrowing the scope is the Main Agent's call.
- Citation sets stay: one `cst_` id stands for a group of record citations, opened as a paged list, not a fourth citation kind. `aml_fetch` registers one, workers may register narrower ones, and a computed number's basis is a set id plus up to `analysis.sample_citations_per_number` (10) records.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| One free-text brief | The audit record could not show sections. |
| Workers get schema and knowledge tools | The brief stops fixing scope and context. |
| The Main Agent lists skills | A model choice where the service knows the states. |
| The worker narrows the scope | It changes the question the brief set. |
| Drop sets | Users could not verify a count. |

## Consequences

- A worker can stall on an item type nobody named. If the pilot shows this often, `schema_describe` for workers is the first fallback.
- Prompts and skills must teach set ids.

## Related

[Tool catalog](../specs/tool-catalog.md), [Analysis Worker and Sandbox](../specs/analysis-worker-and-sandbox.md), [analysis job brief](../prompts/analysis-job-brief.md), [ADR 0010](0010-main-agent-plus-analysis-workers.md), [ADR 0029](0029-three-kinds-of-citation.md).
