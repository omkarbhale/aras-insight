# Chat App specification

Status: MVP specification. Related: [Main Agent](main-agent.md), [Tool Gateway](tool-gateway.md), [Background Jobs](background-jobs.md), [Knowledge and learning](knowledge-and-learning.md), [Audit and progress](audit-and-progress.md), [Persistence](persistence.md), [Security and permissions](../architecture/security-and-permissions.md), [Personas and use cases](../product/personas-and-use-cases.md).

## 1. Purpose

The Chat App is the web interface through which business users (managers, executives, engineers, and the roles around them) ask questions about their Aras data and receive cited answers. It sends users to the Aras authorization server to sign in, shows conversations, streams live progress while an answer is being built, renders citations that open the source in Aras, asks the user to confirm background jobs, captures corrections and feedback, and lets the user resolve contested knowledge.

The Chat App is a thin client: it is the interface that runs in the browser and is served by the service. It never holds an Aras credential (the service's sign-in handler completes the sign-in exchange with Aras and hands the token to the Tool Gateway), runs no agent logic, and never talks to Aras for data; every question and every data access goes through the service.

## 2. Responsibilities and non-responsibilities

### Responsibilities

| Area | Responsibility |
|---|---|
| Sign-in | Redirect the browser to the Aras authorization server, and keep an Aras Insight session for the browser (an opaque cookie). The service's sign-in handler, not the browser code, receives the authorization result, exchanges it for the token, and hands the token to the Tool Gateway. |
| Conversations | List the user's conversations, open one, start a new one, rename and delete them. |
| Asking | Send user messages, show the running turn, allow cancelling. |
| Live progress | Show progress lines from the Main Agent, Analysis Workers, and background jobs while a turn runs; resume the stream after reconnecting. |
| Answers | Render answer text, inline citations, assumptions, and coverage. |
| Citations | Show what each citation refers to and open the source record or file in Aras in a new tab. |
| Confirmations | Show each background job proposal as a card built from the structured proposal (scope and estimate), and send confirm, decline, or cancel. |
| Clarifications | Present a clarifying question with the suggested definition; present contested meanings as choices. |
| Feedback | Let the user mark an answer as helpful or not, submit a correction, and flag a specific citation as wrong. |
| Preferences | Let the user view and delete their own preferences (layer `user_preference`). |

### Non-responsibilities

| Not done here | Done by |
|---|---|
| Deciding how to answer, building queries | [Main Agent](main-agent.md) |
| Obtaining the Aras token at sign-in | The service's sign-in handler |
| Holding and refreshing Aras credentials after sign-in | [Tool Gateway](tool-gateway.md) |
| Checking permissions on data | Aras, through the [Tool Gateway](tool-gateway.md) |
| Running jobs | [Background Jobs](background-jobs.md) |
| Changing knowledge state | [Knowledge Service](knowledge-and-learning.md) |
| Administration screens (configuration, model endpoints) | Not in MVP; configuration is a deployment file ([model configuration](model-configuration.md)) |
| Export, saved questions, dashboards, Teams or Slack | Deferred ([roadmap](../product/roadmap.md)) |
| Editing anything in Aras | Not supported; read-only product |

## 3. Interfaces

### 3.1 Chat App API

The service exposes one HTTP API to the Chat App. Requests and responses go over HTTPS, and one server-to-client event stream per conversation carries live updates; the browser sends everything else through ordinary requests, so a one-way stream is enough. The contract that matters is ordered events with sequence numbers and replay after a reconnect (3.2); the concrete mechanism for the stream (server-sent events or a WebSocket) is an implementation choice. All operations except sign-in require a valid Aras Insight session; all act only on resources owned by the signed-in user.

| Operation | Inputs | Outputs | Errors |
|---|---|---|---|
| Begin sign-in | optional return path | redirect target at the Aras authorization endpoint | ARAS_UNAVAILABLE |
| Complete sign-in (the service's sign-in callback, reached by the browser's redirect back from Aras) | authorization result from Aras | Aras Insight session (an opaque, HTTP-only cookie), user profile (display name, login name); the service holds the token | ARAS_AUTH_EXPIRED (code expired), PERMISSION_DENIED (Aras rejected), ARAS_UNAVAILABLE |
| Sign out | none | session revoked, Aras credential dropped by the gateway | none |
| Get session | none | user profile, credential status (valid, expiring, expired), schema status (`preparing`, `partial`, `ready`; see 5.10) | ARAS_AUTH_EXPIRED |
| List conversations | paging cursor, optional text filter on titles | conversations: id, title, status, last activity, whether a turn is running or waiting | none |
| Create conversation | optional first message | conversation id; if a message was given, the turn id | INVALID_INPUT |
| Get conversation | conversation id, optional cursor | turns in order: user message, outcome, answer, citations, assumptions, coverage, pending question or proposal, feedback already given | NOT_FOUND |
| Rename conversation | conversation id, title | updated conversation | NOT_FOUND, INVALID_INPUT |
| Delete conversation | conversation id | deleted; Sandbox destroyed; transcript removed per retention rules | NOT_FOUND, TURN_IN_PROGRESS (turn running: cancel first) |
| Send message | conversation id, text, the browser's time zone (an IANA name, see 5.2), optional reply-to (clarification id, contested entry id with chosen meaning, the id of an answer being corrected, or the id of a turn that failed with `auth_expired` to continue it) | turn id | INVALID_INPUT, TURN_IN_PROGRESS (a turn is already running in this conversation; there is no queue), SCHEMA_NOT_READY, ARAS_AUTH_EXPIRED, NOT_FOUND |
| Cancel turn | turn id | turn state | NOT_FOUND |
| Subscribe to events | conversation id, last event sequence seen | stream of events (3.2) from that sequence on | NOT_FOUND |
| Get progress | turn id, optional cursor | the stored progress lines of that turn in order (used by "Show steps" and for events older than the replay limit) | NOT_FOUND |
| Get citation | citation id or citation set id | display data and the Aras link; for a set, its label, member count, and a page of member citations | NOT_FOUND |
| Record citation open | citation id | recorded: the citation's opened count goes up by one and its first-opened time is set if empty; no per-click record is kept | NOT_FOUND |
| Confirm job | job id | job state `queued` (confirmed; it waits there while the user's or the system's running-job limit is reached). No turn is started. | NOT_FOUND, JOB_NOT_CONFIRMED (proposal expired, declined, or already decided; the message says which) |
| Decline job | job id | job state `declined`. No turn is started. | NOT_FOUND |
| Cancel job | job id | job state `cancelled`; matches found so far are kept (5.6) | NOT_FOUND |
| List jobs | optional status filter | the user's background jobs with progress | none |
| Submit feedback | turn id, rating (helpful or not helpful), optional text, optional citation ids flagged as wrong, optional correction text | feedback id; when correction text is given, also the id of the turn the correction starts (5.7) | NOT_FOUND, INVALID_INPUT, TURN_IN_PROGRESS (correction text given while a turn is running) |
| List preferences | none | the user's preference entries | none |
| Delete preference | entry id | deleted | NOT_FOUND |
| Report client error | error kind, application version, view name, time, code location (no message content) | accepted | INVALID_INPUT |

Error responses use the shared error shape of the tool result envelope (`code`, `message`, `retryable`, optional `retry_after_ms`) defined in [tool-catalog.md](tool-catalog.md), so the app shows consistent messages. A second message in a conversation whose turn is still running is rejected with its own code, TURN_IN_PROGRESS, so that THROTTLED keeps a single meaning (a concurrency or rate limit with a wait time); the user waits or cancels ([ADR 0048](../decisions/0048-one-running-turn-per-conversation.md)).

### 3.2 Event stream

Events are ordered per conversation by a sequence number, so a client that reconnects asks for events after the last sequence it saw. At most `chat_app.event_replay_limit` events are replayed; when more were missed, the app reads the stored turns and progress lines through Get conversation and Get progress instead.

| Event | Payload | Shown as |
|---|---|---|
| `turn.started` | turn id | Running indicator on the question. |
| `progress` | source (main agent, worker id, job id, budget, system), text, time | A line in the progress panel. |
| `worker.started` / `worker.finished` | worker id, objective in one line, status | A sub-section per worker in the progress panel. |
| `turn.finished` | turn id, outcome | Answer, clarification, job proposal, or failure is fetched and rendered. |
| `job.proposed` | job id, summary | Confirmation card. |
| `job.progress` | job id, done and total units, text | Progress bar on the job card. |
| `job.finished` | job id, status (`completed`, `failed`, or `cancelled`), result turn id if the result was posted to the conversation | Job card updated, result shown. |
| `credential.expiring` / `credential.expired` | time | Sign-in prompt. |

Progress text is produced by agents; it is rendered as plain text (never as markup), see section 9. A progress line is at most `progress.max_text_chars` (300) characters, and a source instance (the Main Agent, each worker, each job) publishes at most one line per `progress.min_interval_ms` (1,000 ms); lines beyond that are coalesced, keeping the latest, so parallel workers do not swallow each other's lines. Progress lines come from the Main Agent, Analysis Workers (through `report_progress`), Background Jobs, the Throttle (budget lines, through the Tool Gateway), and the service itself; the limits are defined in [audit-and-progress.md](audit-and-progress.md).

## 4. Data owned

| Table | Content | Schema |
|---|---|---|
| `users` | Local record of each Aras user who signed in: Aras user id, login name, display name, first and last seen. | [persistence.md#users](persistence.md#users) |
| `auth_sessions` | Aras Insight browser sessions: user, a hash of the session cookie value, created, last activity, expiry, revoked. | [persistence.md#auth_sessions](persistence.md#auth_sessions) |
| `feedback` | Ratings, corrections, flagged citations per turn. | [persistence.md#feedback](persistence.md#feedback) |

The Chat App reads conversations, turns, citations, progress events, background jobs, and preference entries through the API; it does not own them.

## 5. Behaviour

### 5.1 Sign-in through Aras

Users sign in with their Aras identity through the Aras installation's own authorization server, which is the only sign-in path ([ADR 0033](../decisions/0033-sign-in-through-the-aras-authorization-server.md)). Aras Insight has no user store of its own and no login form of its own: the browser is redirected to Aras, the user signs in there (including any single sign-on Aras is configured with), and the service obtains the token. Aras Insight never sees the user's password. An installation without an authorization server is not supported in the MVP; an authorization server with a sign-in client registered for Aras Insight is a deployment prerequisite, and it is one of the compatibility checks run at installation (see [deployment-and-operations.md](../architecture/deployment-and-operations.md)).

1. The user opens the Chat App without a session and is shown a sign-in page naming the Aras installation (`chat_app.installation_label`) with one action, "Sign in with Aras".
2. The app redirects the browser to the Aras authorization endpoint. After the user signs in there, Aras redirects the browser back to the service's sign-in callback with an authorization result. The service's sign-in handler exchanges it, on the service side, for an access token and, where offered, a refresh token.
3. The sign-in handler passes the token to the Tool Gateway, which stores it encrypted (see [tool-gateway.md](tool-gateway.md)) and keyed to the new Aras Insight session. The browser receives only an opaque session cookie; the Chat App never holds the Aras token.
4. The service upserts the `users` row from the Aras identity (Aras user id, login name, display name), creates an `auth_sessions` row, and writes a sign-in audit event. No identity groups are read or recorded; usage is reported per user and in total ([ADR 0039](../decisions/0039-audit-and-usage-records.md)).
5. Discovery runs its cheap change check as this user (see [discovery.md](discovery.md)).
6. When the Aras credential nears expiry and cannot be refreshed, the stream sends `credential.expiring`; on expiry the app shows a non-destructive sign-in prompt that returns the user to the same conversation. Running turns that need Aras end `failed` with reason `auth_expired`; after the user signs in again the app shows "Continue" on that turn, which sends a message with reply-to context naming the failed turn so the Main Agent resumes from the saved transcript ([main-agent.md](main-agent.md#515-continuing-after-sign-in)). The failed turn is not held paused: it holds no agent loop, Sandbox, or budget while the user is away. Where the installation issues a renewable grant for a background job, the job keeps running on its own job-scoped credential; where it does not, the job uses the user's session credential and pauses when that session ends ([background-jobs.md](background-jobs.md), [ADR 0035](../decisions/0035-job-credential-lifecycle.md)).
7. A running turn counts as session activity, so the idle expiry (`chat_app.session_idle_minutes`) never cuts a running turn; only the absolute lifetime (`chat_app.session_max_hours`) ends the session. If that happens during a turn, the Aras credential is dropped with the session, the turn ends `failed` with reason `auth_expired`, and after signing in again the user chooses "Continue" as in step 6.
8. Sign-out revokes the session, asks the gateway to drop the Aras credential, cancels any turn still running in that session (it can no longer reach Aras), and writes an audit event. A background job the user confirmed keeps running where it has its own job-scoped credential (step 6); a job that uses the session credential pauses.

### 5.2 Conversation view

1. A sidebar lists the user's conversations, newest activity first, with a marker for "answer in progress", "waiting for you" (clarification or job confirmation), and "job running".
2. The main panel shows turns in order. Each turn shows the user's question, then either the progress panel (while running) or the result.
3. An answer is rendered in three parts:
   - The answer text, with each `[cite:<id>]` marker replaced by a small numbered reference. The agents write the marker right after the statement it supports, and several adjacent markers stand for several sources; the id may name a record, file, knowledge, or citation-set citation.
   - An "Assumptions" list when the answer has any. Assumptions based on unconfirmed learned knowledge show who-taught-it provenance in general terms ("taught in an earlier conversation") and link to the knowledge citation; the teacher's name is shown only when the deployment setting `knowledge.show_teacher_names` is on (off by default). There are no accept or reject buttons: the user confirms an assumption by carrying on and objects by saying so in the conversation, and the Learner judges the reply ([ADR 0042](../decisions/0042-assumption-disclosure-and-independent-confirmation.md)).
   - A "Coverage" block: records considered of those the user can access, records without linked evidence, files read, files unreadable (for example scanned documents, since OCR is not yet supported) or unsupported, limits reached, time window.
4. Statements marked "unverified statement" by the Main Agent's answer check (the structural checks and the model-based check; [ADR 0047](../decisions/0047-final-answer-check.md)) are shown with a visible marker and tooltip text explaining that no source was found for them.
5. Tables and lists in the answer text are rendered. Charts are not shown in the MVP; tables are rendered as tables. Charts and downloads come together with the export item on the [roadmap](../product/roadmap.md).
6. The input box is disabled with a "Cancel" button while a turn runs in this conversation (a second message is rejected with TURN_IN_PROGRESS; there is no queue).
7. Relative dates in a question ("last quarter") are computed in the user's time zone. The app sends the browser's time zone (an IANA name such as `Europe/Berlin`) with each message; a time-zone preference the user has taught (5.9) takes precedence, and the deployment's default time zone applies when the browser sends none.

### 5.3 Live progress

1. While a turn runs, a progress panel under the question shows progress lines in arrival order, grouped by source: the Main Agent first, then one collapsible group per Analysis Worker with its one-line objective.
2. The panel shows elapsed time and, when the question budget is close to exhaustion, a line saying so.
3. When the answer arrives the panel collapses into a "Show steps" control, so the user can see how the answer was built afterwards (the stored lines are read through Get progress).
4. Disconnects are invisible to the user apart from a brief "reconnecting" marker: the app re-subscribes with the last sequence number and replays missed events.

### 5.4 Citations that open the source in Aras

| Kind | Shown in the reference popover | Open action |
|---|---|---|
| Record | Item type label, keyed name, and the property values the answer relied on when the citation carries them | Opens the item in Aras in a new tab using the link supplied by the Citation Registry. |
| File | File name, locator (page, sheet and cell range, slide, section, or character range), a short passage excerpt, the item it was reached through | Opens the parent item or the file in Aras in a new tab. Deep linking to a page inside the file depends on the viewer Aras offers and is not guaranteed. |
| Citation set | The set's label (for example "change records closed late, last quarter") and its member count | Opens a paged list of the member records inside the Chat App; each member opens in Aras like a record citation. A set is a grouping of record citations, not a separate kind (see [tool-catalog.md](tool-catalog.md#35-citation-summaries)). |
| Knowledge | Entry title, layer and kind (discovered or learned), state, confirmation status, confidence label (high, medium, or low), one-line description; an entry that has since been retired is shown as retired, without a confidence label | Opens a read-only knowledge panel in the Chat App showing the entry body and its provenance in general terms (the teacher's name only when `knowledge.show_teacher_names` is on); for a retired entry that was merged into another, a pointer to that entry. |

The confidence label is derived when the entry is read: high for product skills, mechanical discovered entries in the confirmed state, and learned entries that are confirmed and independently confirmed; medium for model-inferred discovered entries in the confirmed state and for learned entries confirmed but not independently confirmed; low for uncertain or contested entries. Personal preferences and retired entries carry no label ([knowledge-and-learning.md](knowledge-and-learning.md)).

The Aras link is produced by the Citation Registry in the Tool Gateway from the configured installation address (`gateway.aras_base_url`) and the item's type and id, and file links go through the parent item; the app treats the link as opaque. The link form is the Aras client's standard deep-link form for the target version, and it is verified for each supported Aras version as one of the compatibility checks ([deployment-and-operations.md](../architecture/deployment-and-operations.md)). Opening a link in Aras is subject to Aras permissions as usual; the user only ever sees citations for data the agent read as that same user. Each time the user opens a citation, the app calls Record citation open, which raises a counter on the citation and keeps no record of who opened what ([ADR 0039](../decisions/0039-audit-and-usage-records.md)).

### 5.5 Contested-knowledge clarification

1. When a turn ends with a clarification about a contested entry, the app shows the question with one button per competing meaning (two or more, each a single line) and a free-text option "Something else". The question names the meanings and asks the user to choose a button or describe the meaning in words; it does not ask for a typed number ([ADR 0043](../decisions/0043-contested-knowledge.md)).
2. Choosing a button sends a message with reply-to context naming the entry and the chosen meaning; free text is sent as an ordinary reply with the entry id attached, and a meaning the user describes that is not among the buttons is added as a further meaning.
3. The choice is shown in the transcript as the user's answer, so the conversation reads naturally.

An ordinary clarification (missing definition before expensive analysis) shows the question with the suggested definition as a one-click "Use this definition" button and a free-text reply.

### 5.6 Background job confirmation

1. A question that has no starting item is first answered from the cheap search (names, titles, descriptions, file names). That interim answer is shown in the same turn as the proposal, so the user has something even if they decline. The proposal arrives as a turn outcome `job_proposal`, and the Chat App builds the confirmation card from the structured proposal, not from text the Main Agent wrote, so the numbers pass from the tool to the user without a model rewriting them ([ADR 0050](../decisions/0050-background-job-flow.md)). The card shows: a heading (the proposal's title) and one sentence on what the job does (for example "Search the text of all documents attached to changes in the last two years for mentions of the term"); the search terms; the scope (item types, time window, file selection); the estimate, which is the number of candidate files and how many of them already have stored text, the expected duration range (always present), and the approximate number of Aras queries the scan will make; a fixed note that files in unreadable formats are skipped and reported in the coverage; and that the scan runs as the user with the user's permissions, in small batches at low priority, and can be cancelled. The estimate carries no download volume.
2. Buttons: "Confirm" and "Decline". Confirming calls Confirm job; nothing runs before that call, and a typed reply such as "yes" does not confirm. Confirm, Decline, and Cancel only change the job's state, which the card shows; none of them starts a Main Agent turn. Declining records the choice and nothing else happens, because the user already has the interim answer.
3. A proposal expires if not confirmed within a time limit (`jobs.proposal_ttl_hours`, default 24 hours, owned by [background-jobs.md](background-jobs.md)); confirming an expired proposal returns JOB_NOT_CONFIRMED and the app suggests asking again.
4. Once running, the card shows a progress bar and a "Cancel" button. Cancelling keeps what was found so far. A jobs view in the sidebar lists all the user's jobs.
5. When the job finishes, its result is posted into the originating conversation and the Main Agent starts a new turn on it automatically: it reads the matched passages and writes a cited answer that states its coverage, and the conversation is marked "new answer". This happens even if the user is away, so the finished answer is waiting on return; the model cost is bounded by the per-turn budget. A job cancelled after reading some files goes through the same path and produces a partial answer that says how many files were read and which were not searched ([background-jobs.md](background-jobs.md)).

### 5.7 Corrections and feedback

1. Every answer has "Helpful" and "Not helpful" controls and a "Correct this" action.
2. "Correct this" opens a short form: free text ("what is wrong or what is right"), and optionally checkboxes on the answer's citations to flag specific ones as wrong.
3. Submitting calls Submit feedback, which stores a `feedback` row and, when the form carries correction text, also creates the turn in the same call: the correction is sent into the conversation as a user message whose reply-to context is the answer being corrected, the Main Agent responds to it in that new turn, and the call returns the turn id. This makes a correction a learning signal (see [main-agent.md](main-agent.md)) and, when the answer cited a learned entry, the Main Agent reports the correction to the Knowledge Service, which marks that entry uncertain ([knowledge-and-learning.md](knowledge-and-learning.md)). A correction is not a confirmation or an objection to an assumption; those are judged from the user's ordinary replies (5.2).
4. A plain rating does not start a turn.
5. The user can also simply type a correction in the conversation. Send message accepts the reply-to kind "correction of turn" (the answer being corrected), and the Main Agent treats such a message the same way as one from the form.

### 5.8 Returning to a long-running answer

1. A turn keeps running when the user closes the tab, as long as the Aras credential stays valid ([main-agent.md](main-agent.md)); a running turn counts as session activity, so the idle expiry does not end the session under it. Signing out is different: it drops the credential and cancels the session's running turns, and the absolute session lifetime ends a running turn with `auth_expired` so that the user can continue it after signing in again (5.1).
2. On return, the conversation list shows status markers; opening the conversation fetches the turns and subscribes from the last seen sequence. If the turn is still running, the progress panel shows the history so far and continues live.
3. If the answer finished while the user was away, it is shown with a "new" marker.
4. Notifications outside the app (email, chat tools) are not in the MVP; they are on the [roadmap](../product/roadmap.md).

### 5.9 Preferences

A "My preferences" view lists the user's personal entries (for example "show durations in working days", or the time zone to use for relative dates), each with its one-line description and a delete action. Preferences are created by the Learner from the user's conversations; editing beyond delete is not in the MVP: a preference is corrected by telling the agent again ([ADR 0045](../decisions/0045-personal-preferences-layer.md)).

### 5.10 Before the schema is ready

On the first sign-in to an installation, the first discovery run has not finished. The Get session operation reports the schema status, and the app re-reads it while it is not `ready`.

1. `preparing`: Discovery has not yet signalled "Schema ready" (item types, properties, relationship types, and lists stored). The app shows that the product is preparing and disables the input box. A message sent anyway is rejected with the dedicated code SCHEMA_NOT_READY, which is not ARAS_UNAVAILABLE (that code tells the agent to retry). If initial discovery fails, the app says that the product is preparing and cannot answer yet, with the reason.
2. `partial`: "Schema ready" has been signalled and questions are allowed, while lifecycle maps, workflow maps, forms, methods, and generated knowledge are still loading. The app shows a notice that the rest of the configuration is still loading. Query-analysis rules that need types not yet loaded (lifecycle state values, read-time method warnings) are skipped, and the answer's coverage says so ([discovery.md](discovery.md)).
3. `ready`: the notice disappears.

## 6. Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `chat_app.session_idle_minutes` | integer | 480 | Browser session idle expiry; a running turn counts as activity. |
| `chat_app.session_max_hours` | integer | 24 | Absolute browser session lifetime. |
| `chat_app.event_replay_limit` | integer | 2,000 | Maximum events replayed on reconnect; older progress is read with Get progress. |
| `chat_app.installation_label` | string | none | Name of the Aras installation shown on the sign-in page. |

The Aras installation address used for sign-in and links is `gateway.aras_base_url` ([tool-gateway.md](tool-gateway.md)); the proposal lifetime is `jobs.proposal_ttl_hours` ([background-jobs.md](background-jobs.md)). The Chat App does not define its own copies.

## 7. Limits

| Limit | Value |
|---|---|
| Message length | `main_agent.max_message_chars` (default 8,000 characters) |
| Running turns per conversation | 1; a second message is rejected with TURN_IN_PROGRESS |
| Running background jobs per user | 1; further confirmed jobs wait in `queued` (from [background-jobs.md](background-jobs.md)) |
| Open job proposals per user | `jobs.max_proposed_per_user` (3, from [background-jobs.md](background-jobs.md)) |
| Feedback text length | 4,000 characters |
| Conversations listed per page | 50 |

## 8. Failure modes and recovery

| Failure | Detection | Effect on user | Recovery |
|---|---|---|---|
| Aras sign-in unavailable | ARAS_UNAVAILABLE on begin or complete | Sign-in page says Aras cannot be reached | User retries; nothing is cached. |
| Aras credential expired | `credential.expired` event or ARAS_AUTH_EXPIRED | Sign-in prompt over the current view | Sign in again, then choose "Continue" on a turn that ended with `auth_expired`; it resumes from the saved transcript. |
| Event stream drops | Connection error | "Reconnecting" marker | Automatic reconnect with last sequence number. |
| Browser session reaches its absolute lifetime during a turn | Session expiry | Sign-in prompt; the turn shows as failed with `auth_expired` | Sign in again, then choose "Continue". |
| Service restart | Stream drops, then events resume | Short pause | Turns resume server-side ([main-agent.md](main-agent.md#514-resumption)); the MVP runs one active service instance, so no turn has to be taken over by another instance ([ADR 0037](../decisions/0037-single-instance-mvp-and-optional-gateway-process.md)). |
| Citation link not openable | Aras shows an error or access denied | The Aras tab shows the problem | None; Aras permissions may have changed since the answer. |
| Proposal expired | JOB_NOT_CONFIRMED | Card says the proposal expired | Ask again. |
| Message rejected because a turn is running | TURN_IN_PROGRESS | Input explains the wait | Wait or cancel. |
| Message sent before the schema is ready | SCHEMA_NOT_READY | The app shows that the product is preparing | Wait until the schema status is `partial` or `ready` (5.10). |

## 9. Security considerations

1. The Chat App never receives the Aras token; it holds only an opaque, HTTP-only, secure, same-site session cookie. The cookie value is a random reference that the service looks up on every request (only a hash of it is stored), not a signed token, so revoking a session takes effect at once. Requests that change state carry protection against cross-site request forgery: same-site cookies plus a per-session request token.
2. All content from agents, Aras records, and files is rendered as text or a restricted markup subset with links limited to the configured Aras address and the app itself; no raw HTML from answers or progress lines is ever inserted into the page. This prevents content stored in Aras from running script in the user's browser.
3. Every API operation checks that the resource belongs to the session's user. Conversation, turn, citation, job, and preference ids are not guessable, but ownership is checked regardless.
4. Aras Insight never sees the user's password: sign-in happens on the Aras authorization server, to which the browser is redirected. The service receives only the authorization result and the tokens Aras issues for it, and the browser keeps neither a password nor an Aras token in its storage.
5. The app sets a restrictive content security policy and is served only over HTTPS.
6. Deleting a conversation destroys its Sandbox and removes its transcript per [persistence.md](persistence.md); audit events about it remain for the audit retention period without message content.
7. Full detail of the permission model is in [security-and-permissions.md](../architecture/security-and-permissions.md).

## 10. Observability

| Signal | Content |
|---|---|
| Audit events | Sign-in, sign-out, failed sign-in, job confirmed, declined, cancelled, feedback submitted, conversation deleted. See [audit-and-progress.md](audit-and-progress.md). |
| Metrics | Active users per day; conversations and questions per user; time from question to first progress and to answer; share of answers with feedback; helpful rate; corrections per 100 answers; citation opens per answer (citation follow-through, an ROI metric, computed from the opened count kept on each citation; there is no per-click log); job confirmation rate. Usage is reported per user and in total; identity groups are not recorded. |
| Client errors | The browser reports client-side errors to the service through Report client error: error kind, application version, view name, time, and code location, never message content, record values, or file text. The service writes them to its operational log. |

## 11. MVP versus deferred

| In MVP | Deferred |
|---|---|
| Web app with sign-in through Aras | Teams and Slack integration |
| Conversations, live progress, citations, coverage | Export of answers, tables, and charts |
| Background job confirmation | Saved questions, dashboards, scheduled reports |
| Corrections, ratings, contested clarification | Suggested questions per role |
| Preference view and delete | Usage and ROI dashboard |
| Returning to long-running answers | Notifications outside the app |

See the [roadmap](../product/roadmap.md).
