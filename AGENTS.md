# paper2md Development Guidelines

Auto-generated from all feature plans. Last updated: 2025-11-08

## Active Technologies
- Python 3.10 inside `/ssd4/envs/llm_py310_torch271_cu128` + Typer (CLI), pydantic (manifest schemas), requests/httpx (GROBID), pdf2image + Poppler (rasterization), Pillow/numpy/OpenCV (image ops), PyTorch (Qwen3-VL wrappers) (001-paper-md-cli)
- Local filesystem (output bundles under user-selected directories, plus `<output>/cache/tei` for TEI reuse) (001-paper-md-cli)

- Python 3.10 (per `/ssd4/envs/llm_py310_torch271_cu128`) + Typer (CLI), pydantic (manifests), requests/httpx (GROBID), pdf2image/poppler, Pillow, numpy, OpenCV, PyTorch for VLM/OCR wrappers (001-paper-md-cli)

## Project Structure

```text
src/
tests/
```

## Commands

cd src [ONLY COMMANDS FOR ACTIVE TECHNOLOGIES][ONLY COMMANDS FOR ACTIVE TECHNOLOGIES] pytest [ONLY COMMANDS FOR ACTIVE TECHNOLOGIES][ONLY COMMANDS FOR ACTIVE TECHNOLOGIES] ruff check .

## Code Style

Python 3.10 (per `/ssd4/envs/llm_py310_torch271_cu128`): Follow standard conventions

## Recent Changes
- 001-paper-md-cli: Added Python 3.10 inside `/ssd4/envs/llm_py310_torch271_cu128` + Typer (CLI), pydantic (manifest schemas), requests/httpx (GROBID), pdf2image + Poppler (rasterization), Pillow/numpy/OpenCV (image ops), PyTorch (Qwen3-VL wrappers)

- 001-paper-md-cli: Added Python 3.10 (per `/ssd4/envs/llm_py310_torch271_cu128`) + Typer (CLI), pydantic (manifests), requests/httpx (GROBID), pdf2image/poppler, Pillow, numpy, OpenCV, PyTorch for VLM/OCR wrappers

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
