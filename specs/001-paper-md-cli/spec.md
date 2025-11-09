# Feature Specification: Paper-to-Markdown CLI

**Feature Branch**: `001-paper-md-cli`  
**Created**: 2025-11-07  
**Status**: Draft  
**Input**: User description: "Create a cli tool that takes as input a local pdf file (usually an academic paper) and convert it into a corresponding markdown file..."

## Overview & Context

Researchers currently juggle PDFs, local OCR/VLM models, and ad-hoc scripts to summarize papers, which
is brittle and unrepeatable. Paper2MD introduces a Typer CLI that must run inside
`/ssd4/envs/llm_py310_torch271_cu128`, call the co-located GROBID + model stack, and emit a
markdown+manifest package suitable for review. Conversion happens via `paper2md convert`, while the
fidelity audit runs separately via `paper2md verify` so teams can re-check outputs on demand. The CLI
keeps all processing local (privacy mandate), streams large PDFs to respect memory caps, and documents
every reconciliation decision so future audits understand why OCR or VLM outputs were trusted. Success
depends on deterministic automation (`make test`) and structured artifacts that downstream tooling can
rely on without re-opening the original PDF.

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

**Independent Test**: Execute `paper2md convert` against the sample PDF with automation scripts;
verify that the command completes with exit code 0, produces a markdown file with all section headers
in the same order as the source, and emits a manifest tying outputs to inputs.

**Acceptance Scenarios**:

1. **Given** a reachable GROBID Docker service and the CLI running inside the mandated conda
   environment, **When** the user supplies a PDF path and output directory, **Then** the CLI fetches
   the XML scaffold, converts every page to a 300 DPI image, and stores intermediate artifacts.
2. **Given** a converted markdown file, **When** the user inspects the section structure,
   **Then** headings, numbering, and hierarchy match the PDF outline detected by GROBID.

**Failing Tests First**:
- `tests/unit/test_cli.py::test_convert_requires_conda_env` (Typer CliRunner + monkeypatched
  `os.environ`) guards env validation before any pipeline step.
- `tests/unit/pipeline/test_grobid.py::test_fetch_scaffold_caches_xml` mocks `requests` to fail fast
  when the Docker service is unreachable.
- `tests/integration/test_end_to_end.py::test_convert_creates_markdown_manifest` uses fixture PDFs and
  stubbed adapters to assert section ordering before implementation exists.

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

**Failing Tests First**:
- `tests/unit/pipeline/test_vlm_extractor.py::test_detects_expected_assets` stubs the Qwen3-VL calls
  to demand deterministic crops/counts.
- `tests/unit/pipeline/test_ocr.py::test_column_ordering` simulates mixed column layouts to force the
  reconciler to fix ordering.
- `tests/unit/models/test_manifest_assets.py::test_caption_and_equation_numbering` plus
  `tests/unit/models/test_equations.py::test_latex_accuracy_against_goldens` assert numbering/accuracy
  before writer code exists.
- `tests/integration/test_end_to_end.py::test_assets_embedded_with_counts` introduces a PDF fixture
  whose golden manifest must match extracted counts.

---

### User Story 3 - Reviewer audits fidelity (Priority: P2)

As a reviewer, I need an automated fidelity evaluation that compares the generated markdown against
the source PDF so I can trust the conversion without manual spot checks.

**Why this priority**: The review step prevents publishing incomplete or inaccurate markdown outputs.

**Independent Test**: Trigger `paper2md verify` on a completed conversion package; confirm it runs the
local VLM comparison, produces a structured report covering text layout, assets, and equations, and
stores it alongside the markdown without altering the conversion output.

**Acceptance Scenarios**:

1. **Given** a completed markdown conversion, **When** the user runs `paper2md verify`, **Then** the
   CLI outputs a report scoring text structure, figure/table alignment, and equation accuracy, and
   saves the report even if issues are detected.
2. **Given** discrepancies (e.g., missing figure), **When** the verification command runs, **Then** it
   flags the mismatch with page/section references so the user can troubleshoot before re-delivery.

**Failing Tests First**:
- `tests/unit/services/test_evaluation.py::test_scores_record_discrepancies` mocks VLM responses to
  demand structure/asset/equation scoring.
- `tests/unit/test_cli.py::test_verify_command_runs_without_convert` ensures the Typer command wiring
  exists for verification-only runs.
- `tests/integration/test_end_to_end.py::test_verification_flags_corruption` corrupts markdown outputs
  and expects the evaluation report to persist discrepancies before implementation.

---

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
  running inside the `/ssd4/envs/llm_py310_torch271_cu128` conda environment before processing,
  aborting with an actionable error message if the check fails (the tool does not auto-activate the
  environment).
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
  and in-text references via the `EquationRef` schema described in Key Entities.
