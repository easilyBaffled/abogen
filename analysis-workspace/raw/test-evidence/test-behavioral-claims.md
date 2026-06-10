# Test Suite Behavioral Claims

Source: `/Users/Daniel.Michaelis/abogen/tests/` (39 test files, ~5,016 LOC, pytest)

---

## 1. Test Infrastructure (conftest.py)

### Fixtures and Setup
- **Session-scoped settings isolation**: All tests run with `ABOGEN_SETTINGS_DIR` set to a temp directory (autouse, session scope). Prevents tests from reading/writing user's real settings.
- **Module pre-import guard**: Before tests run, `ebooklib`, `bs4`, and `numpy` are pre-imported if available, preventing test stubs from replacing real packages.

### Pattern: Dependency Stubs
Multiple test files install lightweight module stubs for `soundfile`, `static_ffmpeg`, `ebooklib`, `fitz`, `markdown`, `bs4` when not already imported. Allows unit-testing conversion logic without heavy optional dependencies.

---

## 2. Book Parsing (test_book_parser.py)

### Behavioral Claims
- **Factory pattern**: `get_book_parser()` returns `PdfParser` for `.pdf`, `EpubParser` for `.epub`, `MarkdownParser` for `.md`.
- **Explicit type override**: Passing `file_type="epub"` overrides file extension detection.
- **PDF content extraction**: PdfParser extracts text per page (`page_1`, `page_2` keys). Footnote markers like `[12]` are cleaned.
- **Markdown splitting**: MarkdownParser splits on `# ` headings, generating slugified IDs (`chapter-1`, `chapter-2`).
- **EPUB content extraction**: EpubParser preserves file-based keys (`intro.xhtml`, `chap1.xhtml`).
- **EPUB metadata**: Metadata extraction yields `title` and `author` fields.
- **Ordered list handling**: `<ol>` elements converted to `1) Item One`, `2) Item Two` text format.
- **Position finding**: `_find_position_robust` locates HTML IDs within document content; returns 0 for missing IDs.
- **Chapter listing**: `get_chapters()` returns list of `(id, display_title)` tuples.
- **Formatted text output**: `get_formatted_text()` inserts `<<CHAPTER_MARKER:Chapter 1>>` delimiters.
- **file_type property**: Returns `"pdf"`, `"epub"`, or `"markdown"` respectively.

---

## 3. Text Extraction (test_text_extractor.py)

### Behavioral Claims
- **Character count consistency**: `result.total_characters == calculate_text_length(result.combined_text) == sum(chapter.characters)`. All three measures agree.
- **Composer/artist alignment**: EPUB metadata `composer` equals `artist` and is never "Narrator".
- **Series metadata from OPF**: Calibre-style `<meta name="calibre:series">` and `calibre:series_index` are extracted into `result.metadata["series"]` and `result.metadata["series_index"]`.

---

## 4. EPUB Content Slicing (test_epub_content_slicing.py)

### Behavioral Claims
- **Single-file multi-chapter splitting**: When a single XHTML file contains multiple sections with anchored IDs, the parser correctly slices content between anchors.
- **Ordered list renumbering**: `<ol start="3">` correctly produces `3) Item C`, `4) Item D` in the sliced section.

---

## 5. EPUB3 Export (test_epub_exporter.py)

### Behavioral Claims
- **Package structure**: `build_epub3_package()` produces valid EPUB3 ZIP: `mimetype`, `META-INF/container.xml`, `OEBPS/content.opf`, `OEBPS/nav.xhtml`, audio, XHTML chapters, SMIL overlays.
- **SMIL timing**: SMIL documents include `clipBegin="00:00:00.000"` markers. OPF includes `media-overlay` and `media:duration`.
- **Speaker mode metadata**: OPF includes `abogen:speakerMode` metadata.
- **Missing markers graceful handling**: When `chapter_markers=[]` and `chunk_markers=[]`, export still produces valid output.
- **Whitespace preservation**: Double spaces preserved. Original text stored in `<pre class="chapter-original">` block.
- **Sentence-level chunks as paragraphs**: When chunks have `level="sentence"`, they render as `<p class="chunk-group">` elements.

---

## 6. EPUB Navigation Parsing

### 6a. Heuristic Nav Discovery
- **Content-scanning fallback**: When no `ITEM_NAVIGATION` type item exists, parser scans ITEM_DOCUMENT files for `<nav epub:type="toc">` content.

### 6b. HTML Nav Parsing
- **Flat list parsing**: Standard `<nav epub:type="toc"><ol><li><a href="...">` structures parsed correctly.
- **Nested list parsing**: Nested `<ol>` within `<li>` produces both parent and child chapters.
- **Span grouping headers**: `<li><span>Part I</span><ol>...</ol></li>` pattern: span becomes non-content node (`has_content=False`).

