# Executive summary

## The problem

Product data lives in Aras: parts, documents, change orders, problem reports, workflows, approvals, and the files attached to them. The answers leaders and engineers need are in there, but getting them out is slow. A simple question needs someone who knows the data model. An analytical question, such as where change approvals get stuck or which components keep coming back in problem reports, becomes a multi-day exercise of exports, spreadsheets, and manual reading. The reasons behind the numbers sit in free-text fields, sign-off comments, and attached reports that no standard report can summarise. Decisions are made late, or without the evidence.

## The proposed product

Aras Insight is an assistant that answers plain-language questions about our Aras data. It finds the relevant records, reads the relevant files, computes the numbers, and returns an answer that cites every source it used. It runs on our own servers, uses our own AI model endpoints, and works only through Aras, as the person asking.

## What makes it different

| Property | What it means for us |
|---|---|
| Permission-faithful | Every query runs through the Aras server as the signed-in user. Nobody can see anything through Aras Insight that they cannot already see in Aras. There is no service account with wider access. |
| Cited and verifiable | Every answer links to the Aras records, file passages, and knowledge it used, and states its coverage: how many records it looked at, what had no evidence, and what it could not read. |
| Analytics across records and files | Numbers are computed with code over the actual data, never estimated by the model. The same answer can combine structured records with the text of linked reports. |
| Self-hosted, our models | It runs on our infrastructure, with model endpoints we choose and can switch. No data leaves our environment unless we point it at an external model. |
| Learns our installation | It discovers our Aras configuration automatically and learns what our terms and processes mean from use. Nothing is configured by hand per installation. |
| Read-only | It never creates, edits, or deletes anything in Aras. |

## What the MVP delivers

- Plain-language lookups and analytical questions (counts, rankings, trends, durations) for business users in every role.
- "Why" questions answered from description fields, workflow sign-off comments, and linked files (PDFs with a text layer, Word, Excel, PowerPoint, plain text).
- Process analysis: how real items moved through workflows and lifecycles compared with the defined process, and where they waited.
- Citations and coverage on every answer, and stated assumptions where a term had to be interpreted.
- Protection of the production Aras server: every query is checked for cost before it runs, and per-user and system-wide limits apply.
- Long-running questions show live progress, and sessions are saved so users can leave and come back.
- A full-text search across file contents, run as a background job that the user confirms first.

## What is deferred

Writing to Aras, OCR for scanned documents, email and image files, CAD geometry, result caching, an analytics path that avoids the production server, sources outside Aras, chat integrations such as Teams or Slack, export, saved questions and dashboards, scheduled reports, and an ROI dashboard. Each deferred item, the reason, and the trigger for revisiting it are in the [roadmap](product/roadmap.md).

## Main risks

| Risk | Mitigation |
|---|---|
| Wrong or invented answers | Citations on every claim, numbers computed by code, query checking against the real schema, coverage reporting, and an explicit rule against claiming causation. |
| Load on the production Aras server | Static cost analysis of every query, throttling, row budgets, timeouts, and background jobs that run batched and throttled. |
| Data exposure | Access only as the signed-in user, an isolated Sandbox with no network, and self-hosting. Learned knowledge holds no business data values. |
| Model quality varies | Three model slots (main agent, analysis worker, background work) are configured separately, so the strongest model can be used where it matters, and a pilot measures quality on the actual models. |
| Scanned documents are unreadable in the MVP | They are reported in coverage, so answers state the gap instead of hiding it. OCR is on the roadmap. |

The full register is in [risks](business/risks.md).

## What a pilot needs from the company

- A non-production Aras environment that mirrors production configuration, or agreement to run against production with conservative throttling, plus network access from the Aras Insight server to it.
- A server or virtual machine able to run containers, and a PostgreSQL database.
- Access to one or more model endpoints the company approves (internal or a private cloud deployment).
- Security review of the deployment and of the data flow.
- Ten to twenty pilot users across several roles, each with their normal Aras account, and a few hours of their time.
- An agreed set of acceptance questions and success criteria.

The phases, acceptance questions, and exit criteria are in the [pilot plan](business/pilot-plan.md). The business case and how value is measured are in [value and ROI](business/value-and-roi.md).
