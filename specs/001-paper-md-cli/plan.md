# Implementation Plan: Paper2MD CLI

**Branch**: `001-paper-md-cli` | **Date**: 2025-11-09 | **Spec**: `/home/rlan/projects/paper2md/specs/001-paper-md-cli/spec.md`
**Input**: Feature specification from `/specs/001-paper-md-cli/spec.md`

## Summary

Deliver a Typer-based CLI that converts academic PDFs into markdown packages locally while honoring
privacy (no network calls beyond localhost). `paper2md convert` validates the mandated conda env,
streams pages through GROBID + Qwen3-VL (assets + OCR), reconciles outputs into markdown + manifest,
and enforces checksums. `paper2md verify` re-loads an existing package, re-validates the checksum, and
runs the VLM fidelity audit, producing a discrepancy report stored alongside the conversion. Telemetry
scripts capture per-page runtimes so the 20-page fixture averages ≤1 minute per page.

## Technical Context

**Language/Version**: Python 3.10 inside `/ssd4/envs/llm_py310_torch271_cu128`  
**Primary Dependencies**: Typer (CLI), pydantic (manifest schemas), requests/httpx (GROBID), pdf2image + Poppler (rasterization), Pillow/numpy/OpenCV (image ops), PyTorch (Qwen3-VL wrappers)  
**Storage**: Local filesystem (output bundles under user-selected directories, plus `<output>/cache/tei` for TEI reuse)  
**Testing**: pytest + pytest-mock with golden fixture PDFs; automation via `make lint` (ruff) + `make type` (mypy) + `make test`  
**Target Platform**: Single Linux workstation with GPU access sharing the GROBID container and VLM checkpoints  
**Project Type**: Single CLI package under `src/paper2md`  
**Performance Goals**: Average ≤1 minute per page on the 20-page reference paper (≤20 minutes total) and ≤30 s per page in streaming telemetry  
**Constraints**: Must fail fast outside the mandated conda env, operate fully offline, stream >100-page PDFs without exceeding memory caps, persist structured logs with correlation IDs, and verify checksums before delivering or re-reading packages  
**Scale/Scope**: One conversion per CLI invocation, but designed for batch scripting; artifacts must be deterministic for downstream automation

## Constitution Check

- `Readable-first design`: CLI commands map 1:1 with pipeline modules (`config`, `pipeline/*`, `services/*`), each with docstrings explaining external constraints (env checks, cache retention, checksum rationale).
- `TDD-first workflow`: Every adapter (CLI, GROBID, rasterizer, VLM, OCR, markdown writer, checksum, telemetry) gets failing pytest modules before implementation; integrations rely on fixture PDFs with stubbed HTTP/model clients.
- `Automation coverage`: `Makefile` targets chain ruff + mypy + pytest; CI mirrors these plus `tests/integration/test_perf_timings.py` and the promoted 20-page SLA script.
- `Intentional simplicity`: Scope limited to synchronous CLI commands and filesystem artifacts—no services, queues, or remote storage until explicitly demanded.
- `Explain the why`: research.md captures tradeoffs (Typer vs Click, pdf2image streaming settings, checksum-on-verify), while docs (`quickstart.md`, manifest comments) preserve rationale for overrides and telemetry thresholds.

## Project Structure

### Documentation (this feature)

```text
specs/001-paper-md-cli/
├── plan.md          # This document
├── research.md      # Phase 0 findings
├── data-model.md    # Phase 1 entities
├── quickstart.md    # Developer runbook
├── contracts/       # OpenAPI contracts for CLI-equivalent flows
└── tasks.md         # Produced separately by /speckit.tasks
```

### Source Code (repository root)

```text
src/
└── paper2md/
    ├── __init__.py
    ├── cli.py              # Typer entrypoints (convert, verify)
    ├── config.py           # Env + path validation
    ├── pipeline/
    │   ├── __init__.py
    │   ├── orchestrator.py
    │   ├── grobid.py
    │   ├── rasterizer.py
    │   ├── vlm_extractor.py
    │   ├── ocr.py
    │   └── reconciler.py
    ├── models/
    │   ├── manifest.py
    │   └── telemetry.py
    └── services/
        ├── markdown_writer.py
        ├── storage.py
        └── evaluation.py

tests/
├── unit/
│   ├── test_cli.py
│   ├── pipeline/
│   ├── models/
│   ├── services/
│   └── test_docs_fidelity.py
├── integration/
│   ├── test_end_to_end.py
│   └── test_perf_timings.py
└── data/
    └── sample_papers/
```

