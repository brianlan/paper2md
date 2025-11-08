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
- [ ] T002 Configure Poetry or pip-tools project metadata plus editable install in `pyproject.toml`
- [ ] T003 Add reproduction scripts `Makefile` targets (`make lint`, `make test`, `make format`)
- [ ] T004 [P] Create `src/paper2md/config.py` to load env vars (GROBID URL, model paths) with pydantic settings
- [ ] T005 [P] Bootstrap test layout under `tests/unit/` and `tests/integration/` with sample fixtures directory
- [ ] T006 [P] Document run instructions and env checks in `README.md` referencing `quickstart.md`
- [ ] T007 [P] Pin formatter/linter configs (`ruff.toml`, `pyproject.toml` for mypy) mirroring automation plan
- [ ] T008 [P] Add Git pre-commit or CI workflow placeholder ensuring `make test` runs before merges

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T009 Implement `src/paper2md/pipeline/orchestrator.py` skeleton with hooks for each pipeline stage
- [ ] T010 Create GROBID client adapter in `src/paper2md/pipeline/grobid.py` with retry/backoff utilities
- [ ] T011 Implement rasterization helper in `src/paper2md/pipeline/rasterizer.py` using pdf2image stream mode
- [ ] T012 Define shared manifest/evaluation models in `src/paper2md/models/manifest.py` and `evaluation.py`
- [ ] T013 [P] Implement filesystem storage helpers in `src/paper2md/services/storage.py`
- [ ] T014 Wire Typer command to orchestrator entrypoint with argument validation in `src/paper2md/cli.py`
- [ ] T015 Create baseline integration fixture PDF setup under `tests/data/sample_papers/streampetr/`
- [ ] T016 Author initial golden manifest/evaluation JSON examples under `tests/data/goldens/`
- [ ] T017 [P] Write orchestrator unit tests with mocks for each adapter in `tests/unit/pipeline/test_orchestrator.py`
- [ ] T018 [P] Implement `tests/unit/services/test_storage.py` for filesystem helper coverage
- [ ] T019 [P] Add contract for CLI exit codes + logging conventions in `docs/cli-behavior.md`

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Researcher converts a paper to markdown (Priority: P1) 🎯 MVP

**Goal**: Provide an end-to-end CLI command that ingests a PDF, leverages GROBID, rasterizes pages, merges OCR text, and outputs markdown mirroring section hierarchy.

**Independent Test**: Run `paper2md convert` against `/home/rlan/Downloads/streampetr.pdf` using fixtures; assert exit code 0, markdown file present, and manifest records section ordering identical to GROBID outline.

### Tests for User Story 1 (MANDATORY - write first) ⚠️

> Write these tests FIRST, ensure they FAIL before implementation, and rely on mocks/stubs for every
> external dependency so the suite runs deterministically.

- [ ] T020 [P] [US1] Add CLI smoke test in `tests/unit/test_cli.py` ensuring argument parsing and env guard rails
- [ ] T021 [P] [US1] Create GROBID client contract tests with mocked `requests` in `tests/unit/pipeline/test_grobid.py`
- [ ] T022 [P] [US1] Add rasterizer unit tests covering 300 DPI output in `tests/unit/pipeline/test_rasterizer.py`
- [ ] T023 [P] [US1] Expand integration test `tests/integration/test_end_to_end.py` for basic PDF→markdown flow (mock VLM/OCR)

### Implementation for User Story 1

- [ ] T024 [US1] Implement env validation + CLI argument handling in `src/paper2md/cli.py`
- [ ] T025 [US1] Flesh out `src/paper2md/pipeline/grobid.py` to call `http://localhost:8070` and persist TEI XML
- [ ] T026 [US1] Implement `src/paper2md/pipeline/rasterizer.py` to stream PDF pages into `output/pages/page-###.png`
- [ ] T027 [US1] Build scaffold-merging module `src/paper2md/pipeline/reconciler.py` to marry TEI headings with placeholder text nodes
- [ ] T028 [US1] Implement orchestrator happy-path pipeline sequencing in `src/paper2md/pipeline/orchestrator.py`
- [ ] T029 [US1] Generate markdown writer in `src/paper2md/services/markdown_writer.py` embedding section outline
- [ ] T030 [US1] Emit conversion manifest JSON via `src/paper2md/models/manifest.py` with section/order metadata
- [ ] T031 [US1] Update integration test to assert manifest + markdown structure and run via `make test`

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - Analyst catalogues visual assets and equations (Priority: P1)

**Goal**: Extract all figures, tables, algorithms, and equations with correct numbering, captions, crops, and LaTeX, embedding them into the markdown package.

**Independent Test**: Run CLI against fixture PDF containing known counts; verify manifest counts match TEI, assets saved in `assets/`, equations renderable in markdown, and automated test asserts zero missing items.

### Tests for User Story 2 (MANDATORY - write first) ⚠️

