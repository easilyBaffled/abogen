# Behavioral Specification: Integrations and Export

## Module Identity

**Paths**: `abogen/integrations/audiobookshelf.py`, `abogen/integrations/calibre_opds.py`, `abogen/epub3/exporter.py`  
**Role**: External service communication (Audiobookshelf, Calibre OPDS) + output packaging (EPUB3 with SMIL)  
**Boundaries**: Web UI-only features. Does NOT perform TTS, text processing, or job management.  
**Dependencies**: httpx (Audiobookshelf), urllib/xml.etree (OPDS), zipfile/xml (EPUB3)

---

## Audiobookshelf Client

### Configuration (AudiobookshelfConfig frozen dataclass)

| Field | Type | Default | Required |
|-------|------|---------|----------|
| `base_url` | str | -- | Yes |
| `api_token` | str | -- | Yes |
| `library_id` | Optional[str] | None | No |
| `collection_id` | Optional[str] | None | No |
| `folder_id` | Optional[str] | None | No |
| `verify_ssl` | bool | True | No |
| `send_cover` | bool | True | No |
| `send_chapters` | bool | True | No |
| `send_subtitles` | bool | True | No |
| `timeout` | float | 3600.0 | No |

**Source**: `audiobookshelf.py:23-33`

### URL Normalization
**Given** a base_url value  
**When** `normalized_base_url()` is called  
**Then** strips trailing slashes; removes trailing `/api` suffix (case-insensitive)  
**Source**: `audiobookshelf.py:35-44`

### URL Empty
**Given** empty or whitespace-only base_url  
**When** `normalized_base_url()` is called  
**Then** raises `ValueError("Audiobookshelf base URL is required")`  
**Source**: `audiobookshelf.py:37-38`

### Token Validation
**Given** empty or falsy api_token  
**When** `AudiobookshelfClient.__init__()` is called  
**Then** raises `ValueError("Audiobookshelf API token is required")`  
**Source**: `audiobookshelf.py:50-51`

### Bearer Authentication
**Given** a configured client  
**When** HTTP requests are made  
**Then** includes header `Authorization: Bearer {api_token}`  
**Source**: `audiobookshelf.py:115-125`

### Upload Flow
**Given** valid audio_path and metadata  
**When** `upload_audiobook()` is called  
**Then** sends POST to `api/upload` with multipart form data including: library, folder (resolved ID), title, optional author/series/collection  
**Source**: `audiobookshelf.py:93-98`

### Audio Path Validation
**Given** audio_path that does not exist  
**When** `upload_audiobook()` is called  
**Then** raises `AudiobookshelfUploadError("Audio path does not exist: {path}")`  
**Source**: `audiobookshelf.py:87-88`

### File Entry Numbering
**Given** audio, cover, and subtitle files  
**When** building upload entries  
**Then** numbered sequentially: `file0` (audio always), `file1` (cover if send_cover and exists), `file2+` (subtitles if send_subtitles and each exists)  
**Source**: `audiobookshelf.py:174-193`

### Upload HTTP Error
**Given** server returns HTTP error  
**When** upload catches `httpx.HTTPStatusError`  
**Then** raises `AudiobookshelfUploadError` with status code and response text (truncated to 200 chars)  
**Source**: `audiobookshelf.py:99-109`

### Upload Connection Error
**Given** network-level failure  
**When** upload catches `httpx.HTTPError`  
**Then** raises `AudiobookshelfUploadError` with exception message  
**Source**: `audiobookshelf.py:110-111`

---

## Folder Resolution (4 Strategies)

### Strategy 1: Direct ID Match
**Given** folder_id matches a folder's `id` field exactly  
**When** resolving folder  
**Then** that folder returned immediately  
**Source**: `audiobookshelf.py:399-405`

### Strategy 2: Name Match (Case-Insensitive)
**Given** folder_id matches a folder's display name after normalization  
**When** resolving folder  
**Then** that folder returned  
**Source**: `audiobookshelf.py:409-417`

