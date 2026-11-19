# File Service specification

Status: MVP specification. Related: [Tool Gateway](tool-gateway.md), [Tool catalog](tool-catalog.md), [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md), [Background Jobs](background-jobs.md), [Persistence](persistence.md), [Security and permissions](../architecture/security-and-permissions.md)

## 1. Purpose

Much of the reasoning behind engineering decisions is not stored in item properties. It is in the documents attached to items: engineering reports, root-cause analyses, specifications, test results, meeting minutes. The File Service lets the agents read those documents without weakening Aras permissions and without building a separate search index: extracted text is searched by reading it, either by the model through `file_read`, by code in the Sandbox after `file_fetch`, or by a background full-text scan that matches text in the service (see [Background Jobs](background-jobs.md)). No database text index or vector index is built over it.

The File Service is a part of the [Tool Gateway](tool-gateway.md). It fetches vault files through Aras as the signed-in user, extracts their text, stores the extracted text keyed by the Aras file id, and serves that text to the agent tools. It checks the user's access through Aras again on every use, so the stored text never reaches a user who cannot open the file in Aras.

Design choices recorded in the product decisions, and why:

| Choice | Reason |
|---|---|
| No vector index or external search engine in MVP | The model is the semantic layer. It narrows candidate files through the Aras item graph, then reads them. This avoids a second copy of permissions and keeps the system small. |
| Fetch through Aras as the user | Aras remains the only authority on who may read a file. No privileged service account is needed. |
| Extracted text keyed by file id | Aras vault files are immutable (a changed document produces a new file), so extracted text can be stored once and reused for later questions and background scans without becoming stale. |
| Access re-checked on every use | Stored text is a convenience, not a grant. Permissions may change after extraction, and different users ask about the same file. |
| Unreadable files reported, not hidden | Answers must state coverage honestly. A scanned PDF that cannot be read is listed as such rather than silently ignored. |

## 2. Responsibilities and non-responsibilities

Responsibilities:

1. Verify, as the requesting user and through Aras, that the user may read a given file, on every request.
2. Download file content from the Aras vault as the user when no usable extracted text is stored.
3. Detect the file format and extract plain text for the MVP formats.
4. Split extracted text into segments (pages, sheets, slides, sections, character ranges) so passages can be cited precisely.
5. Store extracted text and segment maps in the Database keyed by the Aras file id, together with extraction status and extractor version.
6. Serve text, or a range of it, to the `file_read` tool, write text into a Sandbox for the `file_fetch` tool, and serve text for full-text scans run by [Background Jobs](background-jobs.md).
7. Report per-file read status (read, partially read, unreadable, unsupported, too large, denied, failed) so that coverage in answers is complete.
8. Register `file` citations with the Citation Registry: one per segment of text it returns (or per `find` passage), and one file-level citation for each file it reports as unreadable, unsupported, or too large, so that coverage can list the file and the user can open it in Aras. A file the user may not access gets no citation and no name.

Non-responsibilities:

- It does not decide which files are relevant. The Main Agent or Analysis Worker narrows candidates through AML over the item graph.
- It does not rank, embed, or semantically index text (deferred; see [Roadmap](../product/roadmap.md)).
- It does not run OCR or read images, email files, the legacy binary Office formats (.doc, .xls, .ppt), or other non-MVP formats (deferred).
- It does not interpret CAD geometry (not supported in MVP).
- It does not write to Aras or to the vault. The product is read-only.
- It does not search across files by itself for interactive questions. Literal searching inside fetched text happens in the Sandbox with code, or in a user-confirmed full-text scan job.
- It does not keep a history of Aras data. Extracted text belongs to an immutable file and is not a record of changes.

## 3. Interfaces

