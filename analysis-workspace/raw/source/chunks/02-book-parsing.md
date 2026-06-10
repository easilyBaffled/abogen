# Chunk 2: book-parsing

## Files
- `abogen/book_parser.py`
- `abogen/text_extractor.py`

## Description
Two parallel implementations of book format parsing (EPUB, PDF, Markdown, TXT). `book_parser.py` is the legacy parser (used by PyQt GUI) with in-place navigation structure building. `text_extractor.py` is the refactored version (used by web UI) producing `ExtractionResult` dataclass with typed chapters, metadata, and cover image extraction.

## Internal Relationships
- Both import from `abogen.utils` (detect_encoding, clean_text/calculate_text_length)
- `text_extractor.py` also has its own `clean_text` import from utils
- `book_parser.py` imports `clean_text` and `calculate_text_length` from `subtitle_utils`
- Web UI conversion runner imports exclusively from `text_extractor`
- PyQt conversion imports from `book_parser`
