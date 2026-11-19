# Value and ROI

This document sets out the business case for Aras Insight, the value drivers behind it, how each return metric is defined and measured from the product's own records, and the costs a company carries. It gives no headline ROI figure: the figures depend on the company, and the point of a [pilot](pilot-plan.md) is to measure them.

## The business case in brief

Companies already pay for the data in Aras: the licences, the administration, and the effort of everyone who records changes, approvals, and reports. Much of the return on that investment is lost when the data is hard to question. Analysts and power users spend their time answering other people's data questions; managers commission reports that are stale on arrival; engineers search change history and attachments by hand; decisions wait for evidence or proceed without it.

Aras Insight turns that stored data into answers on demand, at each user's own access level, with citations that make the answers usable for decisions. It keeps no copy or history of business data beyond conversation transcripts and the extracted text of files that users have read (both in the customer's own database), adds no new access policy, and has no write path into Aras, which keeps the cost of adopting it and the risk of running it low.

## Value drivers

| Driver | What changes | Who benefits |
|---|---|---|
| Time to an answer | A question that took hours or days (find the data, export, analyse, read attachments) is answered in one conversation, typically in minutes. | Everyone who asks data questions. |
| Reports and manual analysis avoided | Ad hoc report requests, spreadsheet analyses, and manual reading of attached documents are replaced by questions. | Analysts, Aras administrators, and power users, whose time is freed. |
| Decision latency | Decisions that waited for an analysis can be made the same day, with evidence attached. | Executives, program managers, change boards. |
| Onboarding | New employees ask about the installation's vocabulary and process in plain language and learn from cited answers. | New hires and their mentors. |
| Auditability | Every answer is traceable to records and file passages, and every query is recorded against the user it ran for. | Compliance, quality, and security functions. |
| Process visibility | Deviations and bottlenecks in workflows become measurable on demand rather than through a dedicated study. | Change and configuration management, engineering leadership. |

## ROI metrics

Each metric below is computed from records the product already keeps: sessions and transcripts, the audit record of every query run on a user's behalf, Citation Registry entries, job records, and usage records of model calls (see [audit and progress](../specs/audit-and-progress.md) and [persistence](../specs/persistence.md)). A dashboard presenting them is deferred ([roadmap](../product/roadmap.md)); in the MVP they are produced from the database by query or report.

The metric definitions are **Proposed**.

| Metric | Definition | How it is measured |
|---|---|---|
| Questions answered | Number of user questions that received a final answer, per period, per role. | Count of completed answer turns in session records; role from the user's Aras identity. |
| Active users | Distinct users who asked at least one question per week or month. | Distinct user ids across sessions per period. |
| Repeat use | Share of active users who return in a later period. | Cohort comparison of active users across periods. |
| Time to answer | Elapsed time from question to final answer, by question type (lookup or analysis). | Timestamps of the question and the final answer in session records; type from whether an Analysis Worker ran. |
| Estimated time saved | Sum, over answered questions, of a baseline manual effort for that question type minus the time to answer. | Time to answer from records; baseline per question type established in the pilot by timing or surveying how long the same questions took before. |
| Analysis depth | Records examined and files read per answer. | Row counts and file counts from audit records per answer. |
| Citation follow-through | Share of answers where the user opened at least one cited record or file. | Citation clicks recorded by the Chat App against Citation Registry ids. |
| Correction rate | Share of answers followed by a user correction. | Corrections detected by the Learner (one of its learning signals) per answered question. |
| Clarification rate | Share of analytical questions that needed a clarifying question. | Clarifying-question turns in session records. |
| Coverage quality | Share of examined files that were readable; share of items that had linked evidence. | Coverage sections of answer reports. |
| Knowledge growth and health | Learned entries per state (confirmed, uncertain, contested, retired) over time. | Knowledge Service records. |
| Load on Aras | Queries and rows fetched per period; queries rejected or deferred by the Throttle; queries rejected by cost analysis. | Audit and throttle records. |
| Cost per answer | Model usage cost and infrastructure share per answered question. | Token usage per model slot from usage records, multiplied by the customer's own rates, plus allocated infrastructure cost. |

A simple ROI view for a period is then: estimated time saved multiplied by a loaded hourly cost, divided by the product's cost for the period (below). Companies may add the value of faster decisions or avoided rework where they can attribute it; those are harder to measure and should be reported separately from time saved.

## The cost side

Because Aras Insight is self-hosted, most running costs are the customer's own infrastructure and model usage. Any licence or subscription cost depends on the pricing decision, which is open; see [pricing options](pricing-options.md).

| Cost | What drives it | Notes |
|---|---|---|
| Application hosting | Server or virtual machines for the Chat App, agents, Tool Gateway, and Background Jobs. | Modest; scales with concurrent users. |
| Sandboxes | One hardened container per active conversation, more with the full-VM option. | Scales with concurrent analytical conversations; containers expire with conversations. |
| Database | One PostgreSQL instance: sessions, knowledge, extracted file text, job state, audit and usage. | Extracted text grows with the number of distinct files read. |
| Model usage | Tokens consumed on the customer's own endpoints across the three slots (main, worker, background). | The largest variable cost. Configuring a lighter model for the background slot, and the strongest model only where needed, controls it. No embedding model is needed. |
| Aras server load | Queries run through the production Aras server. | Bounded by throttling and cost analysis; measured by the load metrics above. |
| Administration | Deployment, upgrades, model endpoint configuration. | No per-installation schema configuration is needed; discovery is automatic. |

## Open questions

1. The baseline manual-effort figures per question type must be established per company during a pilot.
2. Whether the Chat App records citation clicks is a **Proposed** requirement, needed for the citation follow-through metric.
