# Data Model: Paper-to-Markdown CLI

## Entity: ConversionJob
- **Description**: Represents a single CLI invocation converting one PDF into markdown.
- **Fields**:
  - `job_id` (UUID, required): Unique identifier emitted in manifests/logs.
  - `pdf_path` (string, required): Absolute path to source PDF; validated for readability.
  - `output_dir` (string, required): Directory where artifacts are written; created if missing.
  - `env_check` (enum {pass, warn, fail}): Result of verifying conda env + GPU availability.
  - `status` (enum {pending, running, succeeded, failed}): Updated throughout pipeline.
  - `started_at` / `completed_at` (ISO-8601 timestamps): For duration tracking.
  - `error_summary` (string, optional): Populated if `status=failed`.
- **Relationships**:
  - One `ConversionJob` links to many `LayoutPage` records (1:N).
  - One `ConversionJob` owns a single `AssetCatalog` snapshot and `MarkdownPackage` bundle.
- **Validation Rules**:
  - `pdf_path` must exist and be a file before starting.
  - `output_dir` must reside on a writable filesystem.

## Entity: LayoutPage
- **Description**: Captures per-page rasterization, OCR output, and detected layout segments.
- **Fields**:
  - `job_id` (UUID, FK → ConversionJob)
  - `page_number` (int, required, 1-indexed)
  - `image_path` (string, required): 300 DPI PNG for the page.
  - `column_layout` (enum {single, double, mixed}): Derived from OCR heuristics.
  - `ocr_text` (rich text payload): Ordered paragraphs including bounding boxes.
  - `grobid_fragments` (list of ids): TEI nodes mapped to this page.
  - `asset_refs` (list<string>): IDs referencing figures/tables/algorithms detected on page.
  - `equations` (list<EquationRef>): Inline or block equations with LaTeX and coordinates.
- **Relationships**:
  - Many `LayoutPage` entries aggregate into one `MarkdownPackage` via reconciler ordering.
- **Validation Rules**:
  - `page_number` must be sequential with no gaps.
  - `column_layout` detection confidence < threshold should raise a warning for review.

## Entity: AssetCatalog
- **Description**: Canonical list of non-textual artifacts.
- **Fields**:
  - `asset_id` (string, e.g., `fig-001`), unique per job.
  - `type` (enum {figure, table, algorithm})
  - `page_number` (int) and `bounding_box` (normalized coordinates)
  - `image_path` (string) – cropped PNG referencing master page image.
  - `caption` (string) – merged from VLM detection and OCR cross-check.
  - `sequence_index` (int) – preserves original numbering order.
- **Relationships**:
  - Linked back to `LayoutPage.asset_refs`.
  - Referenced by `MarkdownPackage` when embedding.
- **Validation Rules**:
  - Counts per `type` must match TEI metadata; mismatches flagged in manifest.
  - Captions must include numbering prefix (e.g., "Figure 2:").

## Entity: EquationRef
- **Description**: Represents LaTeX-renderable equations extracted from OCR/VLM.
- **Fields**:
  - `equation_id` (string, e.g., `eq-03`)
  - `content_latex` (string) – sanitized LaTeX ready for markdown $$ blocks.
  - `display_type` (enum {inline, block})
  - `page_number` (int)
  - `position` (bounding box) – for traceability.
  - `source` (enum {ocr, vlm, grobid}) – indicates origin for reconciliation.
- **Relationships**:
  - Associated with `LayoutPage.equations` and collected into `MarkdownPackage` references.
- **Validation Rules**:
  - Equations flagged as `inline` must have reference anchors mapping to paragraph text.

## Entity: MarkdownPackage
- **Description**: Final deliverable representing the converted paper.
- **Fields**:
  - `markdown_path` (string): Primary `.md` output.
  - `assets_dir` (string): Relative path storing figures/tables/algorithms/equations images.
  - `manifest_path` (string): JSON summary of counts, mapping, warnings.
  - `markdown_checksum` (string): Cryptographic hash (e.g., SHA-256) of the markdown file for integrity verification.
  - `evaluation_report` (string): Path to fidelity review JSON.
  - `section_outline` (ordered list): Derived from TEI, each node containing heading, level,
    source page range, and reconciled text hash for regression tests.
- **Relationships**:
  - Consumes `AssetCatalog`, `LayoutPage`, and `EquationRef` data when rendering markdown.
- **Validation Rules**:
  - `section_outline` must match TEI order exactly.
  - Manifest must include and validate the markdown checksum for downstream integrity checks.
