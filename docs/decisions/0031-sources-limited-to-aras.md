# 0031. Sources limited to Aras and what it connects to

- Status: Accepted
- Scope: MVP

## Context

Product questions often span systems: ERP for cost, MES for manufacturing data, supplier portals. Connecting them multiplies integration, permission, and identity work.

## Decision

Sources are only Aras and the things it connects to (for example its vault server). Other enterprise sources are deferred.

## Alternatives considered

| Alternative | Why rejected for MVP |
| --- | --- |
| Connectors to ERP, MES, or supplier systems | Each has its own permission model and identity; user-level access would need to be re-established per source. Deferred (see [roadmap](../product/roadmap.md)). |
| Multiple Aras instances in one deployment | Deferred; one deployment serves one Aras installation. |

## Consequences

- One identity system (Aras sign-in) and one permission model.
- Questions that need other systems are answered with what Aras holds, and coverage says so.

## Related

[Scope](../product/scope.md).
