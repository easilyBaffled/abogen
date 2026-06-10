# Chunk 11: integrations-and-export

## Files
- `abogen/integrations/__init__.py`
- `abogen/integrations/audiobookshelf.py`
- `abogen/integrations/calibre_opds.py`
- `abogen/epub3/__init__.py`
- `abogen/epub3/exporter.py`

## Description
External service integrations and export. `audiobookshelf.py` provides AudiobookshelfClient for uploading completed audiobooks (multipart with cover, chapters, subtitles, metadata). `calibre_opds.py` implements an OPDS catalog client for browsing/downloading books from Calibre content servers. `epub3/exporter.py` builds EPUB 3 packages with media overlays (XHTML text chapters synchronized with audio via SMIL documents, OPF package manifest, cover image).

## Internal Relationships
- `audiobookshelf.py` uses `httpx` for HTTP
- `calibre_opds.py` uses `httpx` and XML parsing
- `epub3/exporter.py` imports `ExtractionResult`/`ExtractedChapter` from `text_extractor`
- Web service layer (`service.py`) imports from `audiobookshelf`
- Conversion runner imports `build_epub3_package` from epub3
