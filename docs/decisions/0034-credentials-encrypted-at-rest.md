# 0034. Stored Aras credentials are encrypted at rest with a deployment-supplied key

- Status: Accepted
- Scope: MVP

## Context

The Tool Gateway holds an Aras access token, and usually a refresh token, for every signed-in user and background job. Memory only would sign everyone out at each restart, stop a standby taking over and stop jobs outliving a session. In the Database they are a target, so they need protection of their own. The key and other secrets must also reach the service.

## Decision

User and job credentials are stored in encrypted columns using one key supplied by the deployment and referenced by name (`gateway.credential_encryption_key_ref`). The key is never stored in the Database. Tokens are decrypted in memory only for the length of a call and never appear in logs, prompts or the Sandbox. The service refuses to start without the key, with no plain-text fallback.

Secrets (this key, model endpoint credentials, Database credentials) reach the service as environment variables or mounted files, referred to by name. A secret-manager integration can be added on request without a schema change.

At sign-out or session expiry the session credential row is deleted at once and the token is discarded without a call to Aras. A job credential is revoked with Aras where possible and its secret material deleted when the job ends.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Envelope encryption with a key-encryption key in the customer's secret store | More to deploy now; it can be added later without a schema change. |
| Memory only | Restarts sign everyone out, a standby cannot take over, jobs cannot outlive a session. |
| Secret-manager integration in the MVP | Vendor-specific work; environment variables and files suit every platform. |
| Keep the row after sign-out, secret erased | Nothing needs it; the sign-out time is on the session. |

## Consequences

- A database backup holds credentials only encrypted; losing the key means users sign in again.
- Whoever holds the key and the Database can read tokens; a compromised service host exposes the tokens of active users.

## Related

[Tool Gateway](../specs/tool-gateway.md), [Persistence](../specs/persistence.md), [Security and permissions](../architecture/security-and-permissions.md#data-at-rest), [Deployment and operations](../architecture/deployment-and-operations.md#configuration-surface), [ADR 0035](0035-job-credential-lifecycle.md).
