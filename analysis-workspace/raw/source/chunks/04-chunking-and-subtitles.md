# Chunk 4: chunking-and-subtitles

## Files
- `abogen/chunking.py`
- `abogen/subtitle_utils.py`

## Description
`chunking.py` splits chapter text into paragraph or sentence-level chunks, applies text normalization, and preserves display-text mapping for subtitle generation. `subtitle_utils.py` parses SRT/VTT/ASS subtitle files, splits text by voice markers, validates voice names, sanitizes filenames for different OS platforms, and provides the core `clean_text`/`calculate_text_length` used throughout.

## Internal Relationships
- `chunking.py` imports from `kokoro_text_normalization` and `normalization_settings`
- `subtitle_utils.py` imports from `utils` (detect_encoding, load_config) and `constants` (SAMPLE_VOICE_TEXTS, VOICES_INTERNAL)
- Both `book_parser.py` and the conversion runners use `subtitle_utils` directly
