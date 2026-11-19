# 0019. Invalidation by provenance and re-derivation, plus contradiction

- Status: Accepted
- Scope: MVP

## Context

Learned knowledge goes stale when the configuration it depends on changes ([ADR 0018](0018-learned-knowledge-depends-only-on-configuration.md)). It can also simply be wrong. An early idea was a deployment hook: when Aras configuration is deployed, trigger re-discovery and update dependent knowledge. A hook requires integrating with each customer's deployment process.

## Decision

There is no deployment hook in the MVP. Invalidation works from provenance and from contradiction:

- **Discovered entries inferred by the model** (not Learned entries) record the exact configuration items, and their modification stamps, they were derived from. When a source changes they are re-derived automatically; when the last remaining source is deleted they are retired, and while other sources remain the entry is marked uncertain and re-derived from them.
- **Conversation-formed learnings** record the configuration elements they map to (for example the lifecycle state a definition refers to). A renamed or removed element makes them uncertain.
- **Contradiction:** a user correction on an answer that cited a learned entry makes it uncertain. A query built from an entry failing static analysis also makes it uncertain. Disagreement between users makes it contested, and the agent asks which meaning the user intends.
- **States:** confirmed, uncertain, contested, retired. Uncertain entries stay usable but are verified first (a Discovered entry is re-derived; a Learned entry is checked against the configuration or confirmed with the user).

Learning may be automatic; there is no mandatory human approval gate.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Deployment hook in MVP | Requires integration with each customer's configuration deployment process; modification stamps detect the same changes at the next check. Deferred (see [roadmap](../product/roadmap.md)). |
| Administrator-triggered discovery check | Discovery runs only with a signed-in user's credential, and an operator is not necessarily an Aras user. The check at the next sign-in or conversation start already covers an Aras change. |
| Time-based expiry of learned knowledge | Expires correct knowledge and keeps wrong knowledge until the timer. Ageing out unused knowledge is a separate, deferred idea. |
| Mandatory human approval of every learned entry | Creates a review queue nobody owns; disclosure as an assumption achieves review in the flow of work ([ADR 0020](0020-organisation-wide-learnings-with-disclosed-assumptions.md)). |

## Consequences

- Every learned entry must carry provenance and configuration mappings; the entry shape is defined in [knowledge-and-learning.md](../specs/knowledge-and-learning.md).
- Staleness is detected at the next modification-stamp check, not instantly.
- Wrong knowledge is surfaced by use: corrections and failed analysis demote it.
- There is no manual discovery trigger: the check at sign-in and conversation start, limited only by the minimum interval between checks, already covers an Aras upgrade or configuration deployment, with no administrator identity or borrowed credential (see [deployment and operations](../architecture/deployment-and-operations.md#upgrading-aras-or-deploying-aras-configuration-changes)).

## Related

[Discovery](../specs/discovery.md), [ADR 0016](0016-discovery-as-each-user.md), [ADR 0049](0049-discovery-operation.md), [ADR 0040](0040-operator-reports-without-administrator-endpoints.md).
