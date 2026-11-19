# Open questions

This page lists what remains open in the Aras Insight documentation set, and why. Everything else in the documents is decided; the choices behind it are in the [decision records](decisions/README.md).

## Licensing of this repository and of the product

Not decided. The licensing of this repository and of the product is the owner's decision. The repository carries no license file and states no license terms.

## Pricing

Not decided; the decision belongs to the business. [Pricing options](business/pricing-options.md) sets out the models and the trade-offs and gives a recommendation: a flat annual fee per installation for the first customers, because it needs no metering and keeps the promise that the product sends no usage data to its makers; revisit a hybrid model once real usage figures exist. If consumption or hybrid pricing is ever chosen, the billable unit and the way a self-hosted customer reports usage must be settled first.

## Implementation technology

Not decided in these documents: the implementation language, the web framework, the container runtime, and the text-extraction libraries. They are engineering choices that change neither the product nor its security model. The build team records its choice in a new [decision record](decisions/README.md). The technology that is fixed is listed under "Technology choices" in the [architecture overview](architecture/overview.md).
