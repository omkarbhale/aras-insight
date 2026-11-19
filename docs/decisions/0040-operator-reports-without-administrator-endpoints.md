# 0040. Operator reports through documented views and export; no administrator endpoints in the MVP

- Status: Accepted
- Scope: MVP

## Context

A security review, a pilot and an Aras upgrade need operational tooling: reading audit and usage data, checking that an Aras version behaves as the design assumes, and protecting Aras under load. The product has no administrator identity, and features for Aras administrators and developers are outside the MVP ([ADR 0003](0003-business-end-users-first.md)). The tooling must not add a privileged identity or borrow a user's credential ([ADR 0005](0005-user-level-access-through-aras-server.md)).

## Decision

Audit and usage data are read through documented database views that produce the metrics defined in the business case, plus an export operation. There are no administrator-only service endpoints and no administrator sign-in.

The product ships a one-time compatibility check, run by an administrator against a target installation with an ordinary user account. It exercises sign-in, a paged AML read, a configuration read, a file download and a workflow-history read, and reports what works. The remaining checks (read-time server events, file immutability, change-record names, lock behaviour) are documented manual checks that the customer's Aras owners confirm.

There is no manual discovery trigger: the check at sign-in and conversation start covers an Aras change. Only the requesting user can see, confirm, cancel or reauthorise a background job; operators lower limits or pause the job runner to protect Aras.

The reports and the compatibility check are operations tooling for the customer's infrastructure staff, not features for Aras administrators.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Administrator-only endpoints plus database access | Needs a new identity and sign-in; views and export suffice. |
| Database access and export, without views | Every customer would write the metric queries. |
| Manual discovery trigger with another user's credential | Breaks "every call is the user". |
| Administrators viewing or cancelling others' jobs | Jobs run as their user; no administrator identity exists. |
| A shipped check for every compatibility item | A tool can exercise only five of them. |

## Consequences

- Pilot metrics need no manual data collection, but reading them takes database access.
- A graphical usage dashboard is deferred ([roadmap](../product/roadmap.md)).

## Related

[Audit and progress](../specs/audit-and-progress.md), [Deployment and operations](../architecture/deployment-and-operations.md#aras-version-compatibility-to-verify), [Scope](../product/scope.md), [Background Jobs](../specs/background-jobs.md).
