# 0032. Saved sessions and live progress

- Status: Accepted
- Scope: MVP

## Context

Heavy answers can take minutes; background scans longer. Users should not have to watch a spinner or lose work if they close the browser or a service instance restarts.

## Decision

Sessions are saved in the Database during inference. Long-running answers show live progress (workers publish progress lines through `report_progress`), and the user can leave and come back. Learning runs live, or over a session's transcript after an inactivity threshold.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Sessions in memory only | Lost on restart or disconnect. |
| No progress, only the final answer | Users cannot tell whether the system is working or what it is doing. |

## Consequences

- The agent loop persists each step ([ADR 0011](0011-own-thin-agent-loop.md)).
- Transcripts are stored, and are private to their user ([security and permissions](../architecture/security-and-permissions.md#data-at-rest)).
- Progress lines must describe what is being done, not reveal internals that confuse users.
- With one active service instance in the MVP, a restart resumes the interrupted turn from its saved transcript; no ownership or takeover data is needed ([ADR 0037](0037-single-instance-mvp-and-optional-gateway-process.md)).
- Live and batched learning are both core capabilities ([ADR 0053](0053-core-capabilities.md)).

## Related

[Persistence](../specs/persistence.md), [Audit and progress](../specs/audit-and-progress.md), [ADR 0037](0037-single-instance-mvp-and-optional-gateway-process.md).
