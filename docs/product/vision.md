# Product vision

## The problem in depth

### Why answers are slow to get today

A mature Aras installation records, item by item, how a product came to be: parts and their structures, documents and their revisions, change requests and change orders, problem reports, the workflow activities each of them passed through, who signed off and with what comment, and the files in the vault behind all of it. This is some of the most valuable operational data a manufacturing company has. Four obstacles stand between that data and the people who need answers from it.

1. **The data model is a barrier.** To ask Aras anything beyond a saved search, a user has to know which item types exist, how they relate, which property holds which meaning, and which lifecycle state names are in use. In most installations these have been customised over years. The knowledge sits with a few administrators and power users.
2. **Analysis happens outside the system.** Standard searches and reports return lists. Questions about counts over time, durations, rankings, or comparisons are answered by exporting data and working in spreadsheets. Each such question is a small project, done by someone whose time is scarce, and the result is out of date by the time it is shared.
3. **The reasons are in text, not rows.** Why a change order was late, why a part keeps being revised, or why a test failed is written in description fields, workflow sign-off comments, and attached reports. Counting rows cannot answer a "why" question, and reading dozens of attached documents by hand rarely happens.
4. **Generic AI tools cannot be trusted here.** A general assistant does not know this installation's data model, cannot see the data under the user's permissions, cannot compute over thousands of records reliably, and cannot show where its answer came from. Its answers cannot be checked, so they cannot be used for decisions.

### Who suffers

- **Executives and program leaders** decide on schedule, cost, and quality with summaries that are days old or missing.
- **Engineering and quality managers** spend analyst time on recurring questions, or do not ask them.
- **Engineers** search by hand through change history and reports to find out why something is the way it is.
- **Change and configuration managers** have no quick way to see where the process is deviating or waiting.
- **Administrators and power users** become a bottleneck, answering other people's data questions.
- **New employees** take months to learn the installation's structure and vocabulary.

## The vision

Anyone who can sign in to Aras can ask it a question in their own words and get a correct, cited answer at their own access level, in minutes instead of days, without knowing the data model, and without any data leaving the company's control.

Behind that sentence are three commitments:

- **The product does the work an analyst would do.** It finds the records, reads the files, computes the numbers with code, and writes up the result. It does not stop at returning a list.
- **The product earns trust by being checkable.** Every claim is traceable to a record, a file passage, or a stated piece of knowledge, and every answer says what it did not cover.
- **The product gets better at this installation over time.** It discovers the configuration on its own and learns the meaning of local terms and processes from use, so the second person to ask a question benefits from the first.

## Product principles

These principles are product rules. They apply to every feature, prompt, and module.

| Principle | What it means | Why |
|---|---|---|
| Read-only | Aras Insight never creates, edits, promotes, or deletes anything in Aras. | Removes the largest category of risk and approval overhead. Writing is deferred, if it happens at all. |
| User-level access only | Every data access happens as the signed-in user, with that user's Aras permissions. There is no privileged service account, no admin-granted organisation-wide view, and no hybrid model. | Aras permissions are the company's existing, audited access policy. Re-implementing them elsewhere would eventually diverge from them. |
| Everything through the Aras server | In the MVP, data is read only through the Aras server, never directly from its database. | Aras applies permissions and server-side logic at read time, and only the server applies them faithfully. |
| How, never what | Learned knowledge records how the installation works (what a term means, which property holds what, how a process runs). It never stores business data values. | Shared knowledge must be safe to show to every user, and must go stale only when the configuration changes. In the MVP the rule is enforced by prompt. |
| Numbers are computed, never guessed | Every count, sum, ratio, duration, or ranking is computed by code over fetched data. | Language models are unreliable at arithmetic over many rows. Computed numbers can be reproduced. |
| Cite everything | Each claim cites a record (type, id, link), a file (with page or section), or a knowledge entry (with its state and confidence). | Users need to verify before they act. Citations also expose wrong knowledge so it can be corrected. |
| Report coverage | Each answer states how many records it used out of those the user can access, which had no linked evidence, and what could not be read. | An answer is only as good as what it saw. Hiding gaps produces false confidence. |
| Association, not causation | The product reports what appears together ("appears most often on delayed change orders") and does not claim causes the data cannot show. | Overstated causal claims lead to wrong decisions and destroy trust. |

Two further rules follow from the principles. Where a question depends on a definition the installation does not make explicit (for example what counts as "delayed"), the product asks one short question before expensive analysis, offering a definition suggested from discovery, and for cheap lookups it proceeds and states the assumption. Where knowledge taught by one person has not yet been confirmed by another, the product states it as an assumption when it uses it.

## What success looks like

- **For a business user:** a question that used to need an analyst, an export, or an afternoon of reading is answered in one conversation, with links they can click to check.
- **For a manager:** recurring analytical questions (where approvals wait, what keeps recurring, how a program's changes are trending) are asked directly and repeatedly, not commissioned.
- **For IT and security:** the product runs inside the company's environment, on the company's models, adds no new access path around Aras permissions, and produces a complete audit record of what it queried on whose behalf.
- **For the Aras owners:** production load from the product stays within agreed limits, and nobody needs to configure the product by hand when the Aras configuration changes.
- **For the product itself:** citations are accurate when checked, coverage is reported on every answer, and the share of questions answered without a correction rises over time as the product learns the installation.

How these outcomes are measured is defined in [value and ROI](../business/value-and-roi.md). How they are tested before adoption is in the [pilot plan](../business/pilot-plan.md). The design that delivers them starts at the [architecture overview](../architecture/overview.md).
