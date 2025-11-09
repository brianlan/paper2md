# Tasks: Paper-to-Markdown CLI

**Input**: Design documents from `/specs/001-paper-md-cli/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: TDD-first is non-negotiable. Every story MUST list the tests that will be written (and fail)
before implementation. If a spec explicitly forbids tests, call it out as a risk and get approval.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

> Include automation (lint/test/docs scripts) and rationale/comment updates as first-class tasks so
> reviewers can trace compliance with the constitution.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Single project**: `src/`, `tests/` at repository root
- **Web app**: `backend/src/`, `frontend/src/`
- **Mobile**: `api/src/`, `ios/src/` or `android/src/`
- Paths shown below assume single project - adjust based on plan.md structure

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Initialize Typer-based CLI package skeleton in `src/paper2md/__init__.py` and `src/paper2md/cli.py`
- [ ] T002 Configure Poetry project metadata and editable install workflow in `pyproject.toml`
- [ ] T003 Add reproduction scripts `Makefile` targets (`make lint`, `make test`, `make format`)
- [ ] T004 [P] Create `src/paper2md/config.py` to load env vars (GROBID URL, model paths) with pydantic settings
- [ ] T005 [P] Bootstrap test layout under `tests/unit/` and `tests/integration/` with sample fixtures directory
- [ ] T006 [P] Document run instructions and env checks in `README.md` referencing `quickstart.md`
- [ ] T007 [P] Pin formatter/linter configs (`ruff.toml`, mypy settings) mirroring automation plan
- [ ] T008 [P] Add Git pre-commit or CI workflow placeholder ensuring `make test` runs before merges

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: Tests must exist (and fail) before writing implementation code in this phase

- [ ] T009 [P] Author orchestrator unit tests with mocked adapters in `tests/unit/pipeline/test_orchestrator.py`
- [ ] T010 [P] Author filesystem storage helper tests in `tests/unit/services/test_storage.py`
- [ ] T011 Create baseline integration fixture PDF setup under `tests/data/sample_papers/streampetr/`
- [ ] T012 Author initial golden manifest/evaluation JSON examples under `tests/data/goldens/`
- [ ] T013 Add contract for CLI exit codes + logging conventions in `docs/cli-behavior.md`
- [ ] T054 [P] Harden error handling specifically for GROBID/model outages with simulated fault tests in adapters (must pass before any user story code)

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 2b: Observability & Telemetry (Pre-Story Gate)

**Purpose**: Instrumentation mandated by NFR-001/NFR-002 must exist before any user story begins

- [ ] T057 [P] Run the 20-page SLA fixture (`tests/data/sample_papers/streampetr/20p/`), capture perf metrics in `docs/perf-report.md`, and fail if the average exceeds 1 minute per page (≤20 minutes total)
- [ ] T062 [P] Add `tests/integration/test_perf_timings.py` that records per-page durations, fails if any page exceeds 30 s, and updates `docs/perf-report.md` with the captured SLA data
- [ ] T063 [P] Instrument `src/paper2md/pipeline/orchestrator.py` to log per-page start/end timestamps + metrics and persist them to the manifest/logs; add pytest coverage ensuring timings exist
- [ ] T064 [P] Propagate correlation IDs (`job_id`/`page_id`) through adapters (`grobid.py`, `vlm_extractor.py`, `ocr.py`) and assert structured logs contain them via new tests in `tests/unit/pipeline/test_logging.py`

**Checkpoint**: Telemetry + perf automation green — proceed to User Story phases only after this point

---

## Phase 3: User Story 1 - Researcher converts a paper to markdown (Priority: P1) 🎯 MVP

**Goal**: Provide an end-to-end CLI command that ingests a PDF, leverages GROBID, rasterizes pages, merges OCR text, and outputs markdown mirroring section hierarchy with verified checksums.

**Independent Test**: Run `poetry run paper2md convert` against `/home/rlan/Downloads/streampetr.pdf`; assert exit code 0, markdown file present, manifest records section ordering, and checksum verification passes before artifacts are returned.

### Tests for User Story 1 (MANDATORY - write first) ⚠️

> Write these tests FIRST, ensure they FAIL before implementation, and rely on mocks/stubs for every
> external dependency so the suite runs deterministically.

- [ ] T014 [P] [US1] Add CLI smoke test in `tests/unit/test_cli.py` ensuring argument parsing and env guard rails
- [ ] T015 [P] [US1] Create GROBID client contract tests with mocked `requests` in `tests/unit/pipeline/test_grobid.py`, covering cache hit/miss behavior and failure retries
- [ ] T016 [P] [US1] Add rasterizer unit tests covering 300 DPI output in `tests/unit/pipeline/test_rasterizer.py`
- [ ] T017 [P] [US1] Expand integration test `tests/integration/test_end_to_end.py` for baseline PDF→markdown flow (mock VLM/OCR)
- [ ] T018 [P] [US1] Add manifest checksum generation tests in `tests/unit/models/test_manifest_checksum.py`
- [ ] T019 [P] [US1] Add tamper-detection tests ensuring checksum verification rejects altered markdown in `tests/unit/models/test_manifest_checksum.py`
- [ ] T065 [P] [US1] Add dedicated cache reuse tests ensuring TEI scaffold fetches use `<output>/cache/tei/` with checksum versioning (e.g., `tests/unit/pipeline/test_grobid_cache.py`)
- [ ] T066 [P] [US1] Add reconciler logging tests in `tests/unit/pipeline/test_reconciler_logging.py` to assert OCR vs. TEI corrections write structured logs + manifest entries
- [ ] T070 [P] [US1] Add markdown writer tests in `tests/unit/services/test_markdown_writer.py` asserting asset captions and correction summaries embed into sections

### Implementation for User Story 1

- [ ] T020 [US1] Implement env validation + CLI argument handling in `src/paper2md/cli.py`
- [ ] T021 [US1] Flesh out `src/paper2md/pipeline/grobid.py` to call `http://localhost:8070`, persist TEI XML, manage `<output>/cache/tei/`, and invalidate caches when checksums or GROBID versions change
- [ ] T022 [US1] Implement `src/paper2md/pipeline/rasterizer.py` to stream PDF pages into `output/pages/page-###.png`
- [ ] T023 [US1] Build scaffold-merging module `src/paper2md/pipeline/reconciler.py` to marry TEI headings with placeholder text nodes
- [ ] T024 [US1] Implement orchestrator sequencing in `src/paper2md/pipeline/orchestrator.py` to run stages in order
- [ ] T025 [US1] Generate markdown writer in `src/paper2md/services/markdown_writer.py` embedding section outline
- [ ] T026 [US1] Emit conversion manifest JSON via `src/paper2md/models/manifest.py` with section/order metadata
- [ ] T027 [US1] Add checksum generation + verification hooks in `src/paper2md/models/manifest.py`
- [ ] T028 [US1] Enforce checksum verification before delivering outputs in `src/paper2md/services/storage.py`
- [ ] T029 [US1] Update integration test to assert section structure and checksum verification in `tests/integration/test_end_to_end.py`
- [ ] T067 [US1] Instrument `src/paper2md/pipeline/reconciler.py` and manifest models to record OCR vs. TEI corrections with page/section metadata and surface them in logs/manifests
- [ ] T068 [US1] Update `README.md` and `docs/cli-behavior.md` with rationale for OCR overrides, checksum enforcement, and streaming guarantees (Principle V)
- [ ] T072 [US1] Implement `--purge-cache` handling in `src/paper2md/cli.py` + storage helpers, deleting `<output>/cache/tei` when requested and documenting behavior

