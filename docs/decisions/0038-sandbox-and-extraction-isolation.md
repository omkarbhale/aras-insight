# 0038. Sandbox hardening, isolated text extraction, and the file channel into the Sandbox

- Status: Accepted
- Scope: MVP

## Context

Model-written code runs in the Sandbox ([ADR 0009](0009-worker-loop-outside-the-sandbox.md)), and document extractors parse files written by Aras users, so both are assumed hostile. The Sandbox has no network, so something must place files into it, and raw vault bytes would be a second copy of business files.

## Decision

The default Sandbox container uses the runtime's standard hardening profile plus a system-call filter, a non-root user, a read-only base image and no added capabilities, with limits on CPU, memory, disk, process count and run time. A full virtual machine remains an option. The container runtime is not decided here; the build team records its choice in a decision record.

Text extraction runs in a separate hardened container with the same runtime and profile: no network, no credentials, a read-only file system apart from a private temporary directory, memory and time limits, and only the one file it extracts.

For the file channel, the rule is that the Sandbox has no network path in and the host-side write is the only inbound path. The reference design is a per-Sandbox data directory written only by the host-side Tool Gateway and read-only to code inside, plus a writable work directory.

Original file bytes are held only while text is extracted, then deleted. A later re-extraction downloads the file from Aras again, within the question's byte budget.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| A stricter profile (user-space kernel or lightweight VM) as the default | Heavier; available through the VM option. |
| Virtual machine only | Slower and heavier for every conversation. |
| Extraction in a plain process, or inside the Sandbox | Weaker isolation, or untrusted documents beside a worker's data. |
| Caching downloaded bytes | A second copy of business files with its own retention problems. |

## Consequences

- One isolation mechanism to build and review.
- Re-extraction after an extractor upgrade adds load on Aras.
- A container escape remains a residual risk, reduced by the VM option.

## Related

[Analysis Worker and Sandbox](../specs/analysis-worker-and-sandbox.md), [File Service](../specs/file-service.md), [Security and permissions](../architecture/security-and-permissions.md#sandbox-isolation).
