# pdf-inspector

Fast Rust library for PDF classification and text extraction. Detects whether a PDF is text-based or scanned, extracts text with position awareness, and converts to clean Markdown — all without OCR.

Built by [Firecrawl](https://firecrawl.dev) to handle text-based PDFs locally in under 200ms, skipping expensive OCR services for the ~54% of PDFs that don't need them.

## Features

- **Smart classification** — Detect TextBased, Scanned, ImageBased, or Mixed PDFs in ~10-50ms by sampling content streams. Returns a confidence score (0.0-1.0) and per-page OCR routing.
- **Text extraction** — Position-aware extraction with font info, X/Y coordinates, and automatic multi-column reading order.
- **Markdown conversion** — Headings (H1-H4 via font size ratios), bullet/numbered/letter lists, code blocks (monospace font detection), tables, subscript/superscript, URL linking, and page breaks.
- **CID font support** — Proper ToUnicode CMap decoding for Type0/Identity-H fonts, UTF-16BE, UTF-8, and Latin-1 encodings.
- **Lightweight** — Pure Rust, no ML models, no external services. Single dependency on `lopdf` for PDF parsing.

## Quick start

### As a library

Add to your `Cargo.toml`:

```toml
[dependencies]
pdf-inspector = { git = "https://github.com/firecrawl/pdf-inspector" }
```

Detect and extract in one call:

```rust
use pdf_inspector::process_pdf;

let result = process_pdf("document.pdf")?;

println!("Type: {:?}", result.pdf_type);       // TextBased, Scanned, ImageBased, Mixed
println!("Confidence: {:.0}%", result.confidence * 100.0);
println!("Pages: {}", result.page_count);

if let Some(markdown) = &result.markdown {
    println!("{}", markdown);
}
```

Or detect without extracting:

```rust
use pdf_inspector::detect_pdf_type;

let detection = detect_pdf_type("document.pdf")?;

match detection.pdf_type {
    pdf_inspector::PdfType::TextBased => {
        // Extract locally — fast and free
    }
    _ => {
        // Route to OCR service
        // detection.pages_needing_ocr tells you exactly which pages
    }
}
```

Customize the detection scan strategy:

```rust
use pdf_inspector::{process_pdf_with_config, DetectionConfig, ScanStrategy};

// Scan all pages for accurate Mixed vs Scanned classification
let config = DetectionConfig {
    strategy: ScanStrategy::Full,
    ..Default::default()
};
let result = process_pdf_with_config("document.pdf", config)?;

// Sample 5 evenly distributed pages (fast for large PDFs)
let config = DetectionConfig {
    strategy: ScanStrategy::Sample(5),
    ..Default::default()
};
let result = process_pdf_with_config("large.pdf", config)?;

// Only check specific pages
let config = DetectionConfig {
    strategy: ScanStrategy::Pages(vec![1, 5, 10]),
    ..Default::default()
};
let result = process_pdf_with_config("known-layout.pdf", config)?;
```

Process from a byte buffer (no filesystem needed):

```rust
use pdf_inspector::process_pdf_mem;

let bytes = std::fs::read("document.pdf")?;
let result = process_pdf_mem(&bytes)?;
```

### CLI

```bash
# Convert PDF to Markdown
cargo run --bin pdf2md -- document.pdf

# JSON output (for piping)
cargo run --bin pdf2md -- document.pdf --json

# Detection only (no extraction)
cargo run --bin detect-pdf -- document.pdf
cargo run --bin detect-pdf -- document.pdf --json
```

## How classification works

1. Parse the xref table and page tree (no full object load)
2. Select pages based on `ScanStrategy` (default: all pages with early exit)
3. Look for `Tj`/`TJ` (text operators) and `Do` (image operators) in content streams
4. Classify based on text operator presence across sampled pages

This detects 300+ page PDFs in milliseconds. The result includes `pages_needing_ocr` — a list of specific page numbers that lack text, enabling per-page OCR routing instead of all-or-nothing.

### Scan strategies

| Strategy | Behavior | Best for |
|---|---|---|
| `EarlyExit` (default) | Scan all pages, stop on first non-text page | Pipelines routing TextBased PDFs to fast extraction |
| `Full` | Scan all pages, no early exit | Accurate Mixed vs Scanned classification |
| `Sample(n)` | Sample `n` evenly distributed pages (first, last, middle) | Very large PDFs where speed matters more than precision |
| `Pages(vec)` | Only scan specific 1-indexed page numbers | When the caller knows which pages to check |

## API

### Functions

| Function | Description |
|---|---|
| `process_pdf(path)` | Detect, extract, and convert to Markdown |
| `process_pdf_with_config(path, config)` | Same, with custom `DetectionConfig` |
| `process_pdf_mem(bytes)` | Same, from a byte buffer |
| `process_pdf_mem_with_config(bytes, config)` | Same, from bytes with custom config |
| `detect_pdf_type(path)` | Classification only (fastest) |
| `detect_pdf_type_with_config(path, config)` | Classification with custom config |
| `detect_pdf_type_mem(bytes)` | Classification from bytes |
| `detect_pdf_type_mem_with_config(bytes, config)` | Classification from bytes with custom config |
| `extract_text(path)` | Plain text extraction |
| `extract_text_with_positions(path)` | Text with X/Y coordinates and font info |
| `to_markdown(path, options)` | Convert directly to Markdown |
| `to_markdown_from_items(items, options)` | Markdown from pre-extracted `TextItem`s |

### Types

| Type | Description |
|---|---|
| `PdfType` | `TextBased`, `Scanned`, `ImageBased`, `Mixed` |
| `PdfProcessResult` | Full result: markdown, metadata, confidence, timing |
| `PdfTypeResult` | Detection result: type, confidence, page count, pages needing OCR |
| `DetectionConfig` | Configuration for detection: scan strategy, thresholds |
| `ScanStrategy` | `EarlyExit`, `Full`, `Sample(n)`, `Pages(vec)` |
| `TextItem` | Text with position, font info, and page number |
| `MarkdownOptions` | Configuration for Markdown conversion |
| `PdfError` | `Io`, `Parse`, `Encrypted`, `InvalidStructure`, `NotAPdf` |

## Markdown output

The converter handles:

| Element | How it's detected |
|---|---|
| Headings (H1-H4) | Font size ratios relative to body text |
| Bullet lists | `*`, `-`, `*`, `○`, `●`, `◦` prefixes |
| Numbered lists | `1.`, `1)`, `(1)` patterns |
| Letter lists | `a.`, `a)`, `(a)` patterns |
| Code blocks | Monospace fonts (Courier, Consolas, Monaco, Menlo, Fira Code, JetBrains Mono) and keyword detection |
| Tables | Position clustering for column/row boundaries |
| Footnotes | Superscript numbers with corresponding text |
| Sub/superscript | Font size and Y-offset relative to baseline |
| URLs | Converted to Markdown links |
| Hyphenation | Rejoins words broken across lines |
| Page numbers | Filtered from output |
| Drop caps | Large initial letters merged with following text |

## Use case: smart PDF routing

pdf-inspector was built for pipelines that process PDFs at scale. Instead of sending every PDF through OCR:

```
PDF arrives
  → pdf-inspector classifies it (~20ms)
  → TextBased + high confidence?
      YES → extract locally (~150ms), done
      NO  → send to OCR service (2-10s)
```

This saves cost and latency for the majority of PDFs that are already text-based (reports, papers, invoices, legal docs).

## License

MIT
