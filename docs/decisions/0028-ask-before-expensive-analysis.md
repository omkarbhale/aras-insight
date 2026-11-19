# 0028. Ask before expensive analysis; disclose assumptions on cheap work

- Status: Accepted
- Scope: MVP

## Context

Business questions use terms the configuration does not define ("delayed", "recurring", "design review process"). Guessing wrong on an expensive analysis wastes minutes and Aras load. Asking on every cheap lookup is tedious.

## Decision

For expensive analysis with a missing definition, the agent asks one quick question first, offering a definition suggested from discovery. For cheap lookups it proceeds and states the assumption. The answer to the question is a learning signal (explicit teaching).

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Always ask | Tedious for simple questions. |
| Never ask; always assume and disclose | Expensive work may be wasted on a wrong guess. |
| Ask open-ended questions | Business users may not know the configuration; a suggested definition built from discovery is easier to accept or correct. |

## Consequences

- The Main Agent needs a notion of "expensive", decided by the route: any analysis, background job, many-file read, or high-cost query as reported by the Query Analyzer is expensive, and any direct lookup is cheap ([Main Agent](../specs/main-agent.md)).
- Suggested definitions must be grounded in discovered configuration.
- Definitions become learned knowledge, so the question is asked once per organisation, not once per user ([ADR 0020](0020-organisation-wide-learnings-with-disclosed-assumptions.md)).
- Because live learning is a core capability, the answer applies for the rest of the conversation and for others at once ([ADR 0053](0053-core-capabilities.md)).

## Related

[Main Agent](../specs/main-agent.md), [question lifecycle](../architecture/question-lifecycle.md#stage-1-definition-first-because-the-work-is-expensive).
