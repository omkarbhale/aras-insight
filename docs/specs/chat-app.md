# Chat App specification

Status: MVP specification. Related: [Main Agent](main-agent.md), [Tool Gateway](tool-gateway.md), [Background Jobs](background-jobs.md), [Knowledge and learning](knowledge-and-learning.md), [Audit and progress](audit-and-progress.md), [Persistence](persistence.md), [Security and permissions](../architecture/security-and-permissions.md), [Personas and use cases](../product/personas-and-use-cases.md).

## 1. Purpose

The Chat App is the web interface through which business users (managers, executives, engineers, and the roles around them) ask questions about their Aras data and receive cited answers. It signs users in through Aras, shows conversations, streams live progress while an answer is being built, renders citations that open the source in Aras, asks the user to confirm background jobs, captures corrections and feedback, and lets the user resolve contested knowledge.

The Chat App is a thin client. It holds no Aras credentials of its own beyond completing the sign-in exchange, runs no agent logic, and never talks to Aras for data; every question and every data access goes through the service.

## 2. Responsibilities and non-responsibilities

### Responsibilities

| Area | Responsibility |
|---|---|
| Sign-in | Start sign-in through Aras, complete the exchange, hand the resulting credential to the Tool Gateway, and keep an Aras Insight session for the browser. |
| Conversations | List the user's conversations, open one, start a new one, rename and delete them. |
| Asking | Send user messages, show the running turn, allow cancelling. |
| Live progress | Show progress lines from the Main Agent, Analysis Workers, and background jobs while a turn runs; resume the stream after reconnecting. |
| Answers | Render answer text, inline citations, assumptions, and coverage. |
| Citations | Show what each citation refers to and open the source record or file in Aras in a new tab. |
| Confirmations | Present background job proposals with their scope and estimated cost, and send confirm or decline. |
| Clarifications | Present a clarifying question with the suggested definition; present contested meanings as choices. |
| Feedback | Let the user mark an answer as helpful or not, submit a correction, and flag a specific citation as wrong. |
| Preferences | Let the user view and delete their own preferences (layer `user_preference`). |

### Non-responsibilities

| Not done here | Done by |
|---|---|
| Deciding how to answer, building queries | [Main Agent](main-agent.md) |
| Holding and refreshing Aras credentials after sign-in | [Tool Gateway](tool-gateway.md) |
| Checking permissions on data | Aras, through the [Tool Gateway](tool-gateway.md) |
| Running jobs | [Background Jobs](background-jobs.md) |
| Changing knowledge state | [Knowledge Service](knowledge-and-learning.md) |
| Administration screens (configuration, model endpoints) | Not in MVP; configuration is a deployment file ([model configuration](model-configuration.md)) |
| Export, saved questions, dashboards, Teams or Slack | Deferred ([roadmap](../product/roadmap.md)) |
| Editing anything in Aras | Not supported; read-only product |

## 3. Interfaces

### 3.1 Chat App API (Proposed shape)

The service exposes one HTTP API to the Chat App. The resource shape below is **Proposed**; transport choices (request and response over HTTPS, plus one server-to-client event stream for live updates) are technology-neutral and the concrete mechanism (for example server-sent events or a socket) is an implementation choice. All operations except sign-in require a valid Aras Insight session; all act only on resources owned by the signed-in user.

