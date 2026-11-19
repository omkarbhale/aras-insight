# 0010. Main Agent plus Analysis Workers

- Status: Accepted
- Scope: MVP

## Context

Questions vary from "show my open change requests" to "compare the design review process with its defined workflow across six months of runs". A single agent doing everything would either be slow and expensive for simple questions (spinning up a Sandbox each time) or would drag bulk data into its conversation context for complex ones.

## Decision

A Main Agent holds the conversation. It answers simple lookups directly through the Tool Gateway and starts sandboxed Analysis Workers only for heavy work: aggregation, trends, process questions, many files. Several workers can run in parallel for multi-part questions. Workers receive a bounded job and report back with findings, computed numbers, citations, assumptions, and coverage.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| One agent with a Sandbox always available | Simple questions pay Sandbox and planning overhead; the conversation context fills with tool output from heavy work. |
| One agent, no Sandbox | Cannot compute reliably over many rows ([ADR 0008](0008-sandboxed-python-analysis.md)). |
| A deep hierarchy of agents (planner, retriever, analyst, writer) | More hand-offs, more cost, more places to lose citations, without clear benefit at MVP scale. |

## Consequences

- Simple questions are fast and cheap.
- The Main Agent's context stays focused on the conversation; bulk data stays in the Sandbox.
- The Main Agent must write good bounded job descriptions; the job description is the contract.
- Worker reports have a fixed shape so the Main Agent can compose them without recomputation.
- Parallel workers share the conversation's Sandbox and draw on the turn's question budget; each worker has a share of it.

## Related

[Main Agent](../specs/main-agent.md), [Analysis Worker and Sandbox](../specs/analysis-worker-and-sandbox.md), [question lifecycle](../architecture/question-lifecycle.md).