### 6c. Missing File Error Handling
- **Graceful recovery**: Missing files in EPUB manifest don't crash parser. Valid chapters still extracted.

### 6d. NCX Parsing
- **NCX-only parsing**: EPUBs with only NCX (no HTML nav) parsed correctly from `<navPoint>` elements.
- **Nested NCX**: Nested `<navPoint>` structures flattened into list.

### 6e. Standard Nav
- **ITEM_NAVIGATION discovery**: Items with type `ITEM_NAVIGATION` (type code 4) identified as nav documents.
- **Nav property discovery**: ITEM_DOCUMENT items with `properties=["nav"]` also identified.

---

## 7. Conversion Pipeline

### 7a. Chapter Titles
- **Title prefix formatting**: `_format_spoken_chapter_title("1: A Tale", 1, True)` produces `"Chapter 1. A Tale"`.
- **Existing prefix preserved**: Title starting with "Chapter" gets no additional prefix.
- **Empty title handling**: Empty string gets `"Chapter N"` format.
- **Delimiter trimming**: `"7 - Into the Wild"` becomes `"Chapter 7. Into the Wild"`.
- **Heading equivalence**: `_headings_equivalent("1: The House", "Chapter 1: The House")` returns True.
- **Duplicate heading removal**: First line matching chapter title stripped from body text.
- **ALL CAPS normalization**: Becomes title case. Acronyms like "NASA" preserved. Roman numerals preserved.

### 7b. Series Metadata in Intro/Outro
- **Title intro with series**: Intro text starts with `"Book N of the [Series]."`.
- **Article deduplication**: `"The Bas-Lag"` gets `"of The Bas-Lag"` not `"of the The Bas-Lag"`.
- **Outro format**: Ends with `"Book N of [Series]."`.
- **Decimal series index**: `"2.5"` preserved in output.

### 7c. Voice Resolution
- **Formula resolution**: `_chapter_voice_spec` uses `resolved_voice` formula from speaker config.
- **Chunk fallback**: `_chunk_voice_spec` falls back to resolved formula when no chunk-level override.
- **Voice collection**: `_collect_required_voice_ids` extracts voice IDs from formulas and always includes `VOICES_INTERNAL`.

---

## 8. Voice System

### 8a. Voice Cache
- **Download on miss**: `ensure_voice_assets` downloads via `hf_hub_download`. File pattern: `voices/af_nova.pt`.
- **Skip on hit**: Already-cached voices not re-downloaded.
- **Comprehensive collection**: Gathers voices from job.voice, chapters[].voice_formula, chunks[].voice, speakers[].voice_formula.

### 8b. Voice Formula Resolution
- **No-pipeline safety**: `_resolve_voice(None, formula, use_gpu=False)` returns formula unchanged.
- **SuperTonic fallback**: Kokoro formula falls back to valid `DEFAULT_SUPERTONIC_VOICES` entry.

---

## 9. Speaker Analysis (test_speaker_analysis.py)

### Behavioral Claims
- **Gender inference from pronouns**: "He/his" = male, "She/her" = female, no pronouns = unknown.
- **Leading stopword removal**: `"But Volescu said..."` produces key `"volescu"`.
- **Threshold suppression**: Speakers below threshold get `suppressed=True`.
- **Context paragraphs in excerpts**: Sample quotes include paragraph before and after.

---

## 10. Service Layer (test_service.py)

### Behavioral Claims
- **Job processing lifecycle**: Enqueue -> runner called -> COMPLETED with progress=1.0.
- **Default values**: chunk_level="paragraph", speaker_mode="single", chunks=[], generate_epub3=False.
- **Job logging**: `job.add_log("msg", level="error")` emits to stream handlers and appends to job.logs.
- **Logger failure fallback**: Fallback prints to stderr with "Logging failed for job {id}".
- **Retry removes failed job**: `service.retry(job.id)` creates new job with different ID, removes original.
- **Audiobookshelf metadata**: `book_number` key for seriesSequence, normalizes "Book 7 of the Series" to "7", preserves decimals.

---

## 11. Audiobookshelf Client (test_audiobookshelf_client.py)

### Behavioral Claims
- **Upload fields structure**: Produces `fields["series"]`, `fields["seriesSequence"]`, and JSON metadata payload.
- **Alternate key normalization**: `series_index: "Book 3"` normalized to `seriesSequence: "3"`.
- **Decimal preservation**: `seriesSequence: "0.5"` passes through unchanged.

---

## 12. Calibre OPDS Client (test_calibre_opds.py)

