# 0037. One service instance in the MVP; the Tool Gateway as an optional separate process

- Status: Accepted
- Scope: MVP

## Context

Several service instances need a rule for who owns a running turn or job and how another instance takes over after a failure. The Tool Gateway holds every signed-in user's credentials, so a compromised agent process should not reach them. Background jobs need scheduling and claiming, and the Throttle must count running queries. Each answer adds deployment work for the customer.

## Decision

The MVP runs one active service instance; others may exist only as a cold standby. No owner, heartbeat or lease data is kept on turns, analysis jobs or background jobs. After a restart, an interrupted turn resumes from its saved transcript.

Background jobs are scheduled, claimed, retried and tracked in the PostgreSQL database alone, with no message broker or queue product.

Throttle leases live in the database when more than one instance runs and may stay in memory for a single instance. Per-question budgets are always stored in the database, so a resumed turn keeps its accounting.

The Tool Gateway runs inside the service. Running it as its own process with its own database role is an optional hardening setup, not an MVP requirement: the credential columns are already encrypted and read only through the gateway interface.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Several active instances with ownership leases and takeover | Deferred until a customer needs high availability. |
| Restart an interrupted question with a notice | Discards work the saved transcript keeps ([ADR 0032](0032-saved-sessions-and-live-progress.md)). |
| A message broker or queue product | Another component for the customer to install and secure ([ADR 0015](0015-one-postgresql.md)). |
| Separate gateway process required in the MVP | More deployment work for a pilot-sized installation. |

## Consequences

- The single-host topology is the pilot default; availability beyond restart and cold standby is not provided.
- Periodic tasks (inactivity sweep, retention purge) run in the one active instance without coordination.
- Several active instances would need a new record.

## Related

[Deployment and operations](../architecture/deployment-and-operations.md#what-is-deployed), [Tool Gateway](../specs/tool-gateway.md), [Persistence](../specs/persistence.md), [ADR 0015](0015-one-postgresql.md).
