# 0041. No telemetry to the makers of the product

- Status: Accepted
- Scope: MVP and beyond

## Context

Aras data is the customer's product intellectual property, and the product is self-hosted so that it stays inside the customer's network ([ADR 0002](0002-self-hosted.md)). Usage counts, error reports or update checks would each need an outbound path to the makers, and a promise worded "no telemetry by default" invites the question of what a non-default setting sends.

## Decision

The product sends nothing automatically to its makers: no usage counts, no error reports, no prompt content, and no optional or opt-in telemetry. Data leaves the deployment only to the model endpoints the customer configures. If the makers ever need usage figures, the customer exports them from the usage records and sends them deliberately.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Optional telemetry, off by default, with a documented payload and an explicit switch | Weakens the self-hosted claim and needs an outbound path that a security review must examine and block. |
| Built-in usage or entitlement reporting | Makes the product depend on reaching its makers and contradicts the promise that they receive no data. |

## Consequences

- A customer's security review has one answer: the only outbound connections are to Aras and the configured model endpoints.
- The makers have no automatic view of usage or faults. Product improvement relies on pilots and on what customers choose to send.
- Any usage-based commercial arrangement would rest on a report the customer builds from the usage export, not on figures the product sends.

## Related

[ADR 0002](0002-self-hosted.md), [ADR 0039](0039-audit-and-usage-records.md), [ADR 0040](0040-operator-reports-without-administrator-endpoints.md), [Security and permissions: data exposure to the configured model endpoints](../architecture/security-and-permissions.md#data-exposure-to-the-configured-model-endpoints), [Deployment and operations](../architecture/deployment-and-operations.md), [Model configuration](../specs/model-configuration.md).