### Strategy 3: Full Path Match
**Given** folder_id matches a folder's path (from keys: fullPath, fullpath, path, folderPath, virtualPath)  
**When** resolving folder  
**Then** that folder returned  
**Source**: `audiobookshelf.py:419-423`

### Strategy 4: Tail Path Segment
**Given** folder_id (without path separators) matches last segment of a folder's path  
**When** resolving folder  
**Then** that folder returned  
**Source**: `audiobookshelf.py:429-432`

### Folder Cache
**Given** folder previously resolved  
**When** `_ensure_folder()` called again  
**Then** cached tuple returned without network calls  
**Source**: `audiobookshelf.py:387-388`

### Folder Not Found
**Given** no folder matches after all strategies  
**When** resolution exhausted  
**Then** raises `AudiobookshelfUploadError` with suggestion message  
**Source**: `audiobookshelf.py:435-438`

---

## Find/Delete Operations

### Search Endpoint Order (4 Endpoints)
**Given** title and optional folder_id  
**When** `find_existing_items()` builds request list  
**Then** order is:
1. `GET api/folders/{folder_id}/items` (only if folder_id)
2. `GET api/libraries/{library_id}/items`
3. `GET api/items`
4. `GET api/search`
**Source**: `audiobookshelf.py:298-332`

### First-Match Short-Circuit
**Given** multiple search endpoints  
**When** one returns matching items  
**Then** subsequent endpoints not attempted  
**Source**: `audiobookshelf.py:269-270`

### Auth Failure Propagation
**Given** search returns HTTP 401 or 403  
**When** processing response  
**Then** raises `AudiobookshelfUploadError` immediately  
**Source**: `audiobookshelf.py:244-250`

### Delete Via Dual Routes
**Given** item ID to delete  
**When** `_delete_single_item()` called  
**Then** tries: `DELETE api/items/{id}` then `DELETE api/libraries/{lib}/items/{id}`  
**Source**: `audiobookshelf.py:334-359`

### Delete Success Codes
**Given** DELETE response status 200, 202, or 204  
**When** processed  
**Then** item considered successfully deleted  
**Source**: `audiobookshelf.py:347-348`

---

## Calibre OPDS Client

### Feed Fetching
**Given** base_url configured  
**When** `fetch_feed(href)` is called  
**Then** fetches Atom/OPDS XML, parses entries with title/author/links/categories  
**Source**: `calibre_opds.py`

### Search Strategy 1: OpenSearch Description
**Given** base feed has `rel="search"` link with opensearchdescription type  
**When** `search()` called  
**Then** fetches OpenSearch XML, resolves `{searchTerms}` template, fetches and filters results  
**Source**: `calibre_opds.py:287-299`

### Search Strategy 2: Direct Template
**Given** search link href contains `{searchTerms}`  
**When** `search()` called  
**Then** resolves template with URL-encoded query  
**Source**: `calibre_opds.py:302-312`

### Search Strategy 3: Common Path Guesses
**Given** strategies 1-2 failed  
**When** `search()` proceeds  
**Then** tries `search?query=`, `search?q=`, `?search=`  
**Source**: `calibre_opds.py:315-338`

### Search Strategy 4: Local BFS Crawl
**Given** all remote strategies failed  
**When** `search()` falls back  
**Then** crawls catalog via navigation links (up to 40 pages), filters by relevance  
**Source**: `calibre_opds.py:341-375`

### Letter Browsing
**Given** a letter value  
**When** `browse_letter()` normalizes  
**Then** uppercased; "0-9"/"NUMERIC" → "#"; "ALL"/"*" returns feed unchanged  
**Source**: `calibre_opds.py:1256-1268`

### Smart Jump for Letters
**Given** sorted feed with pagination  
**When** browsing to distant letter  
**Then** uses English letter distribution weights to estimate target page offset  
**Source**: `calibre_opds.py:1177-1247`