| Operation | Inputs | Outputs | Errors |
|---|---|---|---|
| Begin sign-in | optional return path | redirect target at the Aras sign-in endpoint, or a sign-in form descriptor when Aras uses login-name and password | ARAS_UNAVAILABLE |
| Complete sign-in | authorization result from Aras (or submitted credentials) | Aras Insight session (an opaque, HTTP-only cookie), user profile (display name, login name) | ARAS_AUTH_EXPIRED (code expired), PERMISSION_DENIED (Aras rejected), ARAS_UNAVAILABLE |
| Sign out | none | session revoked, Aras credential dropped by the gateway | none |
| Get session | none | user profile, credential status (valid, expiring, expired) | ARAS_AUTH_EXPIRED |
| List conversations | paging cursor, optional text filter on titles | conversations: id, title, status, last activity, whether a turn is running or waiting | none |
| Create conversation | optional first message | conversation id; if a message was given, the turn id | INVALID_INPUT |
| Get conversation | conversation id, optional cursor | turns in order: user message, outcome, answer, citations, assumptions, coverage, pending question or proposal, feedback already given | NOT_FOUND |
| Rename conversation | conversation id, title | updated conversation | NOT_FOUND, INVALID_INPUT |
| Delete conversation | conversation id | deleted; Sandbox destroyed; transcript removed per retention rules | NOT_FOUND, THROTTLED (turn running: cancel first) |
| Send message | conversation id, text, optional reply-to (clarification id, job proposal id, contested entry id with chosen meaning, or the id of a turn that failed with `auth_expired` to continue it) | turn id | INVALID_INPUT, THROTTLED, ARAS_AUTH_EXPIRED, NOT_FOUND |
| Cancel turn | turn id | turn state | NOT_FOUND |
| Subscribe to events | conversation id, last event sequence seen | stream of events (3.2) from that sequence on | NOT_FOUND |
| Get citation | citation id or citation set id | display data and the Aras link; for a set, its label, member count, and a page of member citations | NOT_FOUND |
| Confirm job | job id | job state `queued` (confirmed; it waits there while the user's or the system's running-job limit is reached) | NOT_FOUND, JOB_NOT_CONFIRMED (proposal expired, declined, or already decided; the message says which) |
| Decline job | job id | job state `declined` | NOT_FOUND |
| Cancel job | job id | job state `cancelled` | NOT_FOUND |
| List jobs | optional status filter | the user's background jobs with progress | none |
| Submit feedback | turn id, rating (helpful or not helpful), optional text, optional citation ids flagged as wrong, optional correction text | feedback id | NOT_FOUND, INVALID_INPUT |
| List preferences | none | the user's preference entries | none |
| Delete preference | entry id | deleted | NOT_FOUND |

Error responses use the shared error shape of the tool result envelope (`code`, `message`, `retryable`, optional `retry_after_ms`) defined in [tool-catalog.md](tool-catalog.md), so the app shows consistent messages.

### 3.2 Event stream

Events are ordered per conversation by a sequence number, so a client that reconnects asks for events after the last sequence it saw.

| Event | Payload | Shown as |
|---|---|---|
| `turn.started` | turn id | Running indicator on the question. |
| `progress` | source (main agent, worker id, job id), text, time | A line in the progress panel. |
| `worker.started` / `worker.finished` | worker id, objective in one line, status | A sub-section per worker in the progress panel. |
| `turn.finished` | turn id, outcome | Answer, clarification, job proposal, or failure is fetched and rendered. |
| `job.proposed` | job id, summary | Confirmation card. |
| `job.progress` | job id, done and total units, text | Progress bar on the job card. |
| `job.finished` | job id, status, result turn id if the result was posted to the conversation | Job card updated, result shown. |
| `credential.expiring` / `credential.expired` | time | Sign-in prompt. |

Progress text is produced by agents; it is rendered as plain text (never as markup), see section 9.

## 4. Data owned