### Large-PDF Streaming (MVP Scope)

- [ ] T056 [P] [US1] Optimize rasterization/VLM batching for large PDFs (>100 pages) while preserving memory caps and documenting tradeoffs inline
- [ ] T060 [P] [US1] Build >100-page fixture and streaming integration test in `tests/integration/test_large_pdf.py` to assert page-by-page processing stays within memory targets
- [ ] T061 [P] [US1] Document streaming test results and thresholds in `docs/perf-report.md` and reference mitigation strategies

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - Analyst catalogues visual assets and equations (Priority: P1)

**Goal**: Extract all figures, tables, algorithms, and equations with correct numbering, captions, crops, LaTeX, and multi-page handling, embedding them into the markdown package.

**Independent Test**: Run CLI against fixture PDF containing known counts, multi-page figures, and ground-truth equations; verify manifest counts match, stitched figures exist, equations renderable in markdown, and LaTeX accuracy ≥95%.

### Tests for User Story 2 (MANDATORY - write first) ⚠️

- [ ] T030 [P] [US2] Add VLM extractor contract tests with stub Qwen responses in `tests/unit/pipeline/test_vlm_extractor.py`
- [ ] T031 [P] [US2] Create OCR reconciliation tests in `tests/unit/pipeline/test_ocr.py` for column detection accuracy
- [ ] T032 [P] [US2] Add AssetCatalog validation tests ensuring counts and caption numbering in `tests/unit/models/test_manifest_assets.py`
- [ ] T033 [P] [US2] Add LaTeX fidelity tests comparing extracted equations to goldens in `tests/unit/models/test_equations.py`
- [ ] T034 [P] [US2] Extend integration test to cover asset counts, multi-page figures, and LaTeX accuracy in `tests/integration/test_end_to_end.py`

### Implementation for User Story 2

