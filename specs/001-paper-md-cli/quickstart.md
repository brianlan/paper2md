# Quickstart: Paper-to-Markdown CLI

## 1. Environment
1. Ensure Docker container for GROBID is running locally and exposes `http://localhost:8070`.
2. Activate the required conda env:
   ```bash
   conda activate /ssd4/envs/llm_py310_torch271_cu128
   ```
3. Install project dependencies with Poetry (once the CLI package exists):
   ```bash
   poetry install
   ```

## 2. Run a Conversion
```bash
poetry run paper2md convert \
  --pdf /home/rlan/Downloads/streampetr.pdf \
  --output ./output/streampetr \
  --overwrite
```
This command:
- Validates the conda env + GPU access
- Calls GROBID for TEI XML
- Rasterizes each page at 300 DPI
- Extracts figures/tables/algorithms/equations via Qwen3-VL
- Uses Qwen3-VL OCR to capture text ordering and reconcile with TEI
- Writes `paper.md`, asset crops, manifest JSON, telemetry data, and TEI cache artifacts

## 3. Verify the Package
```bash
poetry run paper2md verify \
  --package ./output/streampetr
```
This command checks the manifest + markdown checksum, replays the fidelity evaluation via Qwen3-VL,
and stores `evaluation.json` (structure/assets/equations scores + discrepancies) next to the manifest.

## 4. Review Outputs
- `paper.md`: Primary markdown file with embedded asset references
- `assets/`: Folder with figures/tables/algorithms cropped at source resolution
- `manifest.json`: Counts, warnings, and mappings for traceability
- `telemetry.json`: Per-page timings used to prove the ≤1 minute/page average
- `evaluation.json`: Fidelity report produced after running `paper2md verify`

## 5. Automated Tests
From repo root:
```bash
make test
```
Runs lint (ruff), typing (mypy), and pytest suites (unit + integration). Fixture PDFs live under
`tests/data/sample_papers/` so tests remain deterministic without live services.
