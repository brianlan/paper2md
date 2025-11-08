# Implementation Plan: Paper-to-Markdown CLI

**Branch**: `001-paper-md-cli` | **Date**: 2025-11-07 | **Spec**: specs/001-paper-md-cli/spec.md
**Input**: Feature specification from `/specs/001-paper-md-cli/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Create a conda-bound CLI that ingests an academic PDF, derives a TEI XML scaffold via the local
GROBID service (`http://localhost:8070`), rasterizes each page at 300 DPI, and coordinates the local
Qwen3-VL model (for figures/tables/algorithms/equations *and* OCR text) to produce a fully-structured
markdown package. The CLI enforces the mandated conda environment, instruments per-page timing and
correlation-ID logs for every external call, and re-runs the VLM to audit fidelity so researchers can
trust the conversion without manual QA.

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: Python 3.10 (per `/ssd4/envs/llm_py310_torch271_cu128`)  
**Primary Dependencies**: Typer (CLI), pydantic (manifests), requests/httpx (GROBID), pdf2image/poppler, Pillow, numpy, OpenCV, PyTorch for VLM/OCR wrappers  
**Storage**: Local filesystem outputs (markdown, manifest, crops)  
**Testing**: pytest with pytest-mock + golden fixtures  
**Target Platform**: Linux workstation with GPU access (same host as models/GROBID)  
**Project Type**: Single-project CLI (`src/paper2md/...`)  
**Tooling**: Poetry for dependency management and scripts (`poetry run ...`)  
**Performance Goals**: Convert a 20-page paper including audit in ≤10 minutes; per-page processing ≤30 s with timing telemetry persisted  
**Constraints**: Must run inside prescribed conda env (fail fast if mismatched), offline (local services only), memory-aware streaming for >100 pages  
**Scale/Scope**: Single concurrent conversion per invocation; designed for batch scripting but not multi-tenant service

### Performance & Streaming Strategy

- **Streaming-first pipeline**: Rasterizer processes one page at a time, writing 300 DPI images to disk and discarding intermediate buffers to satisfy the “hundreds of pages” edge case.
- **Batch-size guardrails**: VLM/OCR adapters accept configurable page batches (default = 1) and enforce GPU memory ceilings; exceeding limits downgrades to single-page processing with warning logs.
- **Perf instrumentation**: The orchestrator records per-page start/end timestamps plus duration metrics that feed both logs and `docs/perf-report.md`.
- **Regression artifacts**: Large-PDF fixture (`tests/data/sample_papers/streaming_100p/`) and performance scripts keep SLA measurements reproducible.

### Caching Strategy

- **Scope**: TEI XML cache is scoped per conversion job (output directory) but leverages checksum-based reuse when the same input PDF is processed multiple times.
- **Location**: Cached scaffolds live under `<output>/cache/tei/` with filenames derived from the PDF checksum to avoid collisions.
- **Invalidation**: Any change to the input PDF or GROBID version triggers a new fetch; stale cache entries include metadata (checksum + grobid_version) so tests can verify invalidation.
- **Test coverage**: `tests/unit/pipeline/test_grobid.py` gains cache hit/miss scenarios, and contract docs describe operational expectations for cache persistence.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- `Readable-first design`: Module boundary plan splits CLI, pipeline orchestration, and adapters for
  GROBID/VLM/OCR so each step documents its contract; manifest schemas include intent comments.
- `TDD-first workflow`: Tests will be written before pipeline code using fixture PDFs + mocked
  adapters for GROBID, pdf2image, and VLM/OCR responses; red-green-refactor enforced via pytest.
- `Automation coverage`: `make test` (or `task test`) runs lint (ruff), type check (mypy), and all
  pytest suites; CI mirrors the same commands and blocks merges until green.
- `Intentional simplicity`: MVP limits scope to single synchronous CLI command, streaming pipeline,
  and filesystem outputs; no background queueing or web UI until proven demand.
- `Explain the why`: research.md + inline docstrings capture reasoning for adapter boundaries,
  equation reconciliation, and tradeoffs (e.g., why OCR overrides GROBID text in conflicts).

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Docs & Operational Guides

