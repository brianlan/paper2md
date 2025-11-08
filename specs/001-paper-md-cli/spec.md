# Feature Specification: Paper-to-Markdown CLI

**Feature Branch**: `001-paper-md-cli`  
**Created**: 2025-11-07  
**Status**: Draft  
**Input**: User description: "Create a cli tool that takes as input a local pdf file (usually an academic paper) and convert it into a corresponding markdown file..."

## User Scenarios & Testing *(mandatory)*

<!--
  IMPORTANT: User stories should be PRIORITIZED as user journeys ordered by importance.
  Each user story/journey must be INDEPENDENTLY TESTABLE - meaning if you implement just ONE of them,
  you should still have a viable MVP (Minimum Viable Product) that delivers value.
  
  Assign priorities (P1, P2, P3, etc.) to each story, where P1 is the most critical.
  Think of each story as a standalone slice of functionality that can be:
  - Developed independently
  - Tested independently
  - Deployed independently
  - Demonstrated to users independently
-->

> **Constitution Alignment**
> - Describe the failing tests (unit/integration) that will be written first for each story and the
>   mocks/stubs that keep them hermetic.
> - Call out the automation (lint, CI pipelines, scripts) that will enforce success criteria.
> - Explain how the story keeps code readable and what rationale will be preserved in comments or ADRs.

### User Story 1 - Researcher converts a paper to markdown (Priority: P1)

As a research engineer, I want to run a single CLI command that ingests a PDF such as
`/home/rlan/Downloads/streampetr.pdf` and outputs a markdown package ready for review so that I can
quickly summarize academic papers.

**Why this priority**: This is the core promise of the tool; without reliable end-to-end conversion,
no downstream workflow exists.

**Independent Test**: Execute the CLI against the sample PDF with automation scripts; verify that
the command completes with exit code 0, produces a markdown file with all section headers in the
same order as the source, and emits a manifest tying outputs to inputs.

**Acceptance Scenarios**:

1. **Given** a reachable GROBID Docker service and the CLI running inside the mandated conda
   environment, **When** the user supplies a PDF path and output directory, **Then** the CLI fetches
   the XML scaffold, converts every page to a 300 DPI image, and stores intermediate artifacts.
2. **Given** a converted markdown file, **When** the user inspects the section structure,
   **Then** headings, numbering, and hierarchy match the PDF outline detected by GROBID.

---

### User Story 2 - Analyst catalogues visual assets and equations (Priority: P1)

As a technical writer, I need the CLI to extract all figures, tables, algorithms, and equations with
their captions so I can reuse them in downstream documents without reopening the PDF.

**Why this priority**: Visual assets and formulas are critical for research accuracy; missing one
invalidates the markdown handoff.

**Independent Test**: Run the CLI on a PDF that contains known counts of figures/tables/algorithms;
assert via automated tests that the output manifest counts match, the crop images are 300 DPI, and
captions/equations appear inline in markdown.

**Acceptance Scenarios**:

1. **Given** a PDF page containing multiple figures and tables, **When** the VLM model
   `/ssd4/models/Qwen/Qwen3-VL-8B-Instruct-FP8` is invoked, **Then** it produces one crop per asset,
   keeps full figures intact, and associates each with the correct caption text.
2. **Given** a page with equations, **When** the CLI requests extraction, **Then** it stores the
   LaTeX-renderable expression in markdown format, preserving numbering and references.

---

### User Story 3 - Reviewer audits fidelity (Priority: P2)

As a reviewer, I need an automated fidelity evaluation that compares the generated markdown against
the source PDF so I can trust the conversion without manual spot checks.

**Why this priority**: The review step prevents publishing incomplete or inaccurate markdown outputs.

**Independent Test**: Trigger the CLI verification mode; confirm it runs the local VLM comparison,
produces a structured report covering text layout, assets, and equations, and stores it alongside the
markdown without altering the conversion output.

**Acceptance Scenarios**:

1. **Given** a completed markdown conversion, **When** the CLI calls the VLM review, **Then** it
   outputs a report scoring text structure, figure/table alignment, and equation accuracy, and saves
   the report even if issues are detected.
2. **Given** discrepancies (e.g., missing figure), **When** the review runs, **Then** it flags the
   mismatch with page/section references so the user can troubleshoot.

---

[Add more user stories as needed, each with an assigned priority]

### Edge Cases

- PDFs with mixed single-column and two-column layouts on the same page must still produce
  left-to-right, top-to-bottom text ordering.
- Papers containing fold-out images or figures spanning multiple pages require the CLI to detect and
  stitch related crops while keeping associations intact.
- GROBID service outages or malformed XML responses must yield actionable error messages and skip
  destructive retries.
- OCR discrepancies (e.g., ligatures, mathematical symbols) must be reconciled with GROBID text, and
  unresolved conflicts documented in the output manifest.
- Papers exceeding memory limits (hundreds of pages) should stream processing page-by-page instead of
  loading the entire document to avoid crashes.

