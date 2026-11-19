# Prompt: Background-job confirmation wording

| Field | Value |
|---|---|
| Runs on | The card is rendered by the [Chat App](../specs/chat-app.md) from the proposal returned by `start_background_job`, with no model call. The Main Agent (model slot **main**) writes only the card's title and explanation, the interim answer shown with it, and the answer composed when the job ends |
| Used by | Chat App (the card); Main Agent (title, explanation, interim answer, final or partial answer) |
| Rendered with | Confirm and Decline controls on the card; the job starts only on Confirm, and a running job shows a Cancel control |
| Related | [background jobs](../specs/background-jobs.md), [delegating-analysis](../skills/delegating-analysis.md), unanchored-search (domain skill, see the [catalog](../skills/README.md)), [ADR 0050](../decisions/0050-background-job-flow.md) |

## Purpose

Some questions cannot be answered from item metadata or from a narrowed set of files, for example "which reports mention a particular failure" when nothing in the item graph points to the right files. The product answers these with a full-text scan across file contents, run as a background job: batched, throttled, and as the requesting user, using a renewable credential scoped to the job and dropped when it ends. Because such a job is slow and puts sustained load on the Aras server, it runs only after the user confirms it.

This wording makes that confirmation informed. The user must understand what will be searched, roughly how long it will take, that it runs with their own access, that they can leave, and how to stop it.

## Design notes

- **The card is built from the proposal, not rewritten by a model.** The counts, the scope and the estimate pass from the tool result straight to the user, so no model can change them, and the card can be tested. The Main Agent supplies only the title and one sentence saying what the scan will look for, in the user's terms.
- **Plain words, no internals.** The user sees what will be searched and what it costs them in time, not job ids or batch sizes.
- **Estimates come from the tool.** The estimate holds the number of candidate files, how many already have stored text, the expected duration (always a range) and the approximate query load. There is no download volume: file sizes are not reliably reported, and the user needs counts and time, not megabytes.
- **Show what is already known.** The interim answer, written by the Main Agent and shown with the card, states what the metadata search already found, so the user can decide whether the scan is worth it. The user always has this answer, whether or not they confirm.
- **The credential is explained once, simply.** Users should know the job acts as them and stops acting as them when it ends.
- **Deciding does not start a turn.** Confirm, Decline and Cancel are state changes shown on the card. No model writes a reply to them. The Main Agent acts again only when the job finishes, or is cancelled after it has read files.

## Inputs and placeholders

| Placeholder | Source | Description |
|---|---|---|
| `{{title}}` | `start_background_job` input `title`, written by the Main Agent | The card's heading |
| `{{explanation}}` | `start_background_job` input `explanation`, written by the Main Agent | One sentence saying what the scan will look for, in the user's terms |
| `{{search_terms}}` | `start_background_job` input `search_terms` | The terms the scan looks for, as the user wrote them |
| `{{file_scope}}` | `start_background_job` result (`file_scope`, an echo of the proposal's `file_selection`, for display) | Which files will be searched, for example "files attached to documents of these types, created since a date" |
| `{{file_count}}` | `start_background_job` result (`estimate.candidate_files`) | Number of candidate files the user can access |
| `{{already_extracted}}` | `start_background_job` result (`estimate.cached_files`) | How many of those already have extracted text stored, and so will be fast. When none, its sentence is left out |
| `{{duration_estimate}}` | `start_background_job` result (`estimate.expected_duration`) | Estimated duration, always a range |
| `{{query_load}}` | `start_background_job` result (`estimate.query_load`) | The approximate number of queries the scan will make to Aras |
| `{{found_so_far}}` | Main Agent | In the interim answer: one sentence on what the metadata search found, with citations, or that it found nothing |
| `{{files_read}}` | Background job state | After a cancellation while running: files read before the job stopped |

The card is never filled with "none": a line whose value is empty is left out.

## Full text: interim answer

The Main Agent writes this answer in the same turn that proposes the job, before the card appears. Its coverage says exactly what was and was not searched.

````text
{{found_so_far}}

This covers item names, titles, descriptions and file names only.
````

## Full text: confirmation card

The Chat App renders this card from the proposal.

````text
{{title}}

{{explanation}}

- Looking for: {{search_terms}}
- What: {{file_scope}}, about {{file_count}} files you have access to. {{already_extracted}} have been read before and will be quick.
- How long: about {{duration_estimate}}. It runs in the background, paced so it does not slow Aras down for others (about {{query_load}} queries to Aras).
- Access: it searches as you, with your permissions, using a temporary sign-in limited to this job. That sign-in ends when the job does.
- Files without readable text, such as scanned documents and images, cannot be searched in this release; they will be listed in the answer's coverage.
- You can leave this page and come back; results will appear in this conversation. You can cancel at any time and keep what was found so far.

[Confirm] [Decline]
````

## After the user decides

Confirm, Decline and Cancel do not start a Main Agent turn, and no model writes a reply to them.

- **Confirm.** The card shows the job as running, with live progress (files read of the total, matches so far) and a Cancel control.
- **Decline.** The card shows the proposal as declined. The interim answer stays as the answer to the question.
- **Cancel before the job reads any file.** The card shows the job as cancelled.
- **Cancel while running.** The matches found so far are kept. The job ends as cancelled and Background Jobs posts the partial result into the conversation and starts a Main Agent turn, through the same path as completion. The Main Agent opens its answer with this line, then composes the partial answer as usual, with coverage stating exactly which files were not searched:

````text
Search stopped. It had read {{files_read}} of {{file_count}} files. Here is what it found so far.
````

- **Finished.** Background Jobs posts the result into the conversation and starts a Main Agent turn automatically, even if the user has left. The Main Agent composes the answer with the [composing-the-final-answer](../skills/composing-the-final-answer.md) skill, so the citation, coverage and causal-honesty rules apply in one place. The answer waits in the conversation, and the user sees it when they return. Coverage states files searched, files with no text layer or in unsupported formats, and any files the user lost access to during the job. Passages carry file citations, each with its parent item.
