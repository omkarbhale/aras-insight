# Pricing options

> **Pricing is not decided.** The pricing decision belongs to the business and has not been made. This document lays out realistic pricing models and their trade-offs, and gives a recommendation for the first customers to support that decision. Nothing here is an offer.

## What shapes pricing for this product

- **Self-hosted.** The customer runs the software on its own infrastructure. The vendor carries no hosting cost per customer.
- **Customer-paid model usage.** Model calls go to the customer's own endpoints, so the customer already pays for the largest variable cost directly. A vendor price that also charges per token would charge twice for the same thing.
- **User-level access.** Each user signs in with their own Aras account, so the number of named users is known and auditable.
- **Value scales with questions asked and with the breadth of roles using it,** not with data volume.
- **Usage is measurable** from the product's own audit and usage records (see [value and ROI](value-and-roi.md)), which makes consumption models feasible to audit on both sides.
- **No telemetry to the makers.** The product sends nothing to its makers ([decision record 0041](../decisions/0041-no-telemetry-to-the-makers.md)). Any usage figures a price needs would have to be exported by the customer and sent deliberately.

## Recommendation

For the first customers, the recommendation is **a flat annual fee per installation**, with unlimited users and questions. It needs no metering, so it keeps the no-telemetry promise without a reporting arrangement. It is the simplest model to buy and to budget, and it encourages broad use across roles. Once real usage figures exist, a hybrid model should be revisited.

The recommendation is not a decision. The final call belongs to the business, and until it is made no model, fee, or amount is fixed.

**What the fee includes.** Product updates, including the shipped skills and prompts, and support. It excludes the customer's own infrastructure and the model usage on the customer's own endpoints.

**Pilot pricing.** The pilot is a fixed-fee, time-boxed engagement, independent of the long-term model. If the company adopts the product at the end of the pilot, the pilot fee is credited against the first year's fee. See the [pilot plan](pilot-plan.md).

**Revisit condition.** If consumption or hybrid pricing is ever chosen, the billable unit has to be defined on product work, not on model usage, and the customer has to report its usage without the product sending anything outward. The natural unit is the answered question, which is computable from the usage report that operators already read from the product's database views, and the customer sends that report itself.

## Options

The options below are examples of the models that could be chosen, not a closed list.

### Flat annual fee (per installation)

One annual fee per Aras installation, unlimited users and questions. This is the recommended model for the first customers.

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
| | Needs a usage report that the customer sends deliberately, because the product reports nothing outward. |

### Hybrid

A platform fee per installation plus a seat or consumption component above an included allowance.

| For | Against |
|---|---|
| Predictable base with upside as usage grows. | More complex to explain and to contract. |
| Base fee covers support and upgrades regardless of use. | Allowance sizing needs usage data the first customers will not yet have. |

## Implications of self-hosting

- The customer pays its own **model usage** and **infrastructure**. A vendor fee covers product updates and support.
- Updates ship as new product versions, including new shipped skills and prompts, and are included in the fee.
- A usage-based component would rely on a usage report that the customer exports from its own environment and sends deliberately, which needs agreement on reporting.

## Questions the decision has to answer

1. Is the target buyer an engineering organisation (budget per installation) or a company-wide function (budget per user)?
2. How important is broad, occasional use across roles, which favours flat or active-user pricing?
3. Should the vendor share in usage growth, which favours consumption or hybrid?
