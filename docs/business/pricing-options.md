# Pricing options

> **Pricing is an open decision.** This document lays out realistic pricing models and their trade-offs to support that decision. It does not recommend one, and nothing here is an offer.

## What shapes pricing for this product

- **Self-hosted.** The customer runs the software on its own infrastructure. The vendor carries no hosting cost per customer.
- **Customer-paid model usage.** Model calls go to the customer's own endpoints, so the customer already pays for the largest variable cost directly. A vendor price that also charges per token would charge twice for the same thing.
- **User-level access.** Each user signs in with their own Aras account, so the number of named users is known and auditable.
- **Value scales with questions asked and with the breadth of roles using it,** not with data volume.
- **Usage is measurable** from the product's own audit and usage records (see [value and ROI](value-and-roi.md)), which makes consumption models feasible to audit on both sides.

## Options

### Flat licence (per installation)

One annual fee per Aras installation, unlimited users and questions.

| For | Against |
|---|---|
| Simplest to buy and budget; no metering. | Hard to size fairly across small and large companies. |
| Encourages broad adoption across roles. | Vendor revenue does not grow with usage or value. |
| No incentive for users to ration questions. | A large up-front commitment before value is proven. |

### Per seat

A fee per named or active user per period.

| For | Against |
|---|---|
| Familiar; scales with company size. | Discourages occasional users, who are a large share of the intended audience (executives, new employees). |
| Named users already exist as Aras accounts. | Named-seat models invite seat rationing; active-user models need metering. |

A variant prices **active users** (users who asked at least one question in the period), which keeps occasional users cheap.

### Consumption

A fee per answered question, per analysis, or per unit of work.

| For | Against |
|---|---|
| Aligns cost with use; low entry cost. | Unpredictable budgets; buyers often resist it. |
| Measurable from the product's records. | Users may hesitate to ask, which works against adoption. |
| | Because the customer already pays model costs, charging per token or per call duplicates that cost; the unit must be defined on product work, not model usage. |

### Hybrid

A platform fee per installation plus a seat or consumption component above an included allowance.

| For | Against |
|---|---|
| Predictable base with upside as usage grows. | More complex to explain and to contract. |
| Base fee covers support and upgrades regardless of use. | Allowance sizing needs usage data the first customers will not yet have. |

### Pilot pricing

Independent of the long-term model: a fixed-fee or no-fee pilot for a limited period, users, and scope, converting to the chosen model on success. See the [pilot plan](pilot-plan.md).

## Implications of self-hosting

- The customer pays its own **model usage** and **infrastructure**; the vendor price covers software, upgrades, product skills and prompts, and support.
- Support for the **full-VM Sandbox option** and for regulated or isolated environments may justify a higher tier.
- Upgrades ship as new product versions, including new shipped skills; pricing must say whether upgrades are included.
- Consumption models require the customer to share usage figures from its own environment, which needs agreement on reporting.

## Questions the decision has to answer

1. Is the target buyer an engineering organisation (budget per installation) or a company-wide function (budget per user)?
2. How important is broad, occasional use across roles, which favours flat or active-user pricing?
3. Should the vendor share in usage growth, which favours consumption or hybrid?
4. What is included: support levels, upgrades, the full-VM Sandbox option, help with pilot setup?
