# 0049. Discovery: when it runs, what it generates, and its limits

- Status: Accepted
- Scope: MVP

## Context

Discovery reads the Aras configuration as the signed-in user and keeps the schema and the discovered knowledge current ([ADR 0016](0016-discovery-as-each-user.md)). Its triggers, reads, model use and first run needed fixing.

## Decision

- By default Discovery reads method names, event bindings and modification stamps only. `discovery.read_method_content` also reads source, hashes it and passes a bounded excerpt to the inference model.
- Change checks run at conversation start and after sign-in, within the minimum interval; there is no periodic check. The Tool Gateway raises an early `inconsistency_check` when Aras or the Query Analyzer reports something the schema cannot explain, at most once in a short window.
- Every run first drains a retry queue of failed or deferred inference.
- Inference is capped at 200 calls per run (a configuration key), also on the first run. The rest is queued, and its entries do not exist until generated.
- Summary entries cover workflow maps, lifecycle maps and lists used by a property.
- Questions may begin once item types, properties, relationship types and lists are stored. Rules needing unloaded types are skipped, and coverage says so.
- Discovery's own queries are fixed templates: no validity analysis, read-only enforcement still applies, verified per Aras version before release. One template set ships per supported version, chosen by version detection; `discovery.template_set` overrides it.
- A change run commits in one transaction; the first run commits per configuration type.
- Discovery queries count against query slots and the Throttle, not the question budget.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Read method source by default | Sensitive logic could reach the model. |
| A periodic check | Needs a scheduler and a credential. |
| A built-in meta-model for validation | Duplicates per-version template verification. |
| Block questions until the first run ends | Users wait for knowledge generation. |
| No inference cap | Unbounded model cost. |

## Consequences

- Early answers meet fewer rules until loading ends.
- Template sets need maintenance per supported Aras version.

## Related

[Discovery](../specs/discovery.md), [Query Analyzer](../specs/query-analyzer.md), [ADR 0004](0004-discovery-only-schema.md), [ADR 0036](0036-one-shared-discovered-configuration.md).