The File Service has no externally reachable API. It is called only from within the Tool Gateway, by the tool handlers and by Background Jobs. Agent-facing schemas for `file_read` and `file_fetch` are defined in the [tool catalog](tool-catalog.md#file_read) and [tool catalog](tool-catalog.md#file_fetch); this section defines the internal operations behind them.

Every operation receives a user context: the user id, the user's current Aras credential held by the Tool Gateway (or a job-scoped credential for background jobs), the conversation id, turn id, and tool call id. Results use the tool result envelope defined in [tool catalog](tool-catalog.md).

| Operation | Caller | Inputs | Outputs | Errors |
|---|---|---|---|---|
| Check access | All other operations; Background Jobs (batched form) | One or more Aras file ids; user context | Per file: visible or not visible, file name, size, and the file metadata Aras returns | ARAS_UNAVAILABLE, ARAS_AUTH_EXPIRED, TIMEOUT |
| Get text | `file_read` handler | File id; optional range (a page range, a sheet name with an optional cell range, a slide range, a section heading, or a character offset and length) or a `find` phrase; optional parent item reference the file was reached through; user context | Text (at most `file_service.read_max_chars`) or, for `find`, up to 20 matching passages with context; the segments covered, total length, next-range hint, extraction status, citation ids | PERMISSION_DENIED, NOT_FOUND, FILE_UNREADABLE (including a file over the size limit, reason `file_too_large`), FILE_UNSUPPORTED_FORMAT, ARAS_UNAVAILABLE, ARAS_AUTH_EXPIRED, TIMEOUT, THROTTLED, BUDGET_EXHAUSTED |
| Fetch into Sandbox | `file_fetch` handler | List of file ids (at most `file_service.fetch_max_files`); target Sandbox id; optional parent item references; user context | Per file: status, Sandbox paths written (text and segment map), character count, citation id; a coverage summary | Per-file statuses as above; SANDBOX_UNAVAILABLE for the whole call, or RESULT_TOO_LARGE when a write would exceed the Sandbox disk quota; THROTTLED, BUDGET_EXHAUSTED |
| Get text for scan | Background Jobs (full-text scan) | File id; job-scoped credential; job id | Full extracted text and segment map, or a per-file status | Same per-file statuses; ARAS_AUTH_EXPIRED pauses the job |
| Describe | Main Agent tool handlers, Chat App (citation rendering) | File id; user context | File name, format, size, page or sheet or slide count, extraction status, link to open in Aras | PERMISSION_DENIED, NOT_FOUND |

Error semantics specific to files:

- PERMISSION_DENIED is returned when Aras does not show the file to the user, even if extracted text is stored. The response never contains any part of the stored text, its length, or its segment labels.
- NOT_FOUND is returned when Aras reports that no such file exists. The File Service cannot distinguish "deleted" from "never visible" in every Aras configuration; both appear to the model as "not available to you", and both count as "could not be read" in coverage.
- FILE_UNREADABLE is returned when extraction produced no usable text (for example a scanned PDF without a text layer, a password-protected document, or a corrupt file) or when the file is over the size limit. The error carries a reason in `error.reason`: `no_text_layer`, `encrypted`, `corrupt`, `extraction_timeout`, `extraction_failed`, or `file_too_large` (the file exceeds `file_service.max_file_bytes`; narrowing the request cannot help, so it is not retryable, and `file_fetch` reports it as status `too_large`). The full reason vocabulary, shared with every other specification, is these six plus `truncated` (status `partial`).
- FILE_UNSUPPORTED_FORMAT is returned for formats outside the MVP list, including the legacy binary Office formats. The error carries the detected format in `error.details`.

## 4. Data owned

| Table | Contents | Schema |
|---|---|---|
| `file_texts` | One row per Aras file id: file name, detected format, byte size, extraction status and reason, extractor version, page or sheet or slide count, character count, the extracted text, extraction timestamp, last-used timestamp | [persistence.md#file_texts](persistence.md#file_texts) |
| `file_text_segments` | One row per segment of a file's text: segment number, kind (page, sheet, slide, section, range), label (for example "page 7", "sheet Results", "slide 3", a section heading), start and end character offsets, and a flag for segments that had no extractable text | [persistence.md#file_text_segments](persistence.md#file_text_segments) |

The File Service owns no access-control data. Visibility is never cached: there is no table recording who may read which file.

The vault file bytes are never persisted. They are held in memory or in a temporary location private to the extraction step and deleted when extraction ends. A later re-extraction (for example after an extractor upgrade) downloads the file from Aras again, and that download counts against the question's byte budget. A byte cache would be a second copy of business files with its own permission and retention problems; the stored extracted text already avoids most downloads.

## 5. Behaviour

### 5.1 Access check

1. The File Service issues a read-only AML request as the user for the file's metadata, selecting only the identifying fields it needs (typically the file name, size, and checksum properties of the standard File item type; the names are verified for each supported Aras version through Discovery and recorded with the list of supported versions). The request goes through the AML Executor, so it passes the [Query Analyzer](query-analyzer.md) and the [Throttle](tool-gateway.md) like any other query and is counted in the question budget.
2. If Aras returns the file, access is granted for this operation only. The decision is not stored.
3. If Aras returns no result or an access error, the operation returns PERMISSION_DENIED or NOT_FOUND and stops.
4. For several files in one call, the check is made in a single AML request listing the ids (batched form), capped at `file_service.access_check_batch_size` ids per request.

Server events and custom read-time methods configured in Aras (for example an onBeforeGet event on the File item type, if the installation uses one) run as part of this request, so any custom restriction Aras applies is honoured automatically.

### 5.2 Get text

1. Run the access check (5.1).
2. Look up `file_texts` by the file id.
3. If a row exists with status `ok` or `partial` and an extractor version not older than `file_service.min_extractor_version`, use it and go to step 6.
4. If a row exists with a stored failure status (`unreadable`, `unsupported`, `too_large`, or a deterministic `failed`) and the extractor version is current, return the stored status without downloading again. A stored `too_large` row is evaluated again when `file_service.max_file_bytes` has since been raised above the file's size.
5. Otherwise download and extract (5.3). Only one extraction per file id runs at a time across the service; a second request for the same file waits for the first one to finish and then uses its result.
6. Select the requested text. A page range, slide range, sheet (with an optional cell range), or section heading is translated into the matching segments of the segment map, and their text is returned; if a character offset and length are given, return that range; if no range is given, return text from the beginning. If a `find` phrase is given, search the stored text of this one file for a literal match of the phrase (no ranking and no semantic search) and return up to 20 passages, each with its surrounding context and its locator. Truncate to `file_service.read_max_chars`, ending on a segment or line boundary where possible.
7. Register one `file` citation with the Citation Registry for each segment returned, or for each `find` passage (file id, file name, the segment's or passage's locator, the parent item the file was reached through if supplied, the tool call id). Reuse the existing citation id if the same file and locator were already cited in this conversation.
8. Update `last_used_at` on the `file_texts` row.
9. Return the text (or the passages), the segments covered with their citation ids, the total character count, a next-range hint when text remains, and the extraction status.

### 5.3 Download and extraction

1. Download the file content from the vault as the user, through the mechanism the target Aras version provides for authenticated file download (typically a vault server URL used with the user's token). The product assumes this typical mechanism and verifies it for each supported Aras version; the result is recorded with the list of supported versions and confirmed by the compatibility check before a pilot, and there is no code for several alternative mechanisms in the MVP. If the check fails on an installation, file reading is unavailable there and the file tools return a clear message to that effect. The download counts against the question budget's byte allowance.
2. Reject the file with FILE_UNREADABLE (reason `file_too_large`; status `too_large`) if its size exceeds `file_service.max_file_bytes`, before downloading where Aras reports the size, otherwise as soon as the limit is crossed during download.
3. Detect the format from the content signature, falling back to the file extension. A mismatch between extension and content is resolved in favour of the content.
4. Extract text in an isolated extraction step (see Security) with a timeout of `file_service.extraction_timeout_seconds`. Format-specific rules:

| Format | Segments | Extraction rules |
|---|---|---|
| PDF with text layer | One segment per page, labelled "page N" | Text in reading order as the text layer provides it. A page whose extracted text is shorter than `file_service.min_chars_per_page` is flagged as having no text. If every page is flagged, the status is `unreadable` with reason `no_text_layer`. If some pages are flagged, the status is `partial` and the flagged pages are listed. |
| Word (.docx) | One segment per heading-delimited section, labelled with the heading text; a document without headings is split into range segments | Body text, tables as tab-separated rows, headers and footers once, comments and tracked-change text excluded. |
| Excel (.xlsx) | One segment per sheet, labelled "sheet <name>"; cell ranges given in the locator when citing | Cell values row by row, tab-separated, with the cell reference of the first cell of each row. Formulas are represented by their last calculated value stored in the file; no recalculation. Hidden sheets are included and marked hidden. |
| PowerPoint (.pptx) | One segment per slide, labelled "slide N" | Slide text in shape order, then speaker notes marked as notes. |
| Plain text (.txt) | Range segments of `file_service.range_segment_chars` characters, split on line boundaries | Decoded using the detected encoding, falling back to UTF-8 with replacement characters. |
| CSV | Range segments split on row boundaries | Kept as delimited text; the header row is repeated at the start of each segment. |
| HTML | Section segments by heading where present, otherwise range segments | Visible text only; scripts, styles, and markup removed; link targets dropped. |
| XML | Range segments split on element boundaries | Kept as text including tags, because element names often carry meaning. |

The legacy binary Office formats (.doc, .xls, .ppt) are not read in the MVP: they are reported as unsupported with their detected format. Embedded objects inside Office files (for example a spreadsheet inside a Word document) are not extracted, and coverage does not mention them.

5. Normalise the text: consistent line endings, Unicode normalisation, control characters removed, runs of blank lines collapsed. Segment offsets refer to the normalised text.
6. If the extracted text exceeds `file_service.max_extracted_chars`, keep the leading part up to the limit, set the status to `partial` with reason `truncated`, and record the limit.
7. Write or replace the `file_texts` row and its `file_text_segments` rows in one transaction, with the current extractor version.
8. Failures produce a stored status (`unreadable`, `unsupported`, `too_large`, `failed`) so that the same file is not downloaded repeatedly: a stored failure skips the download until the extractor version changes (5.2). The stored success status is `ok`, which is reported as `read`. A `failed` status caused by a transient condition (timeout of the download, Aras unavailable) is not stored; only deterministic extraction failures are.

### 5.4 Fetch into Sandbox

1. Run the batched access check for all requested files.
2. For each visible file, obtain text as in steps 2 to 5 of 5.2.
3. Hand the text and the segment map to the Tool Gateway's Sandbox write channel, which writes them to the Sandbox paths and manifest entries defined in [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md#53-how-the-tool-gateway-writes-into-the-sandbox). The host-side write is the only inbound path to a Sandbox, which has no network (see [Tool Gateway](tool-gateway.md#56-writing-into-the-sandbox)).
4. Register one `file` citation per segment (page, sheet, slide, section, or range); the citation ids are written into the segment map in the Sandbox. Finer passage citations (a character range within a segment) are registered later, when the worker submits a report that points at them (see [Analysis Worker and Sandbox](analysis-worker-and-sandbox.md#55-report-validation-and-citation-registration)).
5. Return per-file statuses and a coverage summary: requested, read, partially read, unreadable (with reasons), unsupported (with formats), too large, and denied (which also covers files Aras reports as not found). Each unreadable, unsupported, or too-large file is returned with its file-level citation id; denied files carry none.

### 5.5 Coverage reporting

Every file operation returns per-file statuses in a fixed vocabulary: `read`, `partial`, `unreadable`, `unsupported`, `too_large`, `denied`, `failed`. The Main Agent and Analysis Workers carry these into the coverage section of every answer, so a user sees, for example, "12 linked reports read, 3 could not be read because they are scanned images, 1 is not available to you". The product guidance requires this; see [Skills](../skills/README.md).

Each file listed as unreadable, unsupported, or too large is given a file-level citation (file id, file name, and the parent item where known), so the answer's coverage can list it by name and the user can open it in Aras; the citation id is returned with the per-file status. For a file the user cannot access, coverage states only that a file could not be read because it is not available to the user. It gets no citation and does not reveal the file name if Aras did not return it.

### 5.6 Retention and eviction

1. Extracted text is kept while it is used. A periodic task deletes `file_texts` rows (and their segments) whose `last_used_at` is older than `file_service.retention_days_unused`.
2. Eviction never breaks citations: a citation stores the file id and locator, and the text is re-extracted on demand the next time it is opened.
3. When the extractor version is raised (for example an improved PDF extractor), stored rows whose extractor version is older than `file_service.min_extractor_version` are re-extracted on next use, whatever their status, not in bulk.
4. Extracted text is a derived store: deleting it loses nothing that cannot be regenerated from Aras. Including it in database backups is optional and chosen by the customer (see [Deployment and operations](../architecture/deployment-and-operations.md)): a backup without it is smaller and holds fewer copies of vault content, and a restore without it re-reads files through Aras, throttled. It has no separate key in the MVP; it sits in the one Database the customer controls.

## 6. Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `file_service.max_file_bytes` | integer (bytes) | 100 MB | Largest file that is downloaded and extracted |
| `file_service.max_extracted_chars` | integer | 5,000,000 | Largest extracted text kept per file; beyond this the text is truncated and marked partial |
| `file_service.read_max_chars` | integer | 20,000 | Maximum characters returned by one `file_read` call |
| `file_service.fetch_max_files` | integer | 50 | Maximum files per `file_fetch` call |
| `file_service.access_check_batch_size` | integer | 100 | Maximum file ids per batched access check |
| `file_service.extraction_timeout_seconds` | integer | 120 | Time limit for extracting one file |
| `file_service.min_chars_per_page` | integer | 20 | Below this, a PDF page is treated as having no text layer |
| `file_service.range_segment_chars` | integer | 10,000 | Size of range segments for formats without natural segments |
| `file_service.retention_days_unused` | integer (days) | 90 | Extracted text unused for this long is deleted; 0 deletes the text after each use, so every read downloads and extracts again |
| `file_service.min_extractor_version` | string | current release | Stored text from older extractor versions is re-extracted on use |
| `file_service.max_concurrent_extractions` | integer | 4 | Extraction steps running at the same time across the service |
| `file_service.enabled_formats` | list | all MVP formats | Lets an operator disable a format, for example if its extractor is not approved |

These values are starting points. Per-question file budgets and relevance ordering are tuned once the system runs against a real installation.

## 7. Limits

- MVP formats only: PDFs with a text layer, Word (.docx), Excel (.xlsx), PowerPoint (.pptx), txt, csv, html, xml. Scanned documents, images, email files (.msg, .eml), the legacy binary Office formats (.doc, .xls, .ppt), and other formats are reported as unreadable or unsupported.
- Files above `file_service.max_file_bytes` are not read (FILE_UNREADABLE, reason `file_too_large`); extracted text above `file_service.max_extracted_chars` is truncated.
- One `file_read` call returns at most `file_service.read_max_chars` characters; the model pages through longer files using the next-range hint.
- All downloads and access checks count against the question budget or job budget enforced by the Throttle; when the budget is exhausted the call returns BUDGET_EXHAUSTED.
- Embedded objects inside Office files (for example a spreadsheet embedded in a Word document) are not extracted in MVP.

## 8. Failure modes and recovery

| Failure | Detection | Effect on user | Recovery |
|---|---|---|---|
| Aras or vault unavailable | Connection error or timeout from the AML Executor or download | The tool returns ARAS_UNAVAILABLE (retryable); the answer may be delayed or report the file as not read | Retried by the model or worker within budget; nothing is stored |
| User's Aras session expired | Aras authentication error | ARAS_AUTH_EXPIRED; the Chat App asks the user to sign in again | The File Service does not retry. The turn ends `failed` with reason `auth_expired`, and after signing in the user chooses "Continue", which starts a new turn that resumes from the saved transcript; a background scan pauses until the user signs in again |
| Installation does not expose vault download as assumed | Compatibility check, or a download that cannot be made | The file tools return a clear message that files cannot be read on this installation; coverage lists the files as not read | The operator resolves it as described in [Deployment and operations](../architecture/deployment-and-operations.md) |
| Extraction crashes or hangs | Extraction step exits abnormally or exceeds timeout | File reported as unreadable with reason `extraction_failed` or `extraction_timeout` | Deterministic failures stored to avoid repeat downloads; cleared when the extractor version changes |
| Scanned PDF | All pages under the text threshold | File listed in coverage as unreadable (`no_text_layer`) | None in MVP (OCR deferred) |
| Encrypted document | Extractor reports encryption | Unreadable (`encrypted`) | None; the product does not hold document passwords |
| Format mismatch (extension says .pdf, content is an image) | Content signature check | Unsupported, with the detected format | None in MVP |
| Concurrent requests for the same new file | Single-flight lock per file id | Second request waits | Lock released on completion or failure; a stale lock expires after the extraction timeout |
| Database write fails after extraction | Transaction error | Text is still returned for this call; not stored | Next use extracts again |
| Permission revoked after extraction | Access check returns not visible | PERMISSION_DENIED; stored text not disclosed | None needed; text stays until evicted and is served only to users Aras allows |

## 9. Security considerations

- **Aras is the only access authority.** Every use, including reuse of stored text and full-text scans, is preceded by an access check made as the requesting user. There is no cache of access decisions and no privileged account.
- **No information through denial.** A denied response reveals nothing about stored text (no length, no segment names, no status other than "not available to you").
- **Untrusted content.** Documents are parsed by extractors that may have vulnerabilities. Extraction runs in a separate hardened container that uses the same runtime and hardening profile as the Sandbox, as its own instance holding no data other than the one file being extracted. It has no network access, no credentials, a read-only file system apart from a private temporary directory, and memory and time limits, so that a malicious document cannot reach Aras credentials or other users' data ([ADR 0038](../decisions/0038-sandbox-and-extraction-isolation.md)).
- **Prompt injection.** Extracted text may contain instructions aimed at the model. The tool result marks file text as untrusted content, and the product prompts instruct the agents to treat it as data (see [Prompts](../prompts/README.md)). Because the agents are read-only and every query runs as the user, injected text cannot widen access.
- **Extracted text is sensitive.** It is stored only in the product's one Database on the customer's own infrastructure. Encryption of the Database at rest is a control the customer applies (see [Security and permissions](../architecture/security-and-permissions.md#data-at-rest)); whether extracted text is included in backups is the customer's choice, described in [Deployment and operations](../architecture/deployment-and-operations.md).
- **No knowledge leakage.** Text from files is never written into shared knowledge; the Learner records how to find things, never what documents say (see [Knowledge and learning](knowledge-and-learning.md)).

## 10. Observability

- Audit events (see [Audit and progress](audit-and-progress.md#51-audit-event-catalogue)): one `file.access` event per file use, with the action recorded in its details: `check` (user, file ids, visible count, denied count), `fetch` (user, file id, bytes, duration), `extract` (file id, format, status, reason, duration, extractor version), and `serve` (user, file id, range or Sandbox id, characters returned, citation id). Writes of file text into a Sandbox are also recorded as `sandbox.write` by the Tool Gateway.
- Metrics: extraction count and duration by format and status; share of files unreadable by reason; stored-text reuse rate (requests served without download); bytes downloaded per user and per question; eviction counts; extraction queue length.
- Logs never contain extracted text or file content, only ids, sizes, statuses, and durations.
- Progress lines for long fetches ("Reading 14 linked reports, 6 done") are published through the progress channel by the calling agent, not by the File Service itself.

## 11. MVP versus deferred

| Capability | MVP | Roadmap item |
|---|---|---|
| Text PDFs, Word (.docx), Excel (.xlsx), PowerPoint (.pptx), txt, csv, html, xml | Yes | |
| Legacy binary Office formats (.doc, .xls, .ppt) | No; reported as unsupported | Can be added later without a design change |
| Access re-check through Aras on every use | Yes | |
| Extracted-text store keyed by file id | Yes | |
| OCR for scanned PDFs and images | No; reported as unreadable | [OCR for scanned documents](../product/roadmap.md) |
| Email files and other formats | No; reported as unsupported | [Email, image, and other formats](../product/roadmap.md) |
| Search index over file text (a database text index or a vector index) | No; files are searched by reading | Listed among the smaller deferrals in the [roadmap](../product/roadmap.md); revisit when repeat scans dominate the load on Aras. Any index would keep the access check on every use. The [vector index for knowledge](../product/roadmap.md) covers knowledge, not files. |
| CAD geometry | Not supported | [CAD geometry understanding](../product/roadmap.md) |
| Result caching beyond extracted text | No | [Answer and query result caching](../product/roadmap.md) |

See the [Roadmap](../product/roadmap.md) for why each item is deferred and what would trigger revisiting it.
