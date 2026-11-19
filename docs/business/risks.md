# Risks

This register lists the main risks of building, deploying, and adopting Aras Insight, with the mitigations the design already contains; where the MVP has none, the row says so. Likelihood and impact ratings are **Proposed** and should be revisited after a pilot.

Security design detail is in [security and permissions](../architecture/security-and-permissions.md); operational detail is in [deployment and operations](../architecture/deployment-and-operations.md).

## Technical

| Risk | Likelihood | Impact | Mitigation from the design |
|---|---|---|---|
| Generated AML is invalid or silently ignores fields, giving empty or wrong results. | High without mitigation | High | The Query Analyzer checks every query against the discovered schema (unknown item types, properties, relationships, list and state values, value types, ignored fields) and returns diagnostics to the model for a retry. A shipped AML skill guides query writing. |
| Installations differ widely; hand configuration would not scale. | Certain | High | Schema is discovered only, never configured by hand. Meaning is learned from conversations and stored as instance knowledge. |
| Discovered or learned knowledge goes stale after configuration changes. | Medium | Medium | Discovery detects changes through modification stamps. Learned entries record the configuration elements they depend on and become uncertain or retired when those change. Uncertain entries are verified before use. |
| Long analyses fail or time out. | Medium | Medium | Bounded jobs per Analysis Worker, row budgets and timeouts, sessions saved during inference so the user can leave and return, and live progress. Full-text scans run as confirmed background jobs. |
| Large files or many files exceed what an answer can read. | Medium | Medium | Narrowing through the Aras graph before reading, search within files by code in the Sandbox, extracted text stored per file id for reuse, and coverage stating what was not read. |

## Security and privacy

| Risk | Likelihood | Impact | Mitigation from the design |
|---|---|---|---|
| A user sees data they are not permitted to see. | Low | Critical | Every data access goes through the Aras server as the signed-in user; no service account, no admin-granted aggregate access. Stored file text is re-checked through Aras on every use. |
| Code run in the Sandbox exfiltrates data or reaches other systems. | Low | High | The Sandbox has no network, never calls Aras itself, never holds model or Aras credentials, receives data only as files from the Tool Gateway, and expires with the conversation. A full VM is available where a container is not considered enough isolation. |
| Business data leaks into shared learned knowledge, which every user sees. | Medium | High | "How, never what": learned knowledge describes configuration and meaning only, enforced in the MVP by a strong initial prompt. Scheduled automated leak checks are deferred ([roadmap](../product/roadmap.md)). |
| Data leaves the company's environment. | Low | High | Self-hosted; models run on endpoints the customer chooses. Data goes to an external provider only if the customer configures one. |
| Long-running background jobs need credentials beyond the user's session. | Medium | Medium | A renewable credential for the requesting user, scoped to the job and dropped when it ends; no privileged account. This applies where the installation's sign-in issues renewable tokens; otherwise scans run only while the user's session is live. |
| Stored conversations contain sensitive content. | Medium | Medium | Conversations are treated as private and stored only in the customer's own database. |

## Model quality

| Risk | Likelihood | Impact | Mitigation from the design |
|---|---|---|---|
| The model invents facts or citations. | Medium | High | Every claim cites a record, file passage, or knowledge entry recorded in the Citation Registry, so citations point at things that were actually retrieved. Users can click through to verify. |
| The model miscalculates. | High without mitigation | High | Numbers are computed by Python in the Sandbox, never produced by the model directly. |
| The model overstates causes. | Medium | High | Association, not causation, is a standing product rule in the prompts. |
| Ambiguous terms are interpreted wrongly. | High | Medium | One clarifying question before expensive analysis with a suggested definition; stated assumptions for cheap lookups; learned definitions stated as assumptions until independently confirmed. |
| The customer's approved models are weaker than assumed. | Medium | Medium | The design assumes a reasonably competent model. Three configurable slots let the customer place its strongest model where it matters; a pilot measures quality on the actual models. |
| Wrong knowledge spreads to everyone. | Medium | Medium | Provenance on every learned entry; a correction on an answer that cited an entry makes it uncertain; disagreement makes it contested and the agent asks which meaning is intended. |

## Production load

| Risk | Likelihood | Impact | Mitigation from the design |
|---|---|---|---|
| Analytical queries slow the production Aras server for other users. | Medium | High | Static cost analysis of every query (select-all, missing row limits or paging, deep nesting, broad wildcards); throttling per user and system-wide; row budgets per question; timeouts. |
| Full-text scans across many files load the vault. | Medium | Medium | Only as a background job the user confirms, batched and throttled; extracted text is stored per file id, so later scans are cheaper. |
| Repeated identical questions re-run against Aras. | Medium | Low | Accepted in the MVP; caching is deferred. Usage records show whether it becomes a problem. |
| Load remains too high even with throttling. | Low | High | None beyond the controls above in the MVP. An analytics path that avoids production is a roadmap item, subject to preserving user-level access. |

## Aras upgrades and customisation

| Risk | Likelihood | Impact | Mitigation from the design |
|---|---|---|---|
| An Aras upgrade changes item types, properties, or behaviour. | Medium | Medium | Re-discovery picks up configuration changes; dependent discovered knowledge is re-derived or retired, and dependent learned knowledge is marked uncertain. Business item type, property, and state names are never hard-coded. |
| Server events or custom methods change what reads return (for example logic run before a read). | Medium | Medium | Reading through the Aras server means such logic is applied as it is for any other client; the product does not bypass it. |
| A deployment changes configuration without the product noticing quickly. | Low | Low | Stamp-based change detection; a deployment hook is deferred. |

## Adoption

| Risk | Likelihood | Impact | Mitigation from the design |
|---|---|---|---|
| Users do not trust the answers. | Medium | High | Citations, coverage, stated assumptions, and computed numbers make answers checkable. |
| Early answers are weak before the product has learned the installation. | High | Medium | Discovery works from day one; shipped skills cover generic Aras knowledge; learning from corrections is automatic. A pilot should include a short learning period. |
| Users do not know what to ask. | Medium | Medium | None in the app in the MVP. Example questions per role are documented ([personas and use cases](../product/personas-and-use-cases.md)); suggested questions are a roadmap item. |
| Value is hard to demonstrate to leadership. | Medium | Medium | ROI metrics defined from the product's own records ([value and ROI](value-and-roi.md)) and produced by query or report in the MVP; a dashboard is a roadmap item. |
| IT or security rejects the deployment. | Medium | High | Read-only, self-hosted, user-level access, no network in the Sandbox, configurable endpoints, full audit records. |

## Open questions

1. Likelihood and impact ratings are Proposed and need confirmation after a pilot.