- [ ] T035 [US2] Implement `src/paper2md/pipeline/vlm_extractor.py` to call `/ssd4/models/Qwen/Qwen3-VL-8B-Instruct-FP8` per page and save crops
- [ ] T036 [US2] Build `src/paper2md/pipeline/ocr.py` to run `/ssd4/models/Qwen/Qwen3-VL-8B-Instruct-FP8` in OCR mode and emit ordered text blocks
- [ ] T037 [US2] Extend `src/paper2md/pipeline/reconciler.py` to merge OCR content with TEI scaffold and insert figure/table placeholders
- [ ] T038 [US2] Implement `src/paper2md/models/manifest.py` AssetCatalog + EquationRef serialization with numbering and LaTeX sources
- [ ] T039 [US2] Update markdown writer to embed figures/tables/algorithms and equations with captions/LaTeX
- [ ] T040 [US2] Implement multi-page figure detection and stitching logic in `src/paper2md/pipeline/vlm_extractor.py`
- [ ] T041 [US2] Enhance manifest warnings/logging when counts mismatch TEI or multi-page merges occur in `src/paper2md/cli.py`
- [ ] T042 [US2] Generate regression fixtures (images + LaTeX) under `tests/data/goldens/assets/`
- [ ] T043 [US2] Update integration tests to compare runtime manifest/asset outputs to goldens for counts and LaTeX accuracy
- [ ] T069 [US2] Document asset/equation extraction heuristics and rationale in `docs/fidelity-review.md` + manifest comments (Principle V)

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - Reviewer audits fidelity (Priority: P2)

**Goal**: Automatically evaluate the generated markdown against the original PDF using Qwen3-VL to produce a structured fidelity report saved with the package.

**Independent Test**: Execute CLI verification on a completed conversion; assert evaluation JSON includes structure/asset/equation scores and discrepancies, and integration tests flag intentionally corrupted outputs.

### Tests for User Story 3 (MANDATORY - write first) ⚠️

- [ ] T044 [P] [US3] Add evaluation service unit tests with mocked VLM comparison outputs in `tests/unit/services/test_evaluation.py`
- [ ] T045 [P] [US3] Create CLI option test ensuring verification runs independently via `tests/unit/test_cli.py`
- [ ] T046 [P] [US3] Extend integration test to corrupt markdown intentionally and assert discrepancies appear in `tests/integration/test_end_to_end.py`
- [ ] T071 [P] [US3] Add docs lint/check in `tests/unit/test_docs_fidelity.py` verifying `docs/fidelity-review.md` explains current evaluation heuristics (Principle V)
- [ ] T073 [P] [US3] Add manifest reader checksum tests in `tests/unit/models/test_manifest_reader.py` to prove packages re-verify before evaluation

### Implementation for User Story 3

- [ ] T047 [US3] Implement evaluation orchestrator in `src/paper2md/services/evaluation.py` to run comparative VLM prompts
- [ ] T048 [US3] Add CLI flag/command (`paper2md verify`) wired to evaluation service in `src/paper2md/cli.py`
- [ ] T049 [US3] Persist evaluation JSON alongside markdown within `src/paper2md/services/storage.py`
- [ ] T050 [US3] Update manifest schema to link evaluation scores + discrepancy summaries in `src/paper2md/models/manifest.py`
- [ ] T051 [US3] Document evaluation workflow and report interpretation in `docs/fidelity-review.md`
- [ ] T052 [US3] Update integration test harness to run verification mode post-conversion automatically
- [ ] T074 [US3] Ensure `paper2md verify` (and supporting storage helpers) re-run manifest checksum verification before evaluation begins

**Checkpoint**: All user stories should now be independently functional

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T053 [P] Add structured logging + progress reporting to `src/paper2md/cli.py` and `pipeline/orchestrator.py`
- [ ] T055 [P] Improve documentation (`README.md`, `quickstart.md`, `docs/cli-behavior.md`) with rationale + troubleshooting (post-story polish)
- [ ] T058 [P] Add telemetry hooks or manifest flags for unresolved OCR vs. TEI conflicts
- [ ] T059 Ensure intent-focused comments/ADRs capture reconciliation and evaluation heuristics (Principle V reinforcement)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **Phase 2b (Telemetry)**: Depends on Foundational completion - BLOCKS all user stories until per-page metrics + correlation logging exist
- **User Stories (Phase 3+)**: All depend on Foundational + Phase 2b completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P1 → P2)
- **Polish (Final Phase)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P1)**: Builds on US1 outputs and requires asset/equation infrastructure; otherwise independent logic
- **User Story 3 (P2)**: Depends on US1 + US2 outputs existing to compare against
- **Large-PDF streaming tasks (T056, T060, T061)**: Treated as part of US1 MVP; must pass before moving to US2

### Within Each User Story

- Tests (if included) MUST be written and FAIL before implementation
- Models before services
- Services before CLI/manifest wiring
- Core implementation before integration
- Story complete before moving to next priority

### Parallel Opportunities

- Adapter/unit test tasks (T009–T010, T014–T019, T030–T034, T044–T046) can run simultaneously once scaffolding exists.
- VLM and OCR modules in US2 can be developed in parallel because they touch different files.
- Evaluation service (US3) can start while documentation/polish efforts run, as long as converted packages exist for testing.

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test User Story 1 independently
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deliver MVP
3. Add User Story 2 → Test independently → Deliver asset-rich version
4. Add User Story 3 → Test independently → Deliver audited version
5. Execute Polish phase for performance + docs

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1
   - Developer B: User Story 2
   - Developer C: User Story 3
3. Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Verify tests fail before implementing
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