- [ ] T032 [P] [US2] Add VLM extractor contract tests with stub Qwen responses in `tests/unit/pipeline/test_vlm_extractor.py`
- [ ] T033 [P] [US2] Create OCR reconciliation tests in `tests/unit/pipeline/test_ocr.py` for column detection accuracy
- [ ] T034 [P] [US2] Add AssetCatalog validation tests ensuring counts and caption numbering in `tests/unit/models/test_manifest_assets.py`
- [ ] T035 [P] [US2] Extend integration test to include real asset/equation fixtures verifying manifest counts

### Implementation for User Story 2

- [ ] T036 [US2] Implement `src/paper2md/pipeline/vlm_extractor.py` to call `/ssd4/models/Qwen/Qwen3-VL-8B-Instruct-FP8` per page and save crops
- [ ] T037 [US2] Build `src/paper2md/pipeline/ocr.py` to run `/ssd4/models/datalab-to/chandra` and emit ordered text blocks
- [ ] T038 [US2] Extend `src/paper2md/pipeline/reconciler.py` to merge OCR content with TEI scaffold and insert figure/table placeholders
- [ ] T039 [US2] Implement `src/paper2md/models/manifest.py` AssetCatalog + EquationRef serialization with numbering
- [ ] T040 [US2] Update markdown writer to embed figures/tables/algorithms and equations with captions/LaTeX
- [ ] T041 [US2] Add warnings section to manifest when counts mismatch TEI and wire to CLI logging
- [ ] T042 [US2] Create regression fixture assets (images + LaTeX) under `tests/data/goldens/assets/`
- [ ] T043 [US2] Update integration test to compare golden manifest to runtime output for asset/equation coverage

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - Reviewer audits fidelity (Priority: P2)

**Goal**: Automatically evaluate the generated markdown against the original PDF using Qwen3-VL to produce a structured fidelity report saved with the package.

**Independent Test**: Execute CLI verification step on completed conversion; assert evaluation JSON includes structure/asset/equation scores and discrepancies, and integration tests flag intentionally corrupted outputs.

### Tests for User Story 3 (MANDATORY - write first) ⚠️

- [ ] T044 [P] [US3] Add evaluation service unit tests with mocked VLM comparison outputs in `tests/unit/services/test_evaluation.py`
- [ ] T045 [P] [US3] Create CLI option test ensuring verification runs independently via `tests/unit/test_cli.py`
- [ ] T046 [P] [US3] Extend integration test to corrupt markdown intentionally and assert discrepancies appear in report

### Implementation for User Story 3

- [ ] T047 [US3] Implement evaluation orchestrator in `src/paper2md/services/evaluation.py` to run comparative VLM prompts
- [ ] T048 [US3] Add CLI flag/command (`paper2md verify`) wired to evaluation service in `src/paper2md/cli.py`
- [ ] T049 [US3] Persist evaluation JSON alongside markdown within `src/paper2md/services/storage.py`
- [ ] T050 [US3] Update manifest schema to link evaluation scores + discrepancy summaries
- [ ] T051 [US3] Document evaluation workflow and report interpretation in `docs/fidelity-review.md`
- [ ] T052 [US3] Update integration test harness to run verification mode post-conversion automatically

**Checkpoint**: All user stories should now be independently functional

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T053 [P] Add structured logging + progress reporting to `src/paper2md/cli.py` and `pipeline/orchestrator.py`
- [ ] T054 [P] Harden error handling for external services (retry budgets, fallback messages) in adapters
- [ ] T055 [P] Improve documentation (`README.md`, `quickstart.md`, `docs/cli-behavior.md`) with rationale + troubleshooting
- [ ] T056 [P] Optimize rasterization/VLM batching for large PDFs (>100 pages) while preserving memory caps
- [ ] T057 [P] Conduct performance run on 20-page paper, record metrics in `docs/perf-report.md`
- [ ] T058 [P] Add telemetry hooks or manifest flags for unresolved OCR vs. TEI conflicts
- [ ] T059 Ensure intent-focused comments/ADRs exist for reconciliation tradeoffs and evaluation heuristics

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P1 → P2)
- **Polish (Final Phase)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P1)**: Requires asset/equation infrastructure from Phase 2; otherwise independent of US1 logic
- **User Story 3 (P2)**: Depends on US1 + US2 outputs existing to compare against

### Within Each User Story

- Tests (if included) MUST be written and FAIL before implementation
- Models before services
- Services before CLI/manifest wiring
- Core implementation before integration
- Story complete before moving to next priority

### Parallel Opportunities

- Tests for each adapter (GROBID, rasterizer, VLM, OCR) can run simultaneously once scaffolding exists.
- Asset extraction (VLM) and OCR modules in US2 can be developed in parallel because they touch different services/files.
- Evaluation service (US3) can be developed while documentation/polish tasks run, as long as converted packages exist for testing.

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