| Table | Content | Schema |
|---|---|---|
| `users` | Local record of each Aras user who signed in: Aras user id, login name, display name, first and last seen. | [persistence.md#users](persistence.md#users) |
| `auth_sessions` | Aras Insight browser sessions: user, created, last activity, expiry, revoked. | [persistence.md#auth_sessions](persistence.md#auth_sessions) |
| `feedback` | Ratings, corrections, flagged citations per turn. | [persistence.md#feedback](persistence.md#feedback) |

The Chat App reads conversations, turns, citations, progress events, background jobs, and preference entries through the API; it does not own them.

## 5. Behaviour

### 5.1 Sign-in through Aras

Users sign in with their Aras identity; Aras Insight has no user store of its own. Which sign-in method Aras offers is discovered from the target installation. Recent Aras releases typically include an OAuth-based authorization server; older or differently configured installations may only accept login name and password against the Aras server. Both are verified against the target Aras version during installation.

1. The user opens the Chat App without a session and is shown a sign-in page naming the Aras installation.
2. **Proposed, preferred path (authorization server)**: the app redirects to the Aras authorization endpoint. After the user signs in there (including any single sign-on Aras is configured with), Aras returns an authorization result to the service. The service exchanges it for an access token and, where offered, a refresh token.
3. **Fallback path (login name and password)**: the app shows a form; the credentials are sent once to the service over TLS, used to obtain an Aras session or token, and never stored in plain form.
4. The service passes the token to the Tool Gateway, which stores it encrypted (see [tool-gateway.md](tool-gateway.md)) and keyed to the new Aras Insight session. The Chat App receives only an opaque session cookie; it never sees the Aras token.
5. The service upserts the `users` row from the Aras identity (Aras user id, login name, display name), creates an `auth_sessions` row, and writes a sign-in audit event.
6. Discovery runs its cheap change check as this user (see [discovery.md](discovery.md)).
7. When the Aras credential nears expiry and cannot be refreshed, the stream sends `credential.expiring`; on expiry the app shows a non-destructive sign-in prompt that returns the user to the same conversation. Running turns that need Aras end `failed` with reason `auth_expired`; after the user signs in again the app shows "Continue" on that turn, which sends a message with reply-to context naming the failed turn so the Main Agent resumes from the saved transcript ([main-agent.md](main-agent.md#515-continuing-after-sign-in)). Background jobs keep running on their own job-scoped credential ([background-jobs.md](background-jobs.md)).
8. Sign-out revokes the session, asks the gateway to drop the Aras credential, cancels any turn still running in that session (it can no longer reach Aras), and writes an audit event. Background jobs the user confirmed are not affected; they run on their job credential.

### 5.2 Conversation view

1. A sidebar lists the user's conversations, newest activity first, with a marker for "answer in progress", "waiting for you" (clarification or job confirmation), and "job running".
2. The main panel shows turns in order. Each turn shows the user's question, then either the progress panel (while running) or the result.
3. An answer is rendered in three parts:
   - The answer text, with each `[cite:<id>]` marker replaced by a small numbered reference.
   - An "Assumptions" list when the answer has any. Assumptions based on unconfirmed learned knowledge show who-taught-it provenance in general terms ("taught in an earlier conversation") and link to the knowledge citation.
   - A "Coverage" block: records considered of those the user can access, records without linked evidence, files read, files unreadable (for example scanned documents, since OCR is not yet supported) or unsupported, limits reached, time window.
4. Statements marked "unverified statement" by the Main Agent's checks are shown with a visible marker and tooltip text explaining that no source was found for them.
5. Tables and lists in the answer text are rendered; charts are out of scope in MVP unless the answer carries a table, which is shown as a table (**Proposed**; export is deferred).
6. The input box is disabled with a "Cancel" button while a turn runs in this conversation.

### 5.3 Live progress

1. While a turn runs, a progress panel under the question shows progress lines in arrival order, grouped by source: the Main Agent first, then one collapsible group per Analysis Worker with its one-line objective.
2. The panel shows elapsed time and, when the question budget is close to exhaustion, a line saying so.
3. When the answer arrives the panel collapses into a "Show steps" control, so the user can see how the answer was built afterwards.
4. Disconnects are invisible to the user apart from a brief "reconnecting" marker: the app re-subscribes with the last sequence number and replays missed events.

### 5.4 Citations that open the source in Aras

| Kind | Shown in the reference popover | Open action |
|---|---|---|
| Record | Item type label, keyed name, and the property values the answer relied on when the citation carries them | Opens the item in Aras in a new tab using the link supplied by the Citation Registry. |
| File | File name, locator (page, sheet and cell range, slide, section, or character range), a short passage excerpt, the item it was reached through | Opens the parent item or the file in Aras in a new tab. Deep linking to a page inside the file depends on the viewer Aras offers and is not guaranteed. |
| Citation set | The set's label (for example "change records closed late, last quarter") and its member count | Opens a paged list of the member records inside the Chat App; each member opens in Aras like a record citation. A set is a grouping of record citations, not a separate kind (see [tool-catalog.md](tool-catalog.md#35-citation-summaries)). |
| Knowledge | Entry title, layer and kind (discovered or learned), state, confirmation status, confidence label (high, medium, or low), one-line description; an entry that has since been retired is shown as retired, without a confidence label | Opens a read-only knowledge panel in the Chat App showing the entry body and its provenance in general terms; for a retired entry that was merged into another, a pointer to that entry. |

The Aras link is produced by the Citation Registry in the Tool Gateway from the configured installation address (`gateway.aras_base_url`) and the item's type and id; the app treats it as opaque. Opening a link in Aras is subject to Aras permissions as usual; the user only ever sees citations for data the agent read as that same user.

### 5.5 Contested-knowledge clarification

1. When a turn ends with a clarification about a contested entry, the app shows the question with one button per competing meaning (each a single line) and a free-text option "Something else".
2. Choosing a button sends a message with reply-to context naming the entry and the chosen meaning; free text is sent as an ordinary reply with the entry id attached.
3. The choice is shown in the transcript as the user's answer, so the conversation reads naturally.

An ordinary clarification (missing definition before expensive analysis) shows the question with the suggested definition as a one-click "Use this definition" button and a free-text reply.

### 5.6 Background job confirmation

1. A job proposal arrives as a turn outcome `job_proposal` with a card showing: what the job does in one sentence (for example "Search the text of all documents attached to changes in the last two years for mentions of the term"), the scope (item types, time window, estimated number of candidate files), the expected duration range, that it runs as the user with the user's permissions, and that it can be cancelled.
2. Buttons: "Run it" and "Not now". Confirming calls Confirm job; nothing runs before that call. Declining records the choice and the agent answers from what it has.
3. A proposal expires if not confirmed within a time limit (`jobs.proposal_ttl_hours`, **Proposed default** 24 hours, owned by [background-jobs.md](background-jobs.md)); confirming an expired proposal returns JOB_NOT_CONFIRMED and the app suggests asking again.
4. Once running, the card shows a progress bar and a "Cancel" button. A jobs view in the sidebar lists all the user's jobs.
5. When the job finishes, its result is posted into the originating conversation as a new turn produced by the Main Agent ([background-jobs.md](background-jobs.md)), and the conversation is marked "new answer".

### 5.7 Corrections and feedback

1. Every answer has "Helpful" and "Not helpful" controls and a "Correct this" action.
2. "Correct this" opens a short form: free text ("what is wrong or what is right"), and optionally checkboxes on the answer's citations to flag specific ones as wrong.
3. Submitting stores a `feedback` row and also sends the correction into the conversation as a user message with reply-to context, so the Main Agent responds to it in a new turn. This makes a correction a learning signal (see [main-agent.md](main-agent.md)) and, when the answer cited a learned entry, the Main Agent reports the correction to the Knowledge Service, which marks that entry uncertain ([knowledge-and-learning.md](knowledge-and-learning.md)).
4. A plain rating does not start a turn.
5. The user can also simply type a correction in the conversation; the Main Agent treats it the same way.

### 5.8 Returning to a long-running answer

1. A turn keeps running when the user closes the tab, as long as the Aras credential stays valid ([main-agent.md](main-agent.md)). Signing out is different: it drops the credential and cancels the session's running turns (5.1).
2. On return, the conversation list shows status markers; opening the conversation fetches the turns and subscribes from the last seen sequence. If the turn is still running, the progress panel shows the history so far and continues live.
3. If the answer finished while the user was away, it is shown with a "new" marker.
4. Notifications outside the app (email, chat tools) are not in MVP.

### 5.9 Preferences

A "My preferences" view lists the user's personal entries (for example "show durations in working days"), each with its one-line description and a delete action. Preferences are created by the Learner from the user's conversations; editing beyond delete is not in MVP (**Proposed**).

## 6. Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `chat_app.sign_in_method` | enum: `authorization_server`, `login_password` | `authorization_server` (Proposed default) | Which sign-in path to use; set at installation after verifying what Aras offers. |
| `chat_app.session_idle_minutes` | integer | 480 (Proposed default) | Browser session idle expiry. |
| `chat_app.session_max_hours` | integer | 24 (Proposed default) | Absolute browser session lifetime. |
| `chat_app.event_replay_limit` | integer | 2,000 (Proposed default) | Maximum events replayed on reconnect; older progress is fetched on demand. |
| `chat_app.installation_label` | string | none | Name of the Aras installation shown on the sign-in page. |

The Aras installation address used for sign-in and links is `gateway.aras_base_url` ([tool-gateway.md](tool-gateway.md)); the proposal lifetime is `jobs.proposal_ttl_hours` ([background-jobs.md](background-jobs.md)). The Chat App does not define its own copies.

## 7. Limits

| Limit | Value |
|---|---|
| Message length | `main_agent.max_message_chars` (8,000 characters, Proposed default) |
| Running turns per conversation | 1 |
| Running background jobs per user | 1; further confirmed jobs wait in `queued` (from [background-jobs.md](background-jobs.md)) |
| Open job proposals per user | `jobs.max_proposed_per_user` (3, from [background-jobs.md](background-jobs.md)) |
| Feedback text length | 4,000 characters (Proposed default) |
| Conversations listed per page | 50 (Proposed default) |

## 8. Failure modes and recovery

| Failure | Detection | Effect on user | Recovery |
|---|---|---|---|
| Aras sign-in unavailable | ARAS_UNAVAILABLE on begin or complete | Sign-in page says Aras cannot be reached | User retries; nothing is cached. |
| Aras credential expired | `credential.expired` event or ARAS_AUTH_EXPIRED | Sign-in prompt over the current view | Sign in again, then choose "Continue" on a turn that ended with `auth_expired`; it resumes from the saved transcript. |
| Event stream drops | Connection error | "Reconnecting" marker | Automatic reconnect with last sequence number. |
| Service restart | Stream drops, then events resume | Short pause | Turns resume server-side ([main-agent.md](main-agent.md#514-resumption)). |
| Citation link not openable | Aras shows an error or access denied | The Aras tab shows the problem | None; Aras permissions may have changed since the answer. |
| Proposal expired | JOB_NOT_CONFIRMED | Card says the proposal expired | Ask again. |
| Message rejected because a turn is running | THROTTLED | Input explains the wait | Wait or cancel. |

## 9. Security considerations

1. The Chat App never receives the Aras token; it holds only an opaque, HTTP-only, secure, same-site session cookie. Requests that change state carry protection against cross-site request forgery (**Proposed**: same-site cookies plus a per-session request token).
2. All content from agents, Aras records, and files is rendered as text or a restricted markup subset with links limited to the configured Aras address and the app itself; no raw HTML from answers or progress lines is ever inserted into the page. This prevents content stored in Aras from running script in the user's browser.
3. Every API operation checks that the resource belongs to the session's user. Conversation, turn, citation, job, and preference ids are not guessable, but ownership is checked regardless.
4. Login-name and password sign-in, where used, passes credentials straight to the service over TLS; the app never stores them in browser storage.
5. The app sets a restrictive content security policy and is served only over HTTPS.
6. Deleting a conversation destroys its Sandbox and removes its transcript per [persistence.md](persistence.md); audit events about it remain for the audit retention period without message content.
7. Full detail of the permission model is in [security-and-permissions.md](../architecture/security-and-permissions.md).

## 10. Observability

| Signal | Content |
|---|---|
| Audit events | Sign-in, sign-out, failed sign-in, job confirmed, declined, cancelled, feedback submitted, conversation deleted. See [audit-and-progress.md](audit-and-progress.md). |
| Metrics | Active users per day; conversations and questions per user; time from question to first progress and to answer; share of answers with feedback; helpful rate; corrections per 100 answers; citation opens per answer (citation follow-through, an ROI metric); job confirmation rate. |
| Client errors | Client-side errors reported to the service without message content (**Proposed**). |

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

## 12. Open questions

1. **Proposed**: the API resource shape and the single event stream; the concrete transport is an implementation choice.
2. **Proposed**: preferring the Aras authorization server over login-name and password; which methods are available depends on the target Aras version and must be verified.
3. **Proposed**: session lifetimes, replay limits, and feedback limits in sections 6 and 7.
4. **Proposed**: tables shown as tables, no charts in MVP. Whether simple charts are worth adding before export is decided later.
5. **Proposed**: preferences can only be deleted, not edited, in MVP.
6. **Proposed**: request-forgery protection and client error reporting approach.
7. Whether explicit "accept assumption" and "reject assumption" buttons should accompany the implicit confirmation flow described in [main-agent.md](main-agent.md).
8. **Proposed**: a "Continue" action for turns that failed with `auth_expired`, instead of keeping such turns paused until the user signs in.
