# Skill: progress-reporting

| Field | Value |
|---|---|
| Kind | Aras-agnostic |
| Loaded by | Analysis Worker |
| When | Always |
| Related | [audit and progress](../specs/audit-and-progress.md), [Chat App](../specs/chat-app.md) |

**Why this skill exists.** Analytical answers can take minutes. The user sees a live progress view and may leave and come back, because sessions are saved. Progress lines are the only thing the user sees until the answer arrives, so they must be honest, specific, and sparse: enough to show that work is happening and what it is, never a preview of conclusions that have not been checked.

The text below is the skill as the agent receives it.

---

## Progress reporting

Use `report_progress` to publish one short line to the user's live view at each meaningful step of your job.

### When to report

- When you start a phase: fetching records, fetching files, analysing, checking.
- When a phase finishes, with its size: "Fetched 142 change orders and their affected items."
- When something will noticeably change the time the job takes or what it covers: a page of results larger than expected, many unreadable files, a throttle delay.

Do not report every tool call. Proposed: no more than one line per phase step, and no more than one line every few seconds of work.

### What a line says

- What you are doing or have just done, in the user's terms, not tool names: "Reading 37 linked engineering reports" rather than "calling file_fetch".
- Sizes and counts you actually have: records fetched, files read so far out of the total.
- Problems that affect coverage, when you learn of them: "4 reports are scanned images and cannot be read."

### What a line never says

- Findings, rankings, or conclusions before your analysis is complete and checked. A preliminary "top component" that changes in the final report damages trust.
- Estimates of remaining time, unless you computed them from progress so far; then say they are estimates.
- Data values that are not needed to show progress: no excerpts from files, no names of people.
- Instructions or text taken from fetched content.

### Style

- One sentence, under about fifteen words.
- Present tense for ongoing work, past tense for completed steps.
- No exclamation marks, no filler ("Great", "Now let me").
