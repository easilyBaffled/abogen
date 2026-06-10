# Book Parsing Analysis

Chunk: book-parsing
Files analyzed:
- `abogen/book_parser.py` (1071 lines) - Legacy class-hierarchy parser
- `abogen/text_extractor.py` (1060 lines) - Refactored dataclass-based parser

---

## Architectural Comparison

| Aspect | book_parser.py (Legacy) | text_extractor.py (Refactored) |
|--------|------------------------|-------------------------------|
| Used by | PyQt GUI | Web UI |
| Pattern | Class hierarchy (BaseBookParser ABC) | Single function (extract_from_path) |
| Output | Mutable dicts (content_texts, content_lengths) | Immutable dataclasses (ExtractionResult) |
| Navigation | Tree-based processed_nav_structure | Flat sorted NavEntry list |
| Formats | EPUB, PDF, Markdown | EPUB, PDF, Markdown, TXT |
| Text cleanup | Imports from subtitle_utils | Imports from utils |
| Error recovery | Monkey-patches for malformed EPUBs | No monkey-patching |

---

## book_parser.py: Legacy Parser

### Class Hierarchy

- `BaseBookParser` (ABC): defines interface - `get_chapters()`, `get_formatted_text()`, `get_content()`, `file_type` property
- `PdfParser`: PDF via PyMuPDF (fitz)
- `MarkdownParser`: Markdown via python-markdown
- `EpubParser`: EPUB via ebooklib

### Factory Function

`get_book_parser(file_path, file_type=None)` -> returns appropriate parser subclass. Extension detection: .pdf, .epub, .md. Explicit `file_type` parameter overrides extension.

### PdfParser

- Extracts text per page with `page.get_text("text")`
- Cleans footnote markers (`[12]` patterns) from extracted text
- Builds navigation from PDF TOC (`doc.get_toc()`)
- No-TOC fallback: single "Pages" parent with all pages as children
- Nested TOC: level-2 entries become children of preceding level-1
- Content keys: `page_1`, `page_2`, etc.

### MarkdownParser

- Splits on `# ` headings (level-1 only)
- Generates slugified IDs (`chapter-1`, `chapter-2`)
- Converts markdown to HTML then extracts text

### EpubParser

- Content keys: original file names (`intro.xhtml`, `chap1.xhtml`)
- Navigation discovery: tries HTML nav (`<nav epub:type="toc">`), then NCX, then heuristic scan
- Position-robust anchor finding for content slicing between nav entries
- Metadata extraction: title, author from EPUB OPF

### Output Format

- `get_chapters()`: returns `[(id, display_title), ...]`
- `get_formatted_text()`: returns text with `<<CHAPTER_MARKER:Title>>` delimiters
- `content_texts`: dict mapping chapter_id -> text content
- `content_lengths`: dict mapping chapter_id -> character count

---

## text_extractor.py: Refactored Parser

### Data Model

```python
@dataclass
class ExtractedChapter:
    id: str
    title: str
    text: str
    characters: int
    cover_image: Optional[bytes] = None

@dataclass
class ExtractionResult:
    chapters: List[ExtractedChapter]
    metadata: Dict[str, Any]
    combined_text: str
    total_characters: int
    cover_image: Optional[bytes] = None

@dataclass
class NavEntry:
    id: str
    title: str
    src: str
    position: int
    has_content: bool = True

class MetadataSource(Enum):
    OPF = "opf"
    NCX = "ncx"
    HTML_NAV = "html_nav"
```

### Public API

`extract_from_path(path: Path) -> ExtractionResult`: Single entry point. Dispatches based on extension:
- .epub -> _extract_epub
- .pdf -> _extract_pdf
- .md -> _extract_markdown
- .txt -> _extract_txt (not in legacy parser)

### Format-Specific Logic

**EPUB:**
- Richer metadata: series, series_index from Calibre OPF tags
- Cover image extraction from EPUB manifest
- Same nav discovery strategies as legacy (HTML nav, NCX, heuristic)
- Same content slicing logic (position-robust anchor finding)
- YAML frontmatter NOT supported (that's markdown-only)

**PDF:**
- Full metadata from `doc.metadata`: title, author, subject, keywords, creator
- Same TOC-based navigation as legacy
- Footnote cleaning with inline regex

**Markdown:**
- YAML frontmatter parsing for metadata
- Heading-based chapter splitting
- HTML conversion then text extraction

**TXT (new):**
- Detects `<<CHAPTER_MARKER:>>` tags for splitting
- Falls back to single chapter if no markers
- No metadata extraction

### Key Invariant

`result.total_characters == calculate_text_length(result.combined_text) == sum(ch.characters for ch in result.chapters)`

All three character count measures must agree (verified by tests).

---

## Shared EPUB Parsing Logic

Both parsers implement identical strategies for EPUB navigation:

1. **Standard nav discovery**: Look for items with type ITEM_NAVIGATION or properties=["nav"]
2. **HTML nav parsing**: Parse `<nav epub:type="toc"><ol><li><a href="...">` structures
3. **Nested list handling**: Nested `<ol>` within `<li>` produces parent+child chapters
4. **Span grouping**: `<li><span>Part I</span><ol>...</ol></li>` -> non-content grouping node
5. **NCX fallback**: Parse `<navPoint>` elements from NCX document
6. **Heuristic fallback**: Scan ITEM_DOCUMENT files for nav content
7. **Content slicing**: Use anchor positions to extract text between nav entries
8. **Missing file recovery**: Gracefully skip files missing from ZIP (no crash)
9. **Ordered list conversion**: `<ol start="N">` -> `N) Item` text format

---

## Error Handling

### book_parser.py
- Missing EPUB files in manifest: caught, logged, skipped
- Malformed HTML: BeautifulSoup tolerant parsing
- Missing metadata: returns empty strings/None

### text_extractor.py
- Missing EPUB files: graceful skip (tested)
- Invalid PDF: PyMuPDF exception bubbles up
- Empty text after extraction: chapter still created with 0 characters
- Encoding detection: uses `detect_encoding()` with utf-8 fallback
