# Behavioral Specification: Book Parsing

## Module Identity

**Paths**: `abogen/book_parser.py`  
**Role**: Extracts text content, chapter structure, and metadata from EPUB, PDF, Markdown, and plain text files  
**Boundaries**: Input layer only. Produces `content_texts` dict and `processed_nav_structure`. Does NOT normalize text, perform TTS, or manage jobs.  
**Dependencies**: ebooklib, BeautifulSoup4, PyMuPDF (fitz), markdown

---

## Public Interface

| Class | Formats | Library |
|-------|---------|---------|
| `EpubParser` | .epub | ebooklib + BeautifulSoup4 |
| `PdfParser` | .pdf | PyMuPDF (fitz) |
| `MarkdownParser` | .md, .markdown | python-markdown |
| `BaseBookParser` | ABC | — |

### Common Interface (BaseBookParser)

| Method | Returns | Description |
|--------|---------|-------------|
| `load()` | None | Load/parse file into memory |
| `close()` | None | Release file handles |
| `process_content(replace_single_newlines)` | (content_texts, content_lengths) | Extract text + compute lengths |
| `get_chapters()` | List[(id, title)] | Flattened chapter list from nav structure |
| `get_formatted_text()` | str | Full text with `<<CHAPTER_MARKER:name>>` delimiters |
| `get_metadata()` | dict | title, author, language |

### Context Manager Protocol
**Given** a parser instance  
**When** used as context manager  
**Then** `__enter__` returns self, `__exit__` calls `close()`  
**Source**: `book_parser.py:50-56`

---

## PDF Parser

### Load
**Given** a PDF file path  
**When** `PdfParser(path)` constructed  
**Then** opens via `fitz.open()`; raises on failure  
**Source**: `book_parser.py:118-123`

### Text Extraction
**Given** loaded PDF  
**When** `process_content()` called  
**Then** extracts text per page via `fitz.Page.get_text()`, applies `clean_text()`, removes: bracketed numbers `[123]`, standalone page numbers, trailing page numbers, page numbers with dashes  
**Source**: `book_parser.py:135-151`

### TOC-Based Navigation
**Given** PDF with table of contents  
**When** `self.pdf_doc.get_toc()` returns entries  
**Then** builds hierarchical nav structure; bookmarks become nodes; non-bookmarked pages attach to preceding bookmark as children  
**Source**: `book_parser.py:189-281`

### No-TOC Fallback
**Given** PDF without TOC  
**When** `get_toc()` returns empty  
**Then** creates flat "Pages" parent with all pages as children; page titles from first line (if < 100 chars)  
**Source**: `book_parser.py:156-178`

### Page Title Heuristic
**Given** extracted page text  
**When** generating page title  
**Then** uses `"Page {n}"` + first line if non-empty and under 100 chars  
**Source**: `book_parser.py:181-187`

---

## Markdown Parser

### Load
**Given** a markdown file path  
**When** `MarkdownParser(path)` constructed  
**Then** detects encoding via `detect_encoding()`, reads file with `errors="replace"`  
**Source**: `book_parser.py:293-300`

### TOC from Headers
**Given** markdown with headings  
**When** `process_content()` called  
**Then** uses `markdown.Markdown(extensions=["toc", "fenced_code"])` to generate HTML + TOC tokens; converts TOC tokens to unified nav structure  
**Source**: `book_parser.py:323-333`

### Section Splitting
**Given** generated HTML with headers  
**When** splitting content  
**Then** finds header positions in HTML string, splits at header boundaries, extracts text via BeautifulSoup, removes header tag from section content  
**Source**: `book_parser.py:356-393`

### No-Header Fallback
**Given** markdown without any headings  
**When** no TOC generated  
**Then** treats entire document as single chapter with id `"markdown_content"`  
**Source**: `book_parser.py:338-342`

### Empty Section Handling
**Given** header with no following content  
**When** section text is empty after header removal  
**Then** uses header name as chapter content  
**Source**: `book_parser.py:392-393`

---

## EPUB Parser

### Load with Error Recovery
**Given** EPUB with missing referenced files  
**When** `epub.read_epub()` raises KeyError  
**Then** monkey-patches `EpubReader.read_file` to return empty bytes, retries read, then restores original method  
**Source**: `book_parser.py:412-438`

### Metadata Extraction
**Given** loaded EPUB  
**When** `_extract_book_metadata()` called  
**Then** extracts DC metadata: title (fallback: filename), author (fallback: "Unknown Author"), language (fallback: "en")  
**Source**: `book_parser.py:454-474`

### Navigation Detection (2 types)
**Given** EPUB structure  
**When** `_identify_nav_item()` called  
**Then** identifies navigation document: EPUB3 `nav` (HTML5 nav element) or EPUB2 `ncx` (XML navMap)  
**Source**: `book_parser.py:696`

### NCX Parsing (EPUB2)
**Given** NCX navigation  
**When** parsing navPoints  
**Then** recursively traverses `<navPoint>` elements; extracts title from `<navLabel><text>`, src from `<content>` href; resolves document position via fragment ID matching  
**Source**: `book_parser.py:542-604`

