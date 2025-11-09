# Tasks: 001-paper-md-cli

**Input**: Design documents from `/specs/001-paper-md-cli/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Tests**: Every story lists the pytest modules that must be written (and fail) before implementation per constitution TDD mandate.

**Organization**: Tasks are grouped by user story so each slice can be implemented, tested, and delivered independently.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Task can run in parallel (different files, no dependency ordering)
- **[Story]**: Applies only to user story phases (e.g., [US1])
- Include exact file paths for every task

## Path Conventions

- Source: `src/paper2md/`
- Tests: `tests/`
- Docs: `docs/`, `README.md`, `quickstart.md`
- Data fixtures: `tests/data/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Establish project skeleton, tooling, and documentation references.

- [ ] T001 Create Typer CLI package skeleton in `src/paper2md/__init__.py` and `src/paper2md/cli.py` per plan structure
- [ ] T002 Configure Poetry project metadata and dependencies in `pyproject.toml` (Typer, pydantic, requests, pdf2image, Pillow, numpy, OpenCV, PyTorch)
- [ ] T003 Add reproducible automation targets (`make lint`, `make type`, `make test`) in root `Makefile`
- [ ] T004 [P] Add env/config loader `src/paper2md/config.py` validating `/ssd4/envs/llm_py310_torch271_cu128` and local model paths
- [ ] T005 [P] Scaffold pytest layout (`tests/unit/`, `tests/integration/`, `tests/data/`) with `conftest.py` stubs and sample fixtures
- [ ] T006 [P] Document run/setup flow in `README.md` pointing to `quickstart.md`
- [ ] T007 [P] Pin lint/type settings (`ruff.toml`, `mypy.ini`) and hook them into Poetry scripts
- [ ] T008 [P] Add CI workflow `.github/workflows/ci.yml` mirroring `make lint/type/test`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure + failing tests required before any user story work.

- [ ] T009 [P] Author orchestrator tests in `tests/unit/pipeline/test_orchestrator.py` covering stage sequencing + failure propagation
- [ ] T010 [P] Add storage helper tests in `tests/unit/services/test_storage.py` for path creation, atomic writes, and checksum hooks
- [ ] T011 Create baseline fixture PDF + manifest scaffolding under `tests/data/sample_papers/streampetr/`
- [ ] T012 Seed golden manifest/evaluation artifacts in `tests/data/goldens/streampetr/`
- [ ] T013 Define CLI behavior & exit codes in `docs/cli-behavior.md` (env validation, cache purge, checksum failures)
- [ ] T014 [P] Add manifest checksum generation/verification tests in `tests/unit/models/test_manifest_checksum.py`
- [ ] T015 Implement checksum stubs + TODO guards in `src/paper2md/models/manifest.py` so tests fail pending story work
- [ ] T016 [P] Harden outage handling tests in `tests/unit/pipeline/test_grobid_failures.py` simulating GROBID/model downtime

**Checkpoint**: Foundation is green (tests fail as expected) → proceed to telemetry gate.

---

## Phase 2b: Observability & Telemetry (Pre-Story Gate)

**Purpose**: Enforce NFR-001/NFR-002 before story implementation.

- [ ] T017 [P] Add per-page timing integration suite in `tests/integration/test_perf_timings.py` with assertions for ≤30 s per page
- [ ] T018 [P] Instrument `src/paper2md/pipeline/orchestrator.py` to emit `TelemetryRecord`s (start/end/duration) and write `telemetry.json`
- [ ] T019 [P] Propagate correlation IDs (`job_id`, `page_id`) through `src/paper2md/pipeline/grobid.py`, `vlm_extractor.py`, and `ocr.py` with logging assertions in `tests/unit/pipeline/test_logging.py`
- [ ] T020 Run 20-page SLA fixture `tests/data/sample_papers/streampetr/20p/` via `tests/perf/test_20page_average.py`, failing if average >1 min per page, and persist metrics to `docs/perf-report.md`
- [ ] T021 [P] Capture telemetry schema + usage docs in `docs/perf-report.md` and manifest comments for future enforcement

**Checkpoint**: Telemetry scripts + SLA run are green → unlock user story work.

---

## Phase 3: User Story 1 - Researcher converts a paper to markdown (Priority: P1) 🎯 MVP

**Goal**: `paper2md convert` ingests a local PDF, validates env, streams through GROBID + OCR, and outputs markdown + manifest with checksums.

**Independent Test**: Run `poetry run paper2md convert --pdf <fixture> --output <dir>`; expect exit 0, markdown matching GROBID outline, manifest with section map + checksum verification enforced.

### Tests for User Story 1 (MANDATORY - write first) ⚠️