**Structure Decision**: Maintain a single CLI project with clearly separated pipeline adapters,
models, and services. Tests mirror the runtime layout, enabling focused TDD per component plus
fixture-driven integration suites.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| _None_ | — | — |

## Phase 0 – Research & Unknown Resolution

**Objectives**: Resolve tooling best practices for Typer CLI ergonomics, GROBID HTTP usage, streaming
rasterization, Qwen3-VL (assets + OCR + evaluation), telemetry instrumentation, and checksum
verification on subsequent reads.

**Key Findings (see `research.md`)**:
- Typer remains the most readable option thanks to type-hinted commands and auto-generated help; Click
  would add boilerplate without value.
- GROBID interactions stay HTTP-based with checksum-scoped TEI caches and backoff-limited retries; file
  watching was rejected as overly coupled to container internals.
- pdf2image + Poppler at 300 DPI with single-page streaming best balances fidelity + memory; Ghostscript
  or PyMuPDF add dependencies without new guarantees.
- The shared Qwen3-VL checkpoint can power asset detection, OCR, and evaluation, minimizing model
  surface area; specialized multi-model setups were deferred to avoid GPU fragmentation.
- Telemetry requires per-page start/end timestamps plus aggregate metrics stored in manifest +
  `docs/perf-report.md`, enabling gating on the ≤1 minute average SLA.
- Checksum verification must occur before both initial delivery and every `paper2md verify` run; a
  manifest reader helper will fail fast if files were tampered with.

Gate result: PASS — all clarifications resolved.

## Phase 1 – Design, Data, and Contracts

**Outputs**:
- `data-model.md`: Documents ConversionJob, LayoutPage, AssetCatalog, EquationRef, MarkdownPackage,
  TelemetryRecord, and EvaluationReport entities with validation rules for checksum + telemetry data.
- `contracts/conversion-api.yaml`: OpenAPI spec covering `POST /api/conversions` (convert),
  `GET /api/conversions/{jobId}` (status/manifest), and `POST /api/conversions/{jobId}/evaluation`
  (`paper2md verify` equivalent) with checksum verification guarantees.
- `quickstart.md`: Developer steps for env activation, running `paper2md convert`, inspecting outputs,
  executing `paper2md verify`, and invoking automation scripts.
- Agent context updated via `.specify/scripts/bash/update-agent-context.sh codex`, capturing the active
  toolchain for downstream agents.

**Design Highlights**:
- Manifest schema now includes checksum metadata plus `TelemetryRecord` arrays so tests can assert per-
  page durations.
- Verification flow requires checksum re-validation prior to VLM scoring, ensuring tampering is caught
  before comparisons.
- Quickstart and contract docs emphasize the separation between convert and verify, clarifying when
  evaluation data becomes available.

Gate result: PASS — constitution principles remain satisfied with documented rationale.

## Phase 2 – Foundational Build

**Goal**: Establish automation scaffolding, fixtures, and failure-first tests before implementation.

**Deliverables**:
1. Project scaffolding — Typer CLI skeleton, Poetry config, Makefile targets, lint/type configs, CI
   placeholder, and README references.
2. Test foundations — pytest layout, orchestrator/storage unit tests (failing), fixture PDFs, golden
   manifest/evaluation examples, CLI behavior contract doc.
3. Reliability hardening — adapter outage simulations, manifest checksum tests, telemetry test suite,
   and the promoted 20-page SLA run (T057) that enforces the ≤1 minute/page average.

**Gates & Dependencies**:
- Tests T009–T013, T054, T018–T019, T057, T062–T064 must fail before code lands; green runs unlock
  Phase 3 story work.
- Automation scripts (`make lint`, `make type`, `make test`, telemetry runners) must succeed locally
  and inside CI to uphold Principle III.
- Documentation (`docs/cli-behavior.md`, `docs/perf-report.md`) must be updated alongside scaffolding
  so intent + telemetry evidence stay discoverable.

Next step after Phase 2: run `/speckit.tasks` with this plan to schedule story-specific work.
