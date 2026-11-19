# Prompt: Background-job confirmation wording

| Field | Value |
|---|---|
| Runs on | Model slot **main** (the Main Agent writes the message from this template) |
| Used by | Main Agent, after `start_background_job` returns a proposal and before the job runs |
| Rendered with | Confirm and Cancel controls in the [Chat App](../specs/chat-app.md); the job starts only on Confirm |
| Related | [background jobs](../specs/background-jobs.md), [delegating-analysis](../skills/delegating-analysis.md), unanchored-search (domain skill, see the [catalog](../skills/README.md)) |

## Purpose

Some questions cannot be answered from item metadata or from a narrowed set of files, for example "which reports mention a particular failure" when nothing in the item graph points to the right files. The product answers these with a full-text scan across file contents, run as a background job: batched, throttled, and as the requesting user, using a renewable credential scoped to the job and dropped when it ends. Because such a job is slow and puts sustained load on the Aras server, it runs only after the user confirms it.

This wording makes that confirmation informed. The user must understand what will be searched, roughly how long it will take, that it runs with their own access, that they can leave, and how to stop it.

## Design notes

- **Plain words, no internals.** The user sees what will be searched and what it costs them in time, not job ids or batch sizes.
- **Estimates come from the tool.** Counts and durations in the message are those `start_background_job` returned; the Main Agent does not estimate them itself. If the tool returned no duration estimate, the line is omitted rather than guessed.
- **Show what is already known.** The message states what the metadata search already found, so the user can decide whether the scan is worth it.
- **The credential is explained once, simply.** Users should know the job acts as them and stops acting as them when it ends.

## Inputs and placeholders

| Placeholder | Source | Description |
|---|---|---|
| `{{found_so_far}}` | Main Agent | One sentence on what the metadata search found, with citations, or that it found nothing |
| `{{search_description}}` | Main Agent | What the scan will look for, in the user's terms |
| `{{file_scope}}` | `start_background_job` proposal | Which files will be searched, for example "files attached to documents of these types, created since a date" |
| `{{file_count}}` | `start_background_job` proposal | Number of candidate files the user can access |
| `{{already_extracted}}` | `start_background_job` proposal | How many of those already have extracted text stored, and so will be fast |
| `{{duration_estimate}}` | `start_background_job` proposal | Estimated duration, if the tool returned one |
| `{{format_note}}` | Main Agent | Which files in scope cannot be read in this release (scanned documents, images, email files, other formats) and will be listed in coverage, if the proposal reports any |
| `{{alternative}}` | Main Agent | After a cancellation: one sentence offering a cheaper option, or empty |
| `{{files_read}}` | Background job state | After a cancellation while running: files read before the job stopped |

## Full text: confirmation request

The Main Agent writes this message, filling the placeholders and leaving out any line whose placeholder is empty.

````text
{{found_so_far}}

To go further I would need to search inside the files themselves: {{search_description}}.

- What: {{file_scope}}, about {{file_count}} files you have access to. {{already_extracted}} have been read before and will be quick.
- How long: about {{duration_estimate}}. It runs in the background, paced so it does not slow Aras down for others.
- Access: it searches as you, with your permissions, using a temporary sign-in limited to this job. That sign-in ends when the job does.
- {{format_note}}
- You can leave this page and come back; results will appear in this conversation. You can cancel at any time and keep what was found so far.

Shall I start the search?
````

## Full text: after the user responds

**Confirmed.** One line, then progress appears in the live view:

````text
Started. I'll post the results here when the search finishes.
````

**Cancelled before starting.**

````text
Understood, I won't search the file contents. {{alternative}}
````

`{{alternative}}` is one sentence offering a cheaper option when there is one, for example narrowing to files linked to a specific set of items, or nothing.

**Cancelled while running.**

````text
Search stopped. It had read {{files_read}} of {{file_count}} files. Here is what it found so far.
````

followed by the partial answer, composed as usual, with coverage stating exactly which files were not searched.

**Finished.** The answer is composed with the [composing-the-final-answer](../skills/composing-the-final-answer.md) skill. Coverage states files searched, files with no text layer or in unsupported formats, and any files the user lost access to during the job.

## Open questions

1. Proposed: the confirmation is a normal assistant message with Confirm and Cancel controls rendered by the Chat App, rather than a separate dialog. To be aligned with the [Chat App](../specs/chat-app.md).
2. Proposed: when no duration estimate is available, the "How long" line says "This may take a while" instead of a number. The estimation method belongs to [background jobs](../specs/background-jobs.md).