- [ ] T022 [P] [US1] Add CLI env-guard tests in `tests/unit/test_cli.py::test_convert_requires_env`
- [ ] T023 [P] [US1] Create GROBID client/cache tests in `tests/unit/pipeline/test_grobid.py`
- [ ] T024 [P] [US1] Add rasterizer streaming/300 DPI tests in `tests/unit/pipeline/test_rasterizer.py`
- [ ] T025 [P] [US1] Add reconciler logging/correction tests in `tests/unit/pipeline/test_reconciler_logging.py`
- [ ] T026 [P] [US1] Add markdown writer embedding tests in `tests/unit/services/test_markdown_writer.py`
- [ ] T027 [P] [US1] Add TEI cache reuse tests in `tests/unit/pipeline/test_grobid_cache.py`
- [ ] T028 [P] [US1] Extend integration flow `tests/integration/test_convert_end_to_end.py` to assert section ordering + checksum blocking
- [ ] T029 [P] [US1] Add large-PDF streaming test `tests/integration/test_large_pdf.py` to ensure memory-safe batching

### Implementation for User Story 1

- [ ] T030 [US1] Implement env validation + convert command wiring in `src/paper2md/cli.py`
- [ ] T031 [US1] Flesh out config validation + helpful errors in `src/paper2md/config.py`
- [ ] T032 [US1] Implement GROBID client with TEI cache + purge flag in `src/paper2md/pipeline/grobid.py`
- [ ] T033 [US1] Build pdf2image rasterizer with deterministic filenames in `src/paper2md/pipeline/rasterizer.py`
- [ ] T034 [US1] Implement OCR/TEI reconciler scaffold (without asset placement) in `src/paper2md/pipeline/reconciler.py`
- [ ] T035 [US1] Orchestrate pipeline stages + error handling in `src/paper2md/pipeline/orchestrator.py`
- [ ] T036 [US1] Implement markdown writer baseline in `src/paper2md/services/markdown_writer.py`
- [ ] T037 [US1] Finalize manifest schema + checksum generation/verification in `src/paper2md/models/manifest.py`
- [ ] T038 [US1] Enforce checksum verification before delivery in `src/paper2md/services/storage.py`
- [ ] T039 [US1] Update CLI/documentation (`README.md`, `docs/cli-behavior.md`) with convert usage, cache purge, checksum rationale
- [ ] T040 [US1] Document streaming guarantees + results in `docs/perf-report.md`

**Checkpoint**: `paper2md convert` stands alone—markdown + manifest produced with verified checksum.

---

## Phase 4: User Story 2 - Analyst catalogues visual assets and equations (Priority: P1)

**Goal**: Extract figures/tables/algorithms/equations with correct numbering, captions, crops, and embed them into markdown + manifest.

**Independent Test**: Run convert on asset-rich fixture; manifest counts match TEI, stitched multi-page figures persist, equations renderable with ≥95% accuracy via automated comparison.

### Tests for User Story 2 (MANDATORY - write first) ⚠️

- [ ] T041 [P] [US2] Add VLM extractor contract tests in `tests/unit/pipeline/test_vlm_extractor.py`
- [ ] T042 [P] [US2] Add OCR column-ordering tests in `tests/unit/pipeline/test_ocr.py`
- [ ] T043 [P] [US2] Add AssetCatalog numbering/count tests in `tests/unit/models/test_manifest_assets.py`
- [ ] T044 [P] [US2] Add LaTeX fidelity tests in `tests/unit/models/test_equations.py`
- [ ] T045 [P] [US2] Extend integration test in `tests/integration/test_convert_end_to_end.py` to assert asset/equation outputs vs goldens

### Implementation for User Story 2

- [ ] T046 [US2] Implement Qwen3-VL asset extractor in `src/paper2md/pipeline/vlm_extractor.py`
- [ ] T047 [US2] Implement OCR adapter for ordered text blocks in `src/paper2md/pipeline/ocr.py`
- [ ] T048 [US2] Extend reconciler to insert asset placeholders + references in `src/paper2md/pipeline/reconciler.py`
- [ ] T049 [US2] Serialize AssetCatalog + EquationRef data in `src/paper2md/models/manifest.py`
- [ ] T050 [US2] Embed figures/tables/equations with captions + LaTeX in `src/paper2md/services/markdown_writer.py`
- [ ] T051 [US2] Implement multi-page figure stitching + bounding-box aggregation in `src/paper2md/pipeline/vlm_extractor.py`
- [ ] T052 [US2] Surface mismatch warnings/logs in `src/paper2md/cli.py` and manifest anomalies
- [ ] T053 [US2] Create regression goldens (images + LaTeX) under `tests/data/goldens/assets/`
- [ ] T054 [US2] Document extraction heuristics + rationale in `docs/fidelity-review.md`

**Checkpoint**: Markdown now embeds assets/equations accurately; manifest counts align with TEI.

---

## Phase 5: User Story 3 - Reviewer audits fidelity (Priority: P2)

