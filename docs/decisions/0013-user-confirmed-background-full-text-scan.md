# 0013. A user-confirmed background full-text scan

- Status: Accepted
- Scope: MVP

## Context

Some questions have no item anchor ("have we ever had corrosion issues on coastal installations?"). Without a vector index or search engine ([ADR 0012](0012-no-vector-index-graph-first-retrieval.md)), the graph cannot narrow the files to read. Searching item names, titles, descriptions, and file names through AML is cheap but may miss what is only in file contents. Scanning file contents means downloading and reading many files through the production Aras server, which takes time and load.

## Decision

Questions with no item anchor are in the MVP. The agent first searches item names, titles, descriptions, and file names via AML and gives an interim answer. Full-text search across file contents runs as a long-running background job that the requesting user confirms. The job runs batched and throttled, as that user, with a renewable credential scoped to the job and dropped when it ends.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Not supporting unanchored questions in MVP | These are common, high-value questions for business users. |
| Scanning automatically without confirmation | Significant Aras load and time without the user's agreement; also a target for prompt injection (content asking for a scan). |
| A privileged account to scan everything in advance | Rejected by user-level access ([ADR 0005](0005-user-level-access-through-aras-server.md)). |
| Running the scan in the interactive request | Can exceed interactive token lifetimes and holds the user in front of a spinner; the user should be able to leave and come back. |

## Consequences

- The user sees what will be scanned, that it runs as them, and a rough estimate (candidate files, how many already have stored text, expected duration and expected query load, with no download volume) before anything starts.
- Declining the scan is harmless: the user already has the interim answer from the cheap search.
- Where the installation issues a renewable grant, a job has its own job-scoped credential and keeps running after sign-out; where it does not, the job borrows the user's session credential and pauses when that session ends. A job that loses its credential pauses and resumes from its last completed item ([ADR 0035](0035-job-credential-lifecycle.md), [security and permissions](../architecture/security-and-permissions.md#background-jobs-job-scoped-renewable-credentials)).
- Extracted text stored per file id makes later scans cheaper ([ADR 0014](0014-extracted-text-per-immutable-file-id.md)).
- Coverage reports what was scanned, what was unreadable, and what was denied.

## Related

[Background Jobs](../specs/background-jobs.md), [question lifecycle: no item anchor](../architecture/question-lifecycle.md#5-a-question-with-no-item-anchor-that-becomes-a-background-full-text-job), [ADR 0035](0035-job-credential-lifecycle.md), [ADR 0050](0050-background-job-flow.md).