### Relevance Scoring
**Given** entry and query tokens  
**When** `_calculate_match_score()` called  
**Then** scoring: exact title=1000, phrase in title=500, phrase in author=300, phrase in series=200, exact tag=100, token word-boundary in title=50, author=40, series=30, tag=30, summary=15  
**Source**: `calibre_opds.py:1375-1460`

### Authentication
**Given** username and password configured  
**When** HTTP requests made  
**Then** uses HTTP Basic Auth  
**Source**: `calibre_opds.py` (client init)

### Error Handling
**Given** connection failure or HTTP error  
**When** any OPDS operation  
**Then** raises `CalibreOPDSError`; search strategies degrade gracefully through fallbacks  

---

## EPUB3 Exporter

### Package Building
**Given** audio file, chapters with timing, metadata  
**When** `EPUB3PackageBuilder.build()` called  
**Then** produces valid EPUB3 ZIP with: mimetype (first, uncompressed), META-INF/container.xml, content.opf (manifest + spine), nav.xhtml (TOC), per-chapter XHTML + SMIL files, embedded audio  
**Source**: `exporter.py`

### SMIL Time Format
**Given** seconds as float  
**When** `_format_smil_time()` called  
**Then** output is `HH:MM:SS.mmm`; None/negative → `00:00:00.000`  
**Source**: `exporter.py:722-729`

### Chunk ID Normalization
**Given** raw chunk ID  
**When** `_normalize_chunk_id()` called  
**Then** only `[A-Za-z0-9_-]` preserved, max 120 chars; duplicates get `_dup` suffix  
**Source**: `exporter.py:653-660`

### Missing Audio
**Given** audio_path does not exist  
**When** `build()` called  
**Then** raises `FileNotFoundError`  
**Source**: `exporter.py:81-82`

### Audio MIME Detection
**Given** audio filename extension  
**When** `_detect_audio_mime()` called  
**Then** maps: .mp3→audio/mpeg, .m4a/.m4b→audio/mp4, .aac→audio/aac, .wav→audio/wav, .flac→audio/flac, .ogg/.opus→audio/ogg, default→audio/mpeg  
**Source**: `exporter.py:862-873`

### EPUB Mimetype Entry
**Given** ZIP being constructed  
**When** mimetype file added  
**Then** must be first entry, uncompressed, containing `"application/epub+zip"`  

### SMIL-XHTML Cross-References
**Given** SMIL file references XHTML  
**When** `<text src>` fragments are generated  
**Then** each references a valid `<span id>` in the paired XHTML file  

### Book ID Fallback
**Given** no book ID in metadata  
**When** generating content.opf  
**Then** falls back to UUID4  

---

## Error Behaviors Summary

| Module | Error Type | Condition | Behavior |
|--------|-----------|-----------|----------|
| Audiobookshelf | `AudiobookshelfUploadError` | HTTP 4xx/5xx on upload | Status + truncated body |
| Audiobookshelf | `AudiobookshelfUploadError` | Connection failure | Exception message |
| Audiobookshelf | `AudiobookshelfUploadError` | Folder not found | Suggestion message |
| Audiobookshelf | `AudiobookshelfUploadError` | Auth failure (401/403) on search | Immediate raise |
| Audiobookshelf | `ValueError` | Empty base_url or token | At construction time |
| Calibre OPDS | `CalibreOPDSError` | Connection/HTTP/parse errors | Graceful search degradation |
| EPUB3 | `FileNotFoundError` | Audio file missing | At build time |

---

## Invariants

1. **Upload file0 is always audio** — cover and subtitles conditional on config flags + file existence
2. **Folder resolution cached** — first successful resolution persists for client lifetime
3. **Search fallback ordering fixed** — OpenSearch → template → guesses → crawl (OPDS); folder → library → global → search (ABS)
4. **EPUB mimetype first** — ZIP entry ordering required for EPUB validity
5. **SMIL text refs valid** — every `<text src>` fragment targets existing span ID
6. **No retry logic** — neither ABS nor OPDS client retries failed requests
7. **Timeout defaults to 1 hour** — ABS upload timeout accounts for large file uploads
