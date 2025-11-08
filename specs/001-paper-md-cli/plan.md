# Implementation Plan: Paper-to-Markdown CLI

**Branch**: `001-paper-md-cli` | **Date**: 2025-11-07 | **Spec**: specs/001-paper-md-cli/spec.md
**Input**: Feature specification from `/specs/001-paper-md-cli/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Create a conda-bound CLI that ingests an academic PDF, derives a TEI XML scaffold via the local
GROBID service (`http://localhost:8070`), rasterizes each page at 300 DPI, and coordinates local
models (Qwen3-VL for figures/tables/algorithms/equations, Chandra OCR for text) to produce a
fully-structured markdown package. The CLI must also re-run the VLM to audit fidelity and emit a
report covering structure, assets, and equations so researchers can trust the conversion without
manual QA.

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
**Performance Goals**: Convert a 20-page paper including audit in ≤10 minutes; per-page processing ≤30 s  
**Constraints**: Must run inside prescribed conda env, offline (local services only), memory-aware streaming for >100 pages  
**Scale/Scope**: Single concurrent conversion per invocation; designed for batch scripting but not multi-tenant service

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
│   │   ├── ocr.py             # Chandra OCR integration
│   │   └── reconciler.py      # Merge scaffold + OCR + assets
│   ├── models/
│   │   ├── manifest.py        # Pydantic data classes
│   │   └── evaluation.py
│   └── services/
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
pytest-based tests organized by unit vs. integration to keep responsibilities readable.

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
6. **Fidelity review loop** – Defined evaluation report schema and persistence.
7. **Automation approach** – Locked in pytest + ruff + mypy executed via `make test`.

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

## Constitution Check (Post-Design)

- `Readable-first design`: Plan now details dedicated modules and manifest schemas + documentation references—PASS.
- `TDD-first workflow`: Pytest-first strategy plus fixture PDFs + mocks enumerated—PASS.
- `Automation coverage`: `make test` pipeline (ruff, mypy, pytest) specified and to be mirrored in CI—PASS.
- `Intentional simplicity`: Scope limited to synchronous CLI, filesystem outputs, and single evaluation pass—PASS.
- `Explain the why`: research.md and inline comments planned for reconciliation decisions—PASS.

No violations detected; Complexity Tracking remains empty.
