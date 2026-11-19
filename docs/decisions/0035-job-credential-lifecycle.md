# 0035. Job credentials: renewable grant where the installation offers one, pause and resume, hard lifetime

- Status: Accepted
- Scope: MVP

## Context

A user-confirmed full-text scan ([ADR 0013](0013-user-confirmed-background-full-text-scan.md)) can outlast an interactive access token, and the user may sign out while it runs. It must still act as that user and no one else. Whether a renewable grant can be issued depends on the installation, and a long scan should not lose its progress when a credential lapses.

## Decision

At confirmation the Tool Gateway derives a credential for the requesting user, scoped to that job:

- Where the installation's authorization server issues a renewable grant, the job gets its own refresh grant. It keeps running after sign-out and can be revoked without ending the user's session.
- Where it does not, the job borrows the user's session credential and pauses when that session ends, so scans run only while the user stays signed in. The compatibility check settles which applies.

No other identity is ever used. The credential is renewed on a schedule (default every 15 minutes), has a hard lifetime after which it is never renewed (`gateway.job_credential_max_hours`, default 24 hours), and is dropped when the job ends, fails or is cancelled.

When renewal fails (account disabled, access revoked, grant expired) or the hard lifetime is reached, the job pauses. The user is notified, and the job resumes from its last completed item once the user signs in again. A paused job is cancelled after `jobs.max_pause_hours` (default 72).

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Require renewable grants and switch scans off without them | Unanchored questions would be unsupported on those installations. |
| Always use the session credential | Scans would end at every sign-out. |
| Stop the job and report partial coverage on failure | A long scan would lose its checkpoint. |
| No lifetime cap, or one derived from the estimate | A stolen grant would stay usable indefinitely, or the cap is hard to explain. |

## Consequences

- Behaviour differs by installation and is checked once per installation.
- A scan needing more than 24 hours needs the user to sign in again.

## Related

[Background Jobs](../specs/background-jobs.md), [Tool Gateway](../specs/tool-gateway.md), [Security and permissions](../architecture/security-and-permissions.md#background-jobs-job-scoped-renewable-credentials), [ADR 0034](0034-credentials-encrypted-at-rest.md).
