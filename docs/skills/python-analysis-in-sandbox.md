# Skill: python-analysis-in-sandbox

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Analysis Worker |
| When | Always |
| Related | [Analysis Worker and Sandbox](../specs/analysis-worker-and-sandbox.md), [citing-sources](citing-sources.md), [reporting-coverage-and-assumptions](reporting-coverage-and-assumptions.md) |

**Why this skill exists.** The product has no bespoke aggregation engine. Analysis is done by a worker that writes and runs Python over files in its Sandbox: AML results and extracted file text that the Tool Gateway wrote there. The benefit is generality; the risk is sloppy analysis that looks precise. This skill sets the working method: inspect before computing, compute every number in code, keep citation ids attached to the data, check that results reconcile, and keep output small.

The text below is the skill as the agent receives it.

---

## Python analysis in the Sandbox

You analyse data by writing Python and running it with `python_run`. Data reaches the Sandbox only through `aml_fetch` (records) and `file_fetch` (extracted file text); each call tells you the path of the file it wrote. The Sandbox has no network. Nothing you write in it leaves it except through `report_progress` and `submit_report`.

### Every number comes from code

- Every count, sum, average, median, percentage, duration, and ranking in your report is printed by code you ran. Do not do arithmetic in your head, and do not adjust a number by hand afterwards.
- Keep the code that produced each reported number. Your report says, in a sentence, how each number was computed.

### Inspect before you compute

Before analysing a fetched file:

1. Check its size and the number of records.
2. Look at the field names and a few records. Do not assume field names; the brief tells you what to expect, the file tells you what is there.
3. Check types and missing values for every field you will use: empty strings, nulls, dates in unexpected formats, numbers stored as text.
4. Check identity: whether a record can appear more than once (for example several versions of the same item, or the same item reached through two relationships). Deduplicate by the identity the brief specifies, and say how.

If the data does not match what the brief expected (fields missing, far more or fewer records), stop and decide whether the question can still be answered. Report the mismatch; do not quietly change the question.

### Keep citations attached

- Every fetched record carries its citation id; every fetched passage carries its citation id. Keep that field through every filter, join, and grouping.
- When you produce a ranking or a list, keep, for each entry, the citation ids of the records it rests on, so the report can cite representative evidence.
- When you find a passage in file text, keep its citation id, not only the file's.

### Dates and durations

- Parse dates explicitly with their format and time zone. Do not compare dates as strings.
- State which date properties define a duration, and treat records missing either date as missing evidence, not as zero.
- Report durations in units the user will understand (days, not seconds) and say whether they are calendar or working days.

### Distributions, not just averages

- For durations and counts per group, report the median and the spread alongside any average. One extreme record can move an average a long way.
- Report group sizes with every rate or share.
- When comparing groups (for example delayed versus not delayed), compute the same measure for both groups.

### Searching text

- Search with code: normalise case and whitespace, use patterns that tolerate common variants, and search the fetched text rather than reading it all into your context.
- Read only the matched passages and their surroundings with `sandbox_read`, not whole files.
- When you group explanations found in text (for example failure modes), define each group by the words that place a passage in it, assign passages with code, and read a sample from each group to check that the grouping is sound. Report how many passages fell into no group.

### Keep output small

- Print summaries, not raw data. Print at most a few dozen lines per run.
- Write intermediate results to files in the Sandbox and print their paths and sizes.
- Use `sandbox_read` to read a specific part of a file when you need to see it; read ranges, not whole large files.

### Check before you report

- Totals reconcile: groups add up to the population, and the population matches what was fetched minus what you excluded, with the exclusions counted.
- Spot-check: for the top entries of any ranking, read the underlying records and confirm they say what the numbers imply.
- Coverage counts (records with missing evidence, unreadable files, records excluded) are computed, not estimated.

### Errors

If code fails, read the error, fix the cause, and run it again. Do not paper over errors by dropping records silently; if records must be excluded, count them and report the reason.

### Libraries

Use the libraries listed in the Sandbox environment manifest in your brief. If a library you want is not listed, use the standard library. Do not try to install anything.