```text
docs/
├── cli-behavior.md      # Exit codes, env checks, logging conventions
├── fidelity-review.md   # Evaluation workflow + report interpretation
└── perf-report.md       # SLA metrics, perf fixture results, streaming data
```

### Source Code (repository root)
<!--
  ACTION REQUIRED: Replace the placeholder tree below with the concrete layout
  for this feature. Delete unused options and expand the chosen structure with
  real paths (e.g., apps/admin, packages/something). The delivered plan must
  not include Option labels.
-->

```text
src/
├── paper2md/
│   ├── cli.py                 # Typer entrypoint + argument parsing
│   ├── config.py              # Env + path validation
│   ├── pipeline/
│   │   ├── __init__.py
│   │   ├── orchestrator.py    # High-level pipeline controller
│   │   ├── grobid.py          # XML scaffold client
│   │   ├── rasterizer.py      # pdf2image wrapper
│   │   ├── vlm_extractor.py   # Qwen3-VL asset detection
│   │   ├── ocr.py             # Qwen3-VL OCR integration
│   │   └── reconciler.py      # Merge scaffold + OCR + assets
│   ├── models/
│   │   ├── manifest.py        # Pydantic data classes
│   │   └── evaluation.py
│   └── services/
│       ├── markdown_writer.py # Markdown + asset embedding helpers
│       ├── storage.py         # Filesystem I/O helpers
│       └── evaluation.py      # VLM fidelity review orchestrator
tests/
├── unit/
│   ├── test_cli.py
│   ├── pipeline/
│   └── services/
├── integration/
│   ├── test_end_to_end.py
│   └── fixtures/
└── data/
    └── sample_papers/
```

**Structure Decision**: Single CLI project under `src/paper2md` with dedicated pipeline modules and
services (markdown writer, storage, evaluation); pytest-based tests stay organized by unit vs.
integration to keep responsibilities readable.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| _None_ | — | — |

## Phase 0 – Research & Unknown Resolution

All unknowns have been resolved and captured in `research.md`. Key tasks completed:

1. **Environment confirmation** – Validated Python 3.10 + Typer stack fits conda env and local GPU workflow.
2. **GROBID access pattern** – Documented HTTP retries, caching, and why localhost:8070 is authoritative.
3. **Rasterization strategy** – Selected pdf2image with streaming to control memory for >100 pages.
4. **Asset detection pipeline** – Defined per-page Qwen3-VL invocation contract plus JSON manifest schema.
5. **OCR reconciliation** – Established column detection + TEI merge rules with logging of overrides.
6. **Instrumentation & observability** – Defined per-page timing hooks and correlation-ID logging across adapters.
7. **Fidelity review loop** – Defined evaluation report schema and persistence.
8. **Automation approach** – Locked in pytest + ruff + mypy executed via `make test`.

Artifacts: `specs/001-paper-md-cli/research.md`

## Phase 1 – Design, Data, and Contracts

Deliverables produced this phase:

- `data-model.md`: Formalizes ConversionJob, LayoutPage, AssetCatalog, EquationRef, MarkdownPackage.
- `contracts/conversion-api.yaml`: REST contract mirroring CLI pipeline for future service wrappers.
- `quickstart.md`: Developer runbook covering env activation, sample command, output review, and testing.
- `AGENTS.md`: Updated via `.specify/scripts/bash/update-agent-context.sh codex` to record new tech context.

Design Highlights:

1. **Pipeline modules** follow single responsibility (client adapters vs. orchestrator vs. storage).
2. **Data validation** ensures asset counts match TEI and manifest includes checksums.
3. **CLI UX** remains single command with explicit arguments; manifest stores warnings for figure/table mismatches.
4. **Testing layout** distinguishes unit adapters from integration runs with fixture PDFs.
5. **Multi-page figures**: US2 tasks introduce detection + stitching logic plus regression fixtures so multi-page assets stay intact.
6. **Equation fidelity**: Dedicated LaTeX comparison tests enforce the 95% accuracy success criterion.
7. **Checksum verification**: Manifest hooks and storage checks ensure tampering is detected before outputs leave the CLI.
8. **Large-PDF streaming tests**: Polish phase adds >100-page stress tests documenting per-page processing and memory targets.

## Constitution Check (Post-Design)