### Behavioral Claims
- **Series metadata extraction**: `<calibre:series>` and `<calibre:series_index>` map to entry fields.
- **Subtitle extraction**: `<calibre_md:subtitle>` maps to `entry.subtitle`.
- **Series from categories**: `<category scheme="...series" term="Name #N">` extracts series name and index.
- **Author exclusion from series**: Categories matching author name not mapped as series.
- **Summary parsing**: Structured summaries with `RATING:`, `TAGS:`, `SERIES:` prefixes parsed into fields.
- **URL resolution**: Relative URLs resolved against catalog base.
- **Format filtering**: Unsupported MIME types excluded; PDF and EPUB preserved.
- **Search filtering**: Multi-word queries require all terms to match.
- **Pagination traversal**: Follows `next` links across pages.
- **Navigation traversal**: Follows navigation subsection links for nested feeds.
- **Browse letter**: Finds matching letter nav entry and fetches sub-feed with pagination.

---

## 13. Text Normalization (test_text_normalization.py)

### Behavioral Claims
- Title abbreviation expansion: Dr.->Doctor, Mr.->Mister, Ms.->Miz
- Suffix expansion with case preservation
- Terminal punctuation appended when missing
- Roman numeral title conversion (majority-based)
- Grouped numbers: 35,000 -> thirty-five thousand
- Numeric ranges: 1-3 -> one to three
- Fractions: 1/2 -> one half
- Year pronunciation: 1924 -> nineteen hundred twenty four
- Roman numerals in body: Chapter IV -> chapter four
- ALL CAPS quotes normalized to title case
- Recent years: 2025 -> twenty twenty five
- 2000s: 2005 -> two thousand five
- Address context suppresses year pronunciation
- Configurable contraction expansion, sibilant possessives, decades
- Sub-dollar currency: $0.99 -> cents only
- ISO dates expanded
- Internet slang configurable
- spaCy disambiguation: It's been -> It has been; It's cold -> It is cold
- Currency magnitude: $2 million -> two million dollars

---

## 14. Chunking

### Behavioral Claims
- **Chapter grouping**: Sorts by chunk_index within each chapter_index group.
- **Voice spec priority**: Chunk voice > speaker voice > fallback.
- **Title abbreviation merging**: Abbreviation periods don't split sentences.
- **Display text whitespace**: Preserves double spaces and trailing newlines.
- **TTS text preference order**: text > original_text > normalized_text (preserves manual overrides).

---

## 15. Date Normalization (test_date_normalization_comprehensive.py)

- Standard years with appropriate century/decade style
- Address context (within ~60 chars) suppresses year pronunciation
- 2000-2009: "two thousand N" style
- "addresses" plural also triggers suppression

---

## 16. Debug TTS Samples

- EPUB contains all expected marker codes
- Default settings expand contractions, titles, remove footnotes, add terminal punctuation
- POST /settings/debug/run produces manifest.json with overall.wav and per-case WAVs
- Each normalization category has at least 5 test samples
- Profile voices resolved before TTS pipeline

---

## 17. FFMetadata

- Valid FFmpeg metadata format with `;FFMETADATA1` header
- Chapter timing converted to milliseconds
- Voice metadata included in chapter blocks

---

## 18. Manual Overrides

- Manual override replaces token in text
- Manual overrides take precedence over pronunciation_overrides for same token

---

## 19. Output Paths

- Timestamped folder: `{timestamp}_{sanitized_filename}`
- Default layout: audio_dir == subtitle_dir == project_root
- Project layout: creates audio/, subtitles/, metadata/ subdirectories

---

## 20. PDF Structure

- TOC-based structure with correct titles and src references
- No-TOC fallback: single "Pages" parent with all pages as children
- Nested TOC levels become children of preceding parent

---

## 21. Form Handling

- Custom mix stores voice_formula and resolved_voice on speaker
- Speaker reference stores as resolved_voice
- Profile resolution converts blend weights to formula
- SuperTonic profile becomes speaker reference without overriding tts_provider

---

## 22. Settings Integration Secrets

- Blank API token/password fields preserve previously-stored secrets
- Absent integration fields preserve stored secrets

---

## 23. SuperTonic Unsupported Characters

- Unsupported characters stripped and retried
- All-unsupported input returns empty segment list (no crash)

---

## 24. Utils Cache

- ABOGEN_TEMP_DIR configures HF_HOME, XDG_CACHE_HOME, HUGGINGFACE_HUB_CACHE, TRANSFORMERS_CACHE

---

## Summary: Integration Boundaries

### What Is Mocked
- hf_hub_download, epub.read_epub, Flask test client, TTS pipeline, OPDS feed fetching, logging handlers, heavy dependencies via stubs

### What Uses Real Dependencies
- ebooklib, fitz/PyMuPDF, numpy, kokoro_text_normalization, spaCy (when available), Flask app, num2words

### Notable Gaps
- No end-to-end TTS synthesis test
- No network integration tests (mocked fetch)
- PyQt tests require display server
- spaCy tests conditionally skipped
- No audio file format validation tests
- No concurrent job processing tests