### HTML Nav Parsing (EPUB3)
**Given** HTML5 nav element  
**When** parsing nav `<ol>/<li>` structure  
**Then** extracts links/spans from `<li>` elements; handles nested `<ol>` as children; resolves hrefs to doc positions  
**Source**: `book_parser.py:632-694`

### Fragment Position Resolution
**Given** an anchor/fragment ID in a document  
**When** `_find_position_robust()` called  
**Then** tries: (1) BeautifulSoup `find(id=)`, (2) regex for `id|name` attribute, (3) string search for `id="fragment"` or `name="fragment"`; returns 0 if not found  
**Source**: `book_parser.py:492-540`

### Document Key Resolution
**Given** an href from navigation  
**When** `_find_doc_key()` called  
**Then** tries: raw href, URL-decoded href, basename match against doc_order keys  
**Source**: `book_parser.py:476-490`

### Nav Processing Failure Fallback
**Given** nav parsing raises exception  
**When** processing EPUB content  
**Then** falls back to `_process_epub_content_spine_fallback()` (flat spine order)  
**Source**: `book_parser.py:448-450`

### Spine Fallback
**Given** no usable navigation  
**When** `_process_epub_content_spine_fallback()` called  
**Then** processes all spine items (XHTML documents) in order; uses first heading or "Chapter N" as title  
**Source**: `book_parser.py:983`

---

## Unified Navigation Structure

All parsers produce the same structure:

```python
[{
    "title": str,       # Display name
    "src": str,         # Content key (page_id, header_id, nav href)
    "children": [...],  # Nested structure (same shape)
    "has_content": bool # Whether this node has associated text
}]
```

### get_chapters() Flattening
**Given** nav structure exists  
**When** `get_chapters()` called  
**Then** depth-first traversal collecting `(src, title)` pairs for nodes where `has_content=True`  
**Source**: `book_parser.py:69-87`

### get_chapters() No-Nav Fallback
**Given** empty `processed_nav_structure`  
**When** `get_chapters()` called  
**Then** returns `(ch_id, ch_id)` pairs from `content_texts` keys  
**Source**: `book_parser.py:83-86`

---

## Text Cleaning

### clean_text() (from subtitle_utils)
Applied to all extracted text. Normalizes whitespace, strips control characters, collapses multiple blank lines.

### PDF-Specific Cleanup Patterns
| Pattern | Target |
|---------|--------|
| `\[\s*\d+\s*\]` | Bracketed footnote numbers |
| `^\s*\d+\s*$` | Standalone page numbers |
| `\s+\d+\s*$` | Trailing page numbers |
| `\s+[-–—]\s*\d+\s*[-–—]?\s*$` | Page numbers with dashes |

**Source**: `book_parser.py:18-23`

---

## Error Behaviors

### PDF Load Failure
**Given** invalid/corrupt PDF  
**When** `fitz.open()` raises  
**Then** logs error, re-raises exception  
**Source**: `book_parser.py:121-123`

### Markdown Read Failure
**Given** file read error  
**When** `load()` catches Exception  
**Then** logs error, sets `markdown_text = ""`  
**Source**: `book_parser.py:299-300`

### EPUB Missing File Recovery
**Given** KeyError during EPUB read  
**When** referenced file missing from ZIP  
**Then** patches reader to return `b""`, retries, restores original  
**Source**: `book_parser.py:415-438`

### Encoding Detection Fallback
**Given** charset detection fails  
**When** reading markdown/txt  
**Then** `detect_encoding()` returns `"utf-8"` default; `errors="replace"` handles bad bytes  
**Source**: `book_parser.py:296`

---

## State Management

### Instance State (per parser)
- `book_path`: normalized absolute path
- `content_texts`: dict[str, str] mapping chapter_id → text
- `content_lengths`: dict[str, int] mapping chapter_id → char count
- `book_metadata`: dict with title/author/language
- `processed_nav_structure`: hierarchical nav list

### No Module-Level Mutable State
All regex patterns pre-compiled at module level (immutable).

---

## Invariants

1. **Path normalized**: `book_path` always `os.path.normpath(os.path.abspath(path))`
2. **load() called in __init__**: Parser is ready after construction
3. **content_texts keys match nav src**: Every `has_content=True` node's `src` exists as key in `content_texts`
4. **Chapter markers format**: `<<CHAPTER_MARKER:name>>` with exact delimiters
5. **Encoding always detected**: `detect_encoding()` never raises; defaults to UTF-8
6. **Close idempotent**: Calling `close()` multiple times is safe
7. **EPUB monkey-patch temporary**: Original `read_file` always restored in `finally`
8. **PDF page IDs sequential**: Always `page_{n+1}` format (1-indexed)
9. **Markdown content ID**: Without headers, always `"markdown_content"`
10. **Metadata always returns dict**: Never None; fields have string defaults