**Goal**: `paper2md verify` reloads packages, re-validates checksums, runs VLM-based fidelity scoring, and stores discrepancy reports.

**Independent Test**: Convert fixture → run `poetry run paper2md verify --package <dir>`; expect checksum verification, evaluation report persisted even on detected discrepancies, and CLI exit status reflects issues.

### Tests for User Story 3 (MANDATORY - write first) ⚠️

- [ ] T055 [P] [US3] Add evaluation service tests in `tests/unit/services/test_evaluation.py`
- [ ] T056 [P] [US3] Add CLI verify command tests in `tests/unit/test_cli.py::test_verify_runs_without_convert`
- [ ] T057 [P] [US3] Extend integration suite (`tests/integration/test_verify_end_to_end.py`) to corrupt markdown and expect discrepancy logging
- [ ] T058 [P] [US3] Add docs lint/coverage test in `tests/unit/test_docs_fidelity.py` ensuring `docs/fidelity-review.md` stays updated
- [ ] T059 [P] [US3] Add manifest reader checksum tests in `tests/unit/models/test_manifest_reader.py`

### Implementation for User Story 3

- [ ] T060 [US3] Implement VLM evaluation orchestrator in `src/paper2md/services/evaluation.py`
- [ ] T061 [US3] Add `paper2md verify` Typer command + options in `src/paper2md/cli.py`
- [ ] T062 [US3] Ensure manifest reader re-validates checksums before evaluation in `src/paper2md/services/storage.py`
- [ ] T063 [US3] Persist evaluation JSON + link to manifest in `src/paper2md/models/manifest.py`
- [ ] T064 [US3] Update storage helpers to save `evaluation.json` alongside markdown in `src/paper2md/services/storage.py`
- [ ] T065 [US3] Document verification workflow + troubleshooting in `docs/fidelity-review.md` and `quickstart.md`
- [ ] T066 [US3] Update integration harness to auto-run verify after convert in `tests/integration/test_convert_end_to_end.py`

**Checkpoint**: Verification runs independently, enforcing checksum integrity and surfacing fidelity discrepancies.

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Repository-wide refinements post-stories.

- [ ] T067 [P] Add structured logging + progress reporting to `src/paper2md/cli.py` and `src/paper2md/pipeline/orchestrator.py`
- [ ] T068 [P] Refresh developer docs (`README.md`, `docs/cli-behavior.md`, `quickstart.md`) with lessons learned + troubleshooting
- [ ] T069 [P] Re-run 20-page SLA + large-PDF fixtures, archive metrics in `docs/perf-report.md`, and update telemetry thresholds if needed
- [ ] T070 Capture final ADR/rationale for reconciliation & evaluation heuristics in `docs/adr/001-paper2md.md`
- [ ] T071 Add telemetry flags for unresolved OCR vs TEI conflicts in `src/paper2md/models/manifest.py`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)** → prerequisite for all later work.
- **Phase 2 (Foundational)** depends on setup; BLOCKS all user stories until tests/fixtures/docs exist.
- **Phase 2b (Telemetry Gate)** depends on Phase 2; BLOCKS user stories until SLA + logging instrumentation pass.
- **Phase 3 (US1)** begins after telemetry gate; produces MVP convert flow.
- **Phase 4 (US2)** builds atop US1 outputs (needs asset hooks from convert pipeline).
- **Phase 5 (US3)** depends on US1 + US2 artifacts to audit and requires checksum metadata.
- **Phase N (Polish)** runs after desired stories complete.

### User Story Dependencies

- **US1 (P1)**: Requires foundational + telemetry phases only.
- **US2 (P1)**: Requires US1 (uses convert pipeline) plus foundational artifacts.
- **US3 (P2)**: Requires US1 + US2 outputs available for verification.

### Within Each User Story

- Tests fail first → implementation follows.
- Models before services; services before CLI wiring.
- Update docs/automation within the same story phase to keep artifacts independent.

### Parallel Opportunities

- Setup tasks T004–T008 can run concurrently once repo skeleton exists.
- Foundational tests T009–T016 touch different modules and may proceed in parallel.
- Telemetry tasks T017–T021 split between tests, orchestration, and docs, enabling parallel work.
- Within each story, unit tests (e.g., T022–T027, T041–T044, T055–T059) can be divided across contributors while integration tests (T028, T045, T057) gate completion.
- Implementation tasks touching disjoint files (e.g., VLM extractor vs markdown writer) are marked [P] where safe.

---

## Implementation Strategy

1. Finish Phases 1–2b to guarantee automation + telemetry gates.
2. Ship MVP (`paper2md convert`) by completing US1.
3. Layer asset/equation extraction (US2) to deliver reviewer-ready markdown.
4. Add verification workflow (US3) for automated fidelity audits.
5. Execute Polish tasks for logging, docs, and ADRs.

