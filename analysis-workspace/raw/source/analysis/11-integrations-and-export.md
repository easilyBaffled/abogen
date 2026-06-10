# Comprehensive Analysis: Integrations and Export (Chunk 11)

## 1. `abogen/integrations/__init__.py`

Simple package initializer with docstring: `"""Integration clients for external services."""`. Exports nothing explicitly.

---

## 2. `abogen/integrations/audiobookshelf.py`

### Exception Class

**`AudiobookshelfUploadError(RuntimeError)`** - Raised when any upload or API interaction fails.

### Configuration Dataclass

**`AudiobookshelfConfig`** (frozen dataclass):

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `base_url` | `str` | (required) | Server base URL |
| `api_token` | `str` | (required) | Bearer token for authentication |
| `library_id` | `Optional[str]` | `None` | Target library identifier |
| `collection_id` | `Optional[str]` | `None` | Optional collection to place upload in |
| `folder_id` | `Optional[str]` | `None` | Target folder (name, path segment, or ID) |
| `verify_ssl` | `bool` | `True` | Whether to verify TLS certificates |
| `send_cover` | `bool` | `True` | Whether to include cover image |
| `send_chapters` | `bool` | `True` | Whether to include chapter metadata |
| `send_subtitles` | `bool` | `True` | Whether to include subtitle files |
| `timeout` | `float` | `3600.0` | HTTP timeout in seconds (1 hour default) |

**Method `normalized_base_url()`**: Strips trailing slashes and removes trailing `/api` suffix. Raises `ValueError` if `base_url` empty/blank.

---

### Client Class: `AudiobookshelfClient`

#### Constructor `__init__(self, config: AudiobookshelfConfig)`
- Validates `api_token` present (raises `ValueError` if not).
- Normalizes base URL and constructs `_client_base_url` with trailing slash.
- Initializes `_folder_cache` to `None`.

#### HTTP Client Setup

**`_open_client(self) -> httpx.Client`**
- `Authorization: Bearer {token}` header
- `Accept: application/json` header
- Base URL set to `{normalized_base_url}/`
- Configurable timeout and SSL verification

**`_api_path(self, suffix: str) -> str`**: Constructs `"api/{suffix}"` relative paths.

---

#### Public Methods

##### `get_libraries(self) -> List[Dict[str, Any]]`
- **HTTP**: `GET api/libraries`
- **Response**: JSON with `{"libraries": [...]}` array
- **Errors**: Raises `AudiobookshelfUploadError` on any `httpx.HTTPError`

##### `upload_audiobook(self, audio_path, *, metadata, cover_path, chapters, subtitles) -> Dict[str, Any]`
- **Validates**: `audio_path` exists on disk
- **HTTP**: `POST api/upload` (multipart form data)
- **Form Fields** (built by `_build_upload_fields`):
  - `library`: configured library_id
  - `folder`: resolved folder ID (from `_ensure_folder`)
  - `title`: from metadata or audio filename stem
  - `author`: comma-separated author string (optional)
  - `series`: series name (optional)
  - `seriesSequence`: numeric series index (optional)
  - `collectionId`: from config (optional)
  - `metadata`: JSON-serialized metadata dict (including chapters if `send_chapters`)
- **File Entries** (built by `_build_file_entries`):
  - `file0`: audio file (always)
  - `file1`: cover image (if `send_cover` and path exists)
  - `file2+`: subtitle files (if `send_subtitles` and paths exist)
  - MIME types auto-detected via `mimetypes.guess_type`
- **Returns**: Empty dict `{}` on success
- **Errors**: Raises `AudiobookshelfUploadError` with status code and detail (truncated to 200 chars)

##### `find_existing_items(self, title, *, folder_id) -> List[Mapping[str, Any]]`
- Normalizes title via `_normalize_title_value` (collapses whitespace, case-folds).
- **HTTP Requests** (tried in order, stops after first match):
  1. `GET api/folders/{folder_id}/items?library={lib}&search={title}`
  2. `GET api/libraries/{lib}/items?search={title}`
  3. `GET api/items?library={lib}&search={title}`
  4. `GET api/search?query={title}&library={lib}&media=audiobook`
- **Response handling**: Recursively traverses JSON payload to find all mappings with title and ID (via `_extract_candidate_items`).
- **Filtering**: Compares normalized titles; if `folder_id` set, also filters by folder.
- **Auth errors** (401/403): Re-raised as `AudiobookshelfUploadError`.
- **404**: Silently skipped.

##### `delete_items(self, items: Iterable[Mapping | str]) -> None`
- For each item ID, tries both routes:
  1. `DELETE api/items/{item_id}`
  2. `DELETE api/libraries/{lib}/items/{item_id}`
- 200/202/204 = success; 404 continues to next route; others raise error.

##### `resolve_folder(self) -> Tuple[str, str, str]`
- Returns `(folder_id, folder_name, library_name)` via `_ensure_folder()`.

