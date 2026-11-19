# 0025. OCR deferred; MVP readable formats

- Status: Accepted
- Scope: MVP

## Context

Engineering reports and root-cause analyses are often scanned PDFs or images. Reading them requires OCR, which adds a component, accuracy questions (tables, drawings, handwriting), and processing cost. Many other documents are born-digital: PDFs with a text layer, Office files, plain text.

## Decision

OCR for scanned documents is deferred. The MVP reads PDFs with a text layer, Word, Excel, PowerPoint, and plain text formats (txt, csv, html, xml). Email files, images, and other formats are deferred with OCR. Unreadable files are reported in answer coverage, with citations so the user can open them in Aras.

## Alternatives considered

| Alternative | Why rejected for MVP |
| --- | --- |
| OCR in MVP | Extra component and accuracy risk; the architecture does not depend on it and it can be added behind the File Service later. |
| Silently skipping unreadable files | Violates coverage reporting ([ADR 0027](0027-causal-honesty-and-coverage-reporting.md)). |

## Consequences

- Answers over scanned evidence will show gaps, honestly reported. Coverage metrics show how much OCR would add, which informs when to build it.
- Each unreadable file that has a name is registered as a file-level citation, so the user can open it in Aras; a file the user is denied access to is counted but never named.
- The File Service interface is format-agnostic so new extractors can be added.

## Related

[File Service](../specs/file-service.md), [roadmap](../product/roadmap.md).
