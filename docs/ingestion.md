# Ingestion & file formats

Everything you put into a knowledge base — a manual upload, a
[`POST .../upload`](api-reference.md#rag-pipeline-httplocalhost8004) call, or a
[connector](#connectors-crawl-a-source) — flows through one ingestion path:
**extract text → chunk → embed → store**. This page covers what that path
accepts, how a file's type is decided, and the limits worth knowing before you
upload.

## How a file's type is decided

Minder picks an extractor from a file's **actual contents, not its name or
extension**. Each supported format has a detector that either checks the file's
magic bytes or genuinely tries to parse it — so a `report.pdf` that's really a
PNG, or a `.txt` that's actually a CSV export, is handled as what it *is*, not
what it's labelled.

A practical consequence: renaming a file never changes how it's ingested, and a
file that matches **no** registered format is **rejected cleanly** (an HTTP
`415`) rather than being silently decoded into garbage and embedded. If an
upload is rejected, the file genuinely isn't one of the types below.

!!! tip "Check an extraction, don't guess"
    Every uploaded document can be expanded to inspect its **stored chunks**
    (in the console, or via
    [`GET .../documents/{id}/chunks`](api-reference.md#rag-pipeline-httplocalhost8004)).
    That's the fastest way to tell a bad extraction or OCR result apart from a
    retrieval problem.

## Supported formats

| Category | Formats | Notes |
|---|---|---|
| **Text & Markdown** | `.txt`, `.md` | Plain prose; a leading UTF-8 BOM is stripped. |
| **PDF** | `.pdf` | Digital PDFs; **tables are preserved** as Markdown (see [PDF tables](#pdf-tables)). |
| **Word** | `.docx`, `.doc` | Modern OOXML **and** legacy pre-2007 binary Word. |
| **PowerPoint** | `.pptx` | Slide text, in slide order. |
| **Excel** | `.xlsx` | Each sheet extracted as its own table block. |
| **EPUB** | `.epub` | E-book chapters as prose. |
| **HTML** | `.html`, `.htm` | Readable article text only — nav/header/footer boilerplate is stripped. |
| **CSV / TSV** | `.csv`, `.tsv` | Row/column-aware; the header row is repeated in every chunk. |
| **JSON / JSONL** | `.json`, `.jsonl` | Record- and key-path-aware chunking. |
| **XML** | `.xml` | Element/attribute paths preserved; parsed with hardened, entity-safe XML. |
| **Config** | `.yaml`/`.yml`, `.toml`, `.ini`/`.cfg` | Flattened to dotted key paths. |
| **Columnar / row data** | `.parquet`, `.avro`, `.sql` | Tabular data and SQL dumps. |
| **Images (OCR)** | `.png`, `.jpg`/`.jpeg`, `.gif`, `.bmp`, `.tif`/`.tiff`, `.webp` | Scans, screenshots, photos of prose — see [Image OCR](#image-ocr). |
| **Audio & video** | `.mp3`, `.wav`, `.m4a`, `.mp4`, `.m4v`, `.mov`, `.ogg`/`.oga`, `.opus`, `.flac`, `.aac`, `.webm`, `.mkv`, `.avi` | Transcribed to text — see [Audio & video](#audio-video). |

Structured formats (CSV, JSON, XML, spreadsheets, config, …) are chunked with
their **structure in mind** rather than split blindly by character count, so a
retrieved chunk still carries its column names, key paths, or sheet context.

### PDF tables

Tables inside a **digital** (text-based) PDF are extracted with their row and
column structure intact and rendered as Markdown tables, instead of being
flattened into reading order — so a value stays pinned to its row and column in
the stored chunk. Ordinary prose PDFs are unaffected.

!!! note "Scope"
    This covers digital PDFs only. A **scanned** or photographed PDF page has no
    text layer to read; it falls back to plain text extraction and, if there's
    no recoverable text at all, yields nothing. Extracting tables from *images*
    (scanned pages), and complex merged-cell or multi-page-spanning tables, are
    future work.

### Image OCR

Scanned documents, screenshots, and photos of text are run through OCR
(Tesseract) to recover their prose, which is then ingested like any other text.
An image with no legible text produces no chunks and the upload is rejected as
empty.

!!! note "Scope"
    OCR here recovers **prose only** — it does not reconstruct tables or layout
    from an image, and runs with the default English model. Image preprocessing,
    additional language packs, and surfacing a low-confidence signal are future
    work.

### Audio & video {#audio-video}

Uploaded recordings are transcribed by Minder's built-in speech-to-text engine
(the same one behind the [Voice](using-minder.md#voice-platformvoice) page) and
the transcript is ingested as prose. Video containers work too: their **audio
track** is transcribed — the visual content (frames, slides) is not.

!!! note "Scope & limits"
    - Only the audio track is used — no speaker **diarization** or timestamps,
      and no OCR of video frames/slides. These are future work.
    - Transcription needs the speech-to-text service to be running (the
      **voice** capability).
    - A single decode is size-capped (default 25 MB); a long recording is
      processed as a background job (see below), so it never blocks your upload.

## Limits & processing

- **Ingestion is a background job.** An upload returns immediately as
  `processing`; the console polls until it flips to `completed` (with chunk and
  vector counts) or `failed`. This is why a long recording or a large document
  doesn't hold the request open.
- **Upload size.** Documents are capped at `MAX_UPLOAD_SIZE_MB` (default 50 MB),
  with a higher gateway ceiling (`MAX_PROXY_BODY_SIZE_MB`, default 150 MB).
  Audio/video has its own 25 MB per-decode cap on the speech-to-text side.
- **Embeddings must be reachable.** If the embedding backend is unreachable the
  upload returns `503` and the document is **not** indexed — there's no silent
  zero-vector left behind. See [AI setup](ai-setup.md).

## Connectors: crawl a source

Beyond manual uploads, a **connector** pulls content from an external source and
ingests it through the same pipeline. Connectors are ordinary
[plugins](plugins/index.md) (category `connector`) — no bespoke ingestion code,
and the same safety rules as every other plugin.

The first one is **Web Crawl**
([`webcrawl`](https://github.com/minderhq/plugins/tree/main/webcrawl)): give it a
list of public seed URLs and a target knowledge base, and it fetches each page,
extracts the readable text, and uploads it — optionally following same-domain
links to a shallow depth. It's bounded (caps on pages, depth, per-page size, and
a politeness delay) and **SSRF-guarded**: every URL is resolved and rejected if
it points at a private, loopback, link-local, or cloud-metadata address, and
redirects are re-checked on every hop.

Two ways it runs, matching Minder's read/write split:

- A side-effect-free **preview** (crawl + extract, report per-page titles and
  sizes) that never writes to a knowledge base.
- An explicit **ingest** action (JWT-gated) that actually uploads the crawled
  pages into the target knowledge base.

!!! note "Not yet available"
    Connectors for OAuth-gated sources (Notion, Confluence, Google Drive, Slack)
    are the intended next step but are **not** shipped — each needs its own
    credential story. Today the public-web crawler is the one connector.