##### `list_folders(self) -> List[Dict[str, str]]`
- **HTTP**: `GET api/libraries/{library_id}`
- Returns all folders with keys: `id`, `name`, `path`, `library`. Sorted by path/name/id.

---

#### Internal Methods

**`_ensure_folder(self) -> Tuple[str, str, str]`**
- Uses cached value if available.
- Loads library metadata, resolves `folder_id` config value by: direct ID match, name match (case-insensitive), full path match, tail-path-segment match.
- Raises `AudiobookshelfUploadError` with suggestions if not found.

**`_normalize_series_sequence(raw)`** (static)
- Handles int, float, string inputs. Extracts numeric portion via regex. Strips trailing zeros from decimal.

---

## 3. `abogen/integrations/calibre_opds.py`

### Constants and Namespaces

```python
ATOM_NS = "http://www.w3.org/2005/Atom"
OPDS_NS = "http://opds-spec.org/2010/catalog"
DC_NS = "http://purl.org/dc/terms/"
CALIBRE_CATALOG_NS = "http://calibre.kovidgoyal.net/2009/catalog"
CALIBRE_METADATA_NS = "http://calibre.kovidgoyal.net/2009/metadata"
```

**Supported MIME types**: EPUB (`application/epub+zip`, `application/zip`, etc.) and PDF (`application/pdf`).

### Exception Class

**`CalibreOPDSError(RuntimeError)`** - Unrecoverable errors from OPDS client.

### Data Classes

**`OPDSLink`**: Fields: `href`, `rel`, `type`, `title`. Has `to_dict()`.

**`OPDSEntry`**: Full book entry with: `id`, `title`, `position`, `authors`, `subtitle`, `updated`, `published`, `summary`, `download` (OPDSLink), `alternate` (OPDSLink), `thumbnail` (OPDSLink), `links`, `series`, `series_index`, `tags`, `rating`, `rating_max`.

**`OPDSFeed`**: Fields: `id`, `title`, `entries` (List[OPDSEntry]), `links` (Dict[str, OPDSLink]).

**`DownloadedResource`**: Fields: `filename`, `mime_type`, `content` (bytes).

### Standalone Function

**`feed_to_dict(feed: OPDSFeed) -> Dict`**: Delegates to `feed.to_dict()`.

---

### Client Class: `CalibreOPDSClient`

#### Constructor
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `base_url` | `str` | (required) | OPDS catalog URL |
| `username` | `Optional[str]` | `None` | Basic auth username |
| `password` | `Optional[str]` | `None` | Basic auth password |
| `timeout` | `float` | `15.0` | HTTP timeout |
| `verify` | `bool` | `True` | TLS verification |

Uses `httpx.BasicAuth` if username provided. Sets `User-Agent: abogen-calibre-opds/1.0`.

#### Public Methods

##### `fetch_feed(self, href=None, *, params=None) -> OPDSFeed`
- **HTTP**: `GET {resolved_url}`, follows redirects.
- Parses XML as Atom/OPDS feed.
- Raises `CalibreOPDSError` on HTTP errors.

##### `search(self, query: str, start_href=None) -> OPDSFeed`
Multi-strategy search:
1. **OpenSearch**: Looks for `rel="search"` link, fetches OpenSearch description, substitutes `{searchTerms}`.
2. **Direct template**: Checks if feed's search link has `{searchTerms}` in href.
3. **Common path guesses**: Tries `search?query=`, `search?q=`, `?search=`.
4. **Local search (BFS crawl)**: Crawls catalog via pagination/navigation links, filtering locally.

All results filtered through `_filter_feed_entries` (minimum relevance score 10).

##### `browse_letter(self, letter, *, start_href=None, max_pages=40) -> OPDSFeed`
- Navigates alphabetically through sorted feeds.
- Handles `#` for numeric entries, normalizes `0-9`/`NUMERIC` to `#`.
- Detects browse mode (`title`, `author`, `series`, `generic`) from feed title.
- Smart jump using pagination offsets and English letter distribution table.

##### `download(self, href: str) -> DownloadedResource`
- **HTTP**: `GET {resolved_url}`, follows redirects.
- Extracts filename from `Content-Disposition` header or URL path.
- Returns raw bytes with MIME type.

---

#### Relevance Scoring (`_calculate_match_score`)
- Exact title match: 1000
- Phrase in title: 500
- Phrase in author: 300
- Phrase in series: 200
- Exact tag match: 100
- Per-token: title word boundary 50, author 40, series 30, tag 30, summary 15

---

## 4. `abogen/epub3/__init__.py`

Exports: `EPUB3PackageBuilder` (class) and `build_epub3_package` (convenience function).

---

## 5. `abogen/epub3/exporter.py`

### Data Classes

**`ChunkOverlay`** (slots=True):
| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | Unique chunk identifier (sanitized for XML IDs) |
| `text` | `str` | Display text |
| `original_text` | `Optional[str]` | Original unsynthesized text |
| `start` | `Optional[float]` | Audio clip begin (seconds) |
| `end` | `Optional[float]` | Audio clip end (seconds) |
| `speaker_id` | `str` | Speaker identifier |
| `voice` | `Optional[str]` | Voice name/ID |
| `level` | `Optional[str]` | Chunk level |
| `group_id` | `Optional[str]` | Group identifier for rendering |

