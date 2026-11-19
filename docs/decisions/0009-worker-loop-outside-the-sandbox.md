# 0009. Sealed Sandbox, with the worker loop outside it

- Status: Accepted
- Scope: MVP

## Context

The Analysis Worker needs a model (to decide what to fetch and what code to write), Aras data (fetched as the user), and a place to run code. If the whole worker ran inside the Sandbox, the Sandbox would need network access to the model endpoints and to the Tool Gateway, plus model credentials. Model-written code running there could then use that network and those credentials. Code that processes adversarial content (any record or file) must be assumed hostile.

## Decision

The Sandbox has no network and never calls Aras itself. The Analysis Worker's loop runs in our service; only its code execution and file reads happen inside the Sandbox. Data arrives only as files written by the Tool Gateway, which holds the user's Aras credentials, runs static analysis and throttling, and executes AML as the user. One Sandbox per user conversation, expiring with it. A hardened container is the default; a full virtual machine is an option for customers who require it. Fetched file text lands in the Sandbox and is searched with code.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Whole worker agent inside the Sandbox, with network to model and gateway | The Sandbox would hold model credentials and network routes that model-written code could abuse. |
| Sandbox with a tool connection the code itself can call | Lets model-written code issue queries directly, outside the worker's reasoning and harder to audit. The decided design keeps every fetch a deliberate tool call by the worker loop. |
| One shared Sandbox per user, or a pool across users | Cross-conversation or cross-user residue; harder to reason about isolation. |
| VM per conversation as the default | Stronger isolation, but slower and heavier. Offered as an option. |

## Consequences

- A compromised Sandbox can only affect the files of one conversation, which that user could already read.
- The worker's tools split cleanly: `aml_fetch` and `file_fetch` go through the Tool Gateway, `python_run` and `sandbox_read` act in the Sandbox.
- The service must manage Sandbox lifecycle (create on first use, destroy on expiry) and capacity.
- Code inside the Sandbox cannot fetch more data mid-run; the worker fetches, then computes. This shapes how workers plan.

## Related

[Analysis Worker and Sandbox](../specs/analysis-worker-and-sandbox.md), [security and permissions](../architecture/security-and-permissions.md#sandbox-isolation).
