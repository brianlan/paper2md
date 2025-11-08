# Research: Paper-to-Markdown CLI

## 1. Execution Environment & Language
- **Decision**: Implement the CLI in Python 3.10 within the mandated conda env `/ssd4/envs/llm_py310_torch271_cu128` using Typer for ergonomics.
- **Rationale**: Python already powers the required ML bindings (PyTorch) and pdf toolchains, the conda env is pre-provisioned, and Typer keeps commands readable per constitution.
- **Alternatives Considered**:
  - *Click*: Mature but similar surface area; Typer auto-generates help/docs with type hints.
  - *Go/Rust binary*: Faster startup but would complicate integrating Python-only model deps.

## 2. GROBID Interaction Strategy
- **Decision**: Communicate with the local Dockerized GROBID service over HTTP (`http://localhost:8070`) via `requests` with retry/backoff wrappers and TEI XML caching per job.
- **Rationale**: HTTP keeps the coupling minimal, caching avoids reprocessing large PDFs, and retries mitigate transient Docker hiccups without masking hard failures.
- **Alternatives Considered**:
  - *Direct filesystem watching of GROBID output*: Requires changing the container config and increases complexity.
  - *Batching multiple PDFs per call*: Not needed now and risks mixing manifests in memory.

## 3. Page Rasterization & Layout Prep
- **Decision**: Use `pdf2image` (Poppler backend) to convert each page to 300 DPI PNG files streamed page-by-page.
- **Rationale**: pdf2image is reliable for academic PDFs, produces Pillow images consumable by both VLM and OCR steps, and streaming respects memory constraints for large papers.
- **Alternatives Considered**:
  - *Ghostscript CLI*: Adds shell dependency variance and complicates testing.
  - *PyMuPDF*: Powerful but heavier dependency; pdf2image already satisfies resolution needs.

## 4. Visual Asset Extraction
- **Decision**: Wrap the local model `/ssd4/models/Qwen/Qwen3-VL-8B-Instruct-FP8` with a service that requests per-page detections (figures, tables, algorithms) and exports crops + captions in a normalized JSON manifest.
- **Rationale**: Page-scoped passes keep GPU memory predictable, allow targeted retries, and make it easy to reconcile asset counts with GROBID metadata.
- **Alternatives Considered**:
  - *Full-document detection*: Harder to parallelize and increases risk of missing multi-page figures.
  - *Multiple specialized models*: Would require additional deployments and violate "local only" constraint.

## 5. Text Extraction & Reconciliation
- **Decision**: Run `/ssd4/models/datalab-to/chandra` OCR on each rasterized page, detect column breaks, and merge with the TEI scaffold by matching headings + paragraph hashes, favoring OCR text when conflicts arise but logging overrides.
- **Rationale**: OCR is more faithful for glyph-level accuracy, while TEI provides structure; merging both preserves headings/subsections while fixing typos, satisfying the "readable over clever" principle.
- **Alternatives Considered**:
  - *Use GROBID text only*: Faster but risks mangled equations/ligatures.
  - *Use OCR only*: Would lose clean section hierarchy and numbering.

## 6. Fidelity Evaluation Workflow
- **Decision**: After markdown generation, re-run Qwen3-VL in comparison mode that accepts both the PDF images and rendered markdown snapshots, producing a JSON scorecard (structure, assets, equations) stored alongside the package.
- **Rationale**: Re-using the same VLM ensures consistent perception between extraction and evaluation, and a structured scorecard gives reviewers actionable metrics without mutating the output.
- **Alternatives Considered**:
  - *Manual checklist*: Too slow and subjective.
  - *Text-only diffing*: Cannot validate visual fidelity or layout alignment.

## 7. Testing & Automation Strategy
- **Decision**: Adopt pytest with golden fixture PDFs plus mocked adapters for external services; enforce via `make test` that runs ruff + mypy + pytest before any implementation merges.
- **Rationale**: Aligns with constitution TDD mandates, keeps automation deterministic, and is easy to mirror in CI.
- **Alternatives Considered**:
  - *Ad-hoc scripts*: Harder to maintain and review.
  - *Heavy integration tests only*: Slow feedback and brittle when services are offline.