**`ChapterDocument`** (slots=True):
| Field | Type | Description |
|-------|------|-------------|
| `index` | `int` | Zero-based chapter index |
| `title` | `str` | Chapter title |
| `xhtml_name` | `str` | e.g., `chapter_0001.xhtml` |
| `smil_name` | `str` | e.g., `chapter_0001.smil` |
| `chunks` | `List[ChunkOverlay]` | Ordered chunks |
| `start` | `Optional[float]` | Chapter audio start time |
| `end` | `Optional[float]` | Chapter audio end time |

---

### Class: `EPUB3PackageBuilder`

#### Constructor Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `output_path` | `Path` | Destination .epub file path |
| `book_id` | `str` | Unique identifier (falls back to UUID4) |
| `extraction` | `ExtractionResult` | Parsed text/chapters from source |
| `metadata_tags` | `Dict[str, Any]` | User-supplied metadata |
| `chapter_markers` | `Sequence[Dict]` | Chapter timing data |
| `chunk_markers` | `Sequence[Dict]` | Per-chunk timing data |
| `chunks` | `Iterable[Dict]` | Full chunk details |
| `audio_path` | `Path` | Path to audio file |
| `speaker_mode` | `str` | `"single"` or multi-speaker |
| `cover_image_path` | `Optional[Path]` | Cover image |
| `cover_image_mime` | `Optional[str]` | Cover MIME override |

#### Method: `build(self) -> Path`

Produces complete EPUB 3 ZIP archive:

```
/
├── mimetype                    (uncompressed, first ZIP entry)
├── META-INF/
│   └── container.xml
└── OEBPS/
    ├── content.opf
    ├── nav.xhtml
    ├── text/
    │   └── chapter_NNNN.xhtml  (one per chapter)
    ├── smil/
    │   └── chapter_NNNN.smil   (one per chapter)
    ├── audio/
    │   └── {audio_filename}
    ├── images/
    │   └── {cover_filename}    (if cover provided)
    └── styles/
        └── style.css
```

---

### SMIL Synchronization Logic

1. **Time format**: `_format_smil_time(value)` converts seconds to `HH:MM:SS.mmm`. None/negative -> `00:00:00.000`.

2. **Chunk-to-audio mapping**: Each `ChunkOverlay` carries `start`/`end` times mapping to `clipBegin`/`clipEnd` in SMIL `<audio>` element.

3. **Text-to-audio mapping**: SMIL `<text>` references XHTML chapter with fragment ID pointing to chunk's `<span>` id.

4. **Chapter-level sequencing**: All chunks in single `<seq>` element with `epub:textref` to chapter XHTML.

5. **Marker resolution pipeline** (in `_build_chapter_documents`):
   - chunk_markers grouped by `chapter_index`, sorted by `(chunk_index, start_time)`
   - If no markers exist but chunks exist, synthetic markers created from chunk data
   - If still none, single auto-marker covering entire chapter timespan

6. **Overlay construction** (in `_build_overlays_for_chapter`):
   - IDs normalized (alphanumeric, underscore, hyphen; max 120 chars). Duplicates get `_dup` suffix.
   - Group IDs: for "sentence" level, group = base chunk ID without `_sN` suffix.

7. **Original text restoration** (`_restore_original_chunk_text`):
   - Uses regex with `\s+` for flexible whitespace matching
   - Cursor tracks position; results cached in `_CHUNK_REGEX_CACHE`

---

### Error Handling

**Audiobookshelf Client**:
- No automatic retry. Every HTTP error -> `AudiobookshelfUploadError`.
- `find_existing_items` tries multiple routes; stops on first match; swallows non-auth errors.
- `delete_items` tries two DELETE routes per item; 404 = "already gone".
- Auth failures (401/403) immediately re-raised.

**Calibre OPDS Client**:
- No automatic retry.
- `search` degrades gracefully: server search -> common URL patterns -> local BFS crawl.
- `fetch_feed` raises immediately on HTTP errors.
- `browse_letter` swallows errors during navigation link fetching.

**EPUB3 Builder**:
- Raises `FileNotFoundError` if audio file missing.
- No error handling for disk I/O (exceptions propagate naturally).
- Invalid metadata handled gracefully (None checks, safe type coercion).

---

### Audio MIME Type Mapping

| Extension | MIME Type |
|-----------|-----------|
| `.mp3` | `audio/mpeg` |
| `.m4a` | `audio/mp4` |
| `.m4b` | `audio/mp4` |
| `.aac` | `audio/aac` |
| `.wav` | `audio/wav` |
| `.flac` | `audio/flac` |
| `.ogg` | `audio/ogg` |
| `.opus` | `audio/ogg` |
| (default) | `audio/mpeg` |