- `Readable-first design`: Plan now details dedicated modules and manifest schemas + documentation references—PASS.
- `TDD-first workflow`: Pytest-first strategy plus fixture PDFs + mocks enumerated—PASS.
- `Automation coverage`: `make test` pipeline (ruff, mypy, pytest) specified and to be mirrored in CI—PASS.
- `Intentional simplicity`: Scope limited to synchronous CLI, filesystem outputs, and single evaluation pass—PASS.
- `Explain the why`: research.md and inline comments planned for reconciliation decisions—PASS.

No violations detected; Complexity Tracking remains empty.

## Phase 2 – Foundational Build

- **Goal**: Lock in automation scaffolding, fixtures, and blocking tests before any user story code ships.
- **Deliverables**: pytest layout (`tests/unit`, `tests/integration`), baseline fixtures, orchestrator/storage unit tests, CLI behavior contract (`docs/cli-behavior.md`), and checksum manifest schemas.
- **Gates**: Tests in T009–T013 + outage handling (T054) must fail first, CI automation (`make lint`, `make test`) must be runnable locally, and constitution principles I–III are re-asserted via readable module boundaries + TDD evidence.
- **Dependencies**: Completes after Phase 1; prerequisite for telemetry and every user story.

## Phase 2b – Observability & Telemetry (Pre-Story Gate)

- **Goal**: Satisfy NFR-001/002 before feature work by instrumenting per-page metrics, correlation IDs, and SLA tests.
- **Deliverables**: Telemetry tests (`tests/integration/test_perf_timings.py`, `tests/unit/pipeline/test_logging.py`), orchestrator instrumentation (T062–T064), and perf-report documentation with captured metrics.
- **Gate**: User Story phases may not start until telemetry tests fail/pass cycle is complete; ensures Principle III (“Relentless Automation”) remains enforceable.

## Phase 3 – User Story 1 (Researcher Conversion MVP)

- **Scope**: CLI command (`paper2md convert`) covering env validation, GROBID fetch + cache, rasterization, scaffold reconciliation (with OCR correction logging surfaced to manifest + logs), markdown writer, and checksum enforcement.
- **Tests first**: T014–T019 + T065 + T066 cover CLI, GROBID client/cache reuse, rasterizer, reconciler logging, manifest checksums, and end-to-end flow with mocked adapters.
- **Exit criteria**: Integration test proves PDF→markdown pipeline, telemetry hooks stay green, correction logs are persisted/readable, README/docs explain OCR overrides (Principle V), and manifest checksum verification blocks tampered output.
- **Edge-case handling**: Large-PDF streaming + fixture/documentation tasks (T056, T060, T061) are part of this phase so MVP already honors the memory guarantees from the spec.

## Phase 4 – User Story 2 (Asset & Equation Catalog)

- **Scope**: Qwen3-VL asset detection, OCR reconciliation, AssetCatalog/EquationRef models, multi-page figure stitching, markdown embedding, regression goldens, and documentation of extraction heuristics (Principle V).
- **Tests first**: T030–T034 expand adapter coverage; integration suite validates counts, LaTeX fidelity, and stitched assets.
- **Dependencies**: Builds on Phase 3 outputs and reuses GROBID cache + telemetry infrastructure.

## Phase 5 – User Story 3 (Fidelity Review)

- **Scope**: Evaluation service orchestrator, CLI `verify` command, discrepancy reporting, manifest linkage, and documentation of interpretation guidance (docs/fidelity-review.md updated per Principle V).
- **Tests first**: T044–T046 extend unit + integration coverage for evaluation flows, including corruption detection.
- **Exit criteria**: Verification can run independently, reports persist alongside markdown, and automation scripts cover both convert + verify workflows.

## Phase N – Polish & Quality Gates

- **Scope**: Structured logging polish, documentation enhancements, performance runs on reference hardware, telemetry annotations, and ADR/comment hygiene.
- **Key moves**: Logging UX (T053), doc/tooling polish (T055), perf reporting (T057), OCR/TEI telemetry (T058), and rationale capture (T059) round out the release once core stories ship.
- **Dependencies**: Execute after primary user stories but continue to honor telemetry + automation gates; performance reports are updated once SLA runs complete.
