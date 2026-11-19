# 0028. Ask before expensive analysis; disclose assumptions on cheap work

- Status: Accepted
- Scope: MVP

## Context

Business questions use terms the configuration does not define ("delayed", "recurring", "markup process"). Guessing wrong on an expensive analysis wastes minutes and Aras load. Asking on every cheap lookup is tedious.

## Decision

For expensive analysis with a missing definition, the agent asks one quick question first, offering a definition suggested from discovery. For cheap lookups it proceeds and states the assumption. The answer to the question is a learning signal (explicit teaching).

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Always ask | Tedious for simple questions. |
| Never ask; always assume and disclose | Expensive work may be wasted on a wrong guess. |
| Ask open-ended questions | Business users may not know the configuration; a suggested definition built from discovery is easier to accept or correct. |

## Consequences

- The Main Agent needs a notion of "expensive" (Query Analyzer cost estimates and the configured threshold, [deployment and operations](../architecture/deployment-and-operations.md#configuration-surface)).
- Suggested definitions must be grounded in discovered configuration.
- Definitions become learned knowledge, so the question is asked once per organisation, not once per user ([ADR 0020](0020-organisation-wide-learnings-with-disclosed-assumptions.md)).

## Related

[Main Agent](../specs/main-agent.md), [question lifecycle](../architecture/question-lifecycle.md#stage-1-definition-first-because-the-work-is-expensive).