## Requirements *(mandatory)*

<!--
  ACTION REQUIRED: The content in this section represents placeholders.
  Fill them out with the right functional requirements.
-->

> Keep requirements intentionally small and reference how each one upholds KISS, DRY, SOLID, and YAGNI.
> Explicitly note the automated checks that will verify the behavior to maintain TDD coverage.

### Functional Requirements

- **FR-001**: The CLI MUST accept a local PDF path plus output directory arguments and verify it is
  running inside the `/ssd4/envs/llm_py310_torch271_cu128` conda environment before processing.
- **FR-002**: The tool MUST call the existing GROBID Docker service available on `http://localhost:8070`,
  retrieve the TEI XML scaffold, and cache it for downstream reconciliation, failing fast with clear
  errors if unreachable.
- **FR-003**: The CLI MUST rasterize every PDF page into a 300 DPI image and persist images with
  deterministic naming (e.g., `page-001.png`) for later VLM/OCR steps.
- **FR-004**: The system MUST orchestrate the local VLM
  `/ssd4/models/Qwen/Qwen3-VL-8B-Instruct-FP8` to detect figures, tables, algorithms, and equations,
  ensuring the count of extracted assets matches the paper counts reported by GROBID.
- **FR-005**: Each detected figure/table/algorithm MUST be saved as a whole-image crop unless it
  inherently spans multiple pages, and its caption text MUST be stored and linked in the markdown.
- **FR-006**: Equations MUST be exported as markdown-renderable LaTeX strings, preserving numbering
  and in-text references.
- **FR-007**: The CLI MUST analyze layout using the OCR model `/ssd4/models/datalab-to/chandra`,
  merge OCR text with the GROBID scaffold, and resolve conflicts by favoring OCR content while
  logging any corrections back into the scaffold.
- **FR-008**: The final markdown output MUST mirror the section hierarchy from GROBID, embed links to
  extracted assets, and include a manifest summarizing page-to-section mappings plus any anomalies.
- **FR-009**: After conversion, the CLI MUST re-invoke the VLM to produce a fidelity evaluation
  (structure, assets, equations) and save that report alongside the markdown without mutating the
  generated content.
- **FR-010**: Automation hooks MUST exist (`make test` or equivalent) to run deterministic regression
  tests covering PDF ingestion, asset counts, and evaluation reporting using fixture documents.

### Key Entities *(include if feature involves data)*

- **ConversionJob**: Captures input file metadata, environment checks, timestamps, and references to
  artifacts (XML, page images, markdown, reports).
- **LayoutPage**: Represents a single PDF page with pointers to its rasterized image, OCR text,
  detected columns, and asset bounding boxes for reconciliation logic.
- **AssetCatalog**: Structured listing of every figure, table, algorithm, and equation, including
  captions, LaTeX, references, and output file paths.
- **MarkdownPackage**: Bundle containing the final markdown, embedded asset links, manifest, and
  evaluation report for delivery to the user.

## Success Criteria *(mandatory)*

<!--
  ACTION REQUIRED: Define measurable success criteria.
  These must be technology-agnostic and measurable.
-->

### Measurable Outcomes

- **SC-001**: 100% of conversion runs produce markdown whose section ordering matches the PDF outline
  as reported by GROBID (verified via automated comparison).
- **SC-002**: Figures, tables, algorithms, and equations extracted per run match the counts reported
  by the evaluation report within a tolerance of zero missing items.
- **SC-003**: At least 95% of equations in validation PDFs are rendered with mathematically identical
  LaTeX (spot-checked via automated symbol comparison).
- **SC-004**: The entire CLI workflow completes within 10 minutes for a 20-page paper on reference
  hardware, including the fidelity evaluation step.

## Assumptions & Dependencies

- GROBID Docker service remains accessible at `http://localhost:8070`; outages are outside the CLI
  scope but must be surfaced to users.
- Local models `/ssd4/models/Qwen/Qwen3-VL-8B-Instruct-FP8` and `/ssd4/models/datalab-to/chandra`
  stay installed with necessary runtime dependencies and GPU access.
- Users launch the CLI from the specified conda environment; the tool only warns (does not manage)
  environment activation.
- Sample PDFs (including `streampetr.pdf`) are available for automated regression tests and QA.

## Rationale & Readability Notes *(mandatory)*

- GROBID supplies reliable structure, so it is used strictly as the scaffold while OCR text provides
  the human-readable body, keeping intent clear when reconciling discrepancies.
- Local models are mandated for privacy and latency; the CLI must describe in docs why each model is
  invoked and how their outputs map to markdown sections for reviewer transparency.
- Automation-first (tests + evaluation reports) ensures every change is validated; comments will
  explain reconciliation decisions (e.g., when OCR overrides GROBID) so future maintainers understand
  tradeoffs.
- Any unavoidable complexity (e.g., multi-page figures) must be justified inline in specs/ADRs with
  a plan to simplify once upstream models support richer annotations.