- **FR-007**: The CLI MUST analyze layout using `/ssd4/models/Qwen/Qwen3-VL-8B-Instruct-FP8` in OCR mode,
  merge OCR text with the GROBID scaffold, and resolve conflicts by favoring OCR content while
  logging any corrections back into the scaffold.
- **FR-008**: The final markdown output MUST mirror the section hierarchy from GROBID, embed links to
  extracted assets, and include a manifest summarizing page-to-section mappings plus any anomalies.
- **FR-009**: The CLI MUST expose a `paper2md verify` command that re-invokes the VLM on an existing
  conversion package to produce a fidelity evaluation (structure, assets, equations) and save that
  report alongside the markdown without mutating the generated content.
- **FR-011**: The manifest MUST record a cryptographic checksum of the markdown output and verify the
  checksum whenever the package is read or delivered.

### Non-Functional Requirements

- **NFR-001**: The CLI MUST complete a 20-page conversion run (the `paper2md convert` pipeline) with an
  average processing time of ≤1 minute per page (≤20 minutes total for the convert fixture). Verification
  is measured separately. The orchestrator MUST emit per-page duration metrics (start/end timestamps,
  page index) that are persisted to logs and perf fixtures for regression tests so the rolling average
  can be enforced.
- **NFR-002**: Structured logging MUST record every external call (GROBID, pdf2image, models) with a
  consistent correlation identifier (`job_id` + `page_id` where applicable) so outages are diagnosable
  post-run and traces can be reconstructed.
- **NFR-003**: The tool MUST degrade gracefully when GROBID or local models are unreachable by emitting
  actionable error messages and skipping destructive retries, per constitution Principle IV.
- **NFR-004**: Documentation (`README`, `quickstart`, manifest schema comments) MUST explain why OCR
  overrides GROBID or why evaluation heuristics were chosen, satisfying Principle V.
- **NFR-005**: Automation scripts (`make lint`, `make type`, `make test`, CI workflows) MUST block
  merges unless ruff, mypy, unit, integration, and regression tests for convert/verify pass, tying
  together the deterministic hooks required for FR-010 (now folded here) and Principle III.

### Key Entities *(include if feature involves data)*

- **ConversionJob**: Captures input file metadata, environment checks, timestamps, and references to
  artifacts (XML, page images, markdown, reports).
- **LayoutPage**: Represents a single PDF page with pointers to its rasterized image, OCR text,
  detected columns, and asset bounding boxes for reconciliation logic.
- **AssetCatalog**: Structured listing of every figure, table, algorithm, and equation, including
  captions, LaTeX, references, and output file paths.
- **EquationRef**: Normalized description of each equation, including its LaTeX source, numbering,
  page index, bounding box, and reference anchors so markdown and evaluation steps can link back to
  the original PDF coordinates.
- **MarkdownPackage**: Bundle containing the final markdown, embedded asset links, manifest, and
  evaluation report for delivery to the user.
- *TelemetryRecord* and *EvaluationReport* definitions live in `specs/001-paper-md-cli/data-model.md`.
  They provide the per-page timing metrics (used by NFR-001) and checksum-validated evaluation outputs
  required by FR-009/FR-011; reference that document for field-level details.

## Success Criteria *(mandatory)*

<!--
  ACTION REQUIRED: Define measurable success criteria.
  These must be technology-agnostic and measurable.
-->

### Measurable Outcomes

- **SC-001**: 100% of conversion runs produce markdown whose section ordering matches the PDF outline
  as reported by GROBID (verified via automated comparison).
- **SC-002**: Figures, tables, algorithms, and equations extracted per run match the counts reported
  by the evaluation report (when `paper2md verify` is executed) within a tolerance of zero missing
  items.
- **SC-003**: At least 95% of equations in validation PDFs are rendered with mathematically identical
  LaTeX (validated via automated comparison tests against golden fixtures).
- **SC-004**: The `paper2md convert` workflow averages ≤1 minute per page on the 20-page reference
  paper (≤20 minutes total), with evidence captured in perf reports. Verification runs may add time but
  are tracked independently.

## Assumptions & Dependencies

- GROBID Docker service remains accessible at `http://localhost:8070`; outages are outside the CLI
  scope but must be surfaced to users.
- Local model `/ssd4/models/Qwen/Qwen3-VL-8B-Instruct-FP8` stays installed with necessary runtime
  dependencies and GPU access for both asset detection and OCR passes.
- Users launch the CLI from the specified conda environment; the tool verifies this requirement and
  exits with an actionable error if mismatched, but it does not attempt to manage/activate the
  environment automatically.
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
