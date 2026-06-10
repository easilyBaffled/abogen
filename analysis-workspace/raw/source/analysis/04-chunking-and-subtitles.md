# Chunking and Subtitles Analysis

Chunk: chunking-and-subtitles
Files analyzed:
- `abogen/chunking.py`
- `abogen/subtitle_utils.py`

---

## File: abogen/chunking.py

### Chunk Dataclass (frozen)

Fields:
- `id: str` -- Unique identifier (e.g., `chap0001_p0002`)
- `chapter_index: int` -- Chapter this belongs to
- `chunk_index: int` -- Ordinal position within chapter
- `level: ChunkLevel` -- "paragraph" or "sentence"
- `text: str` -- Whitespace-normalized text
- `speaker_id: str = "narrator"` -- Speaker identifier
- `voice: Optional[str] = None` -- TTS voice name
- `voice_profile: Optional[str] = None` -- Voice profile reference
- `voice_formula: Optional[str] = None` -- Voice blending formula
- `display_text: Optional[str] = None` -- Original text for subtitle display

### Main Entry Point: chunk_text()

Parameters: chapter_index, chapter_title, text, level, speaker_id="narrator", voice=None, voice_profile=None, voice_formula=None, chunk_prefix=None

Behavior:
- `chunk_prefix` defaults to `f"chap{chapter_index:04d}"`
- **Paragraph level**: Split on double-newlines, ID format `{prefix}_p{para_index:04d}`
- **Sentence level**: For each paragraph, split into sentences. ID format: `{prefix}_p{para_index:04d}_s{sent_local_index:04d}`
- Both levels include `normalized_text` (pipeline-normalized) and `original_text`
- Both call `_attach_display_text` at end to reconcile display text with source

### Text Splitting Strategies

**Paragraph**: Splits on `(?:\r?\n){2,}` (two or more consecutive newlines). Each segment stripped; empty skipped.

**Sentence**: Uses `(?<!\b[A-Z])[.!?][\s\n]+` -- splits on sentence-ending punctuation followed by whitespace, with negative lookbehind for single uppercase letters (initials).

**Abbreviation merging**: When a sentence ends with a known abbreviation (Mr, Mrs, Dr, Prof, Rev, Sr, Jr, St, Gen, Lt, Col, Sgt, Capt, vs, etc.), it merges with the next sentence to prevent false splits.

### Display Text Reconstruction

`_attach_display_text(source, chunks)`: Locates each chunk's text within original source using flexible whitespace matching. Cursor-based sequential matching with fallback to searching from position 0.

### Integration with Normalization

`_normalize_chunk_text(value)`:
1. Gets runtime normalization settings
2. Builds apostrophe config
3. Calls `normalize_for_pipeline(value, config, settings)` for full normalization
4. Applies whitespace collapse

### Edge Cases
- Empty text: falls back to `[text.strip()]`
- No sentences found: paragraph treated as single sentence
- Display text pattern miss: chunk retains existing text as fallback

---

## File: abogen/subtitle_utils.py

### Text Cleaning Functions

**`clean_subtitle_text(text)`**: Removes METADATA_*, CHAPTER_MARKER, and VOICE tags.

**`calculate_text_length(text)`**: Character count after removing markers and newlines.

**`clean_text(text)`**: Normalizes whitespace, standardizes multiple newlines to exactly two, optionally replaces single newlines with spaces (config-driven, default True).

### Subtitle File Parsing

| Format | Function | Time Resolution | Special Features |
|--------|----------|----------------|-----------------|
| SRT | `parse_srt_file` | Milliseconds (`,` separator) | Index numbers, HTML tag removal |
| VTT | `parse_vtt_file` | Milliseconds (`.` separator) | STYLE/NOTE blocks, voice tags, MM:SS short format |
| ASS/SSA | `parse_ass_file` | Centiseconds (`.` separator) | Format-line column mapping, styling overrides, \N/\n newlines |
| Timestamp-text | `parse_timestamp_text_file` | Milliseconds | Free-form text between timestamps |

All parsers return `List[(start_seconds, end_seconds, text)]` (or end=None for last entry).

All parsers use `detect_encoding` for charset auto-detection.

### Timestamp Detection

`detect_timestamps_in_text(file_path)`: Checks first 50 non-empty lines. Returns True if >= 2 timestamp-only lines exist AND they constitute >5% of checked lines. Supports `HH:MM:SS` and `HH:MM:SS,ms` formats.

### Voice Marker System

**`split_text_by_voice_markers(text, default_voice)`**: Core voice-marker parsing.

Returns: `(segments, last_voice, valid_count, invalid_count)`

Behavior:
1. Finds all `<<VOICE:name>>` markers in text
2. Text before first marker uses `default_voice`
3. Validates each voice via `validate_voice_name`
4. Valid voices normalized to canonical lowercase from VOICES_INTERNAL
5. Invalid voices silently skipped (counted but previous voice persists)
6. Formula voices (`af_heart*0.5 + am_echo*0.5`) parsed per-component
7. Returns `last_voice_used` for persistence across chapter boundaries

**`validate_voice_name(voice_name)`**: Validates against VOICES_INTERNAL. Supports formula parsing (split on `+`, then on `*`). Case-insensitive. Returns `(is_valid, invalid_name)`.

### Filename Sanitization

`sanitize_name_for_os(name, is_folder=True)`: Platform-aware sanitization.
- **Windows**: Removes illegal chars, handles reserved names (CON, PRN, etc.)
- **macOS**: Removes colons and control chars
- **Linux**: Removes `/` and null bytes
- All: truncates to 255, falls back to "audiobook" if empty

### Regex Patterns (All Pre-compiled)

Key patterns:
- `_METADATA_TAG_PATTERN`: `<<METADATA_*:*>>` 
- `_CHAPTER_MARKER_PATTERN`: `<<CHAPTER_MARKER:*>>`
- `_VOICE_MARKER_SEARCH_PATTERN`: captures inside `<<VOICE:(...)>>`
- `_SENTENCE_SPLIT_REGEX`: `(?<!\b[A-Z])[.!?][\s\n]+`
- `_PARAGRAPH_SPLIT_REGEX`: `(?:\r?\n){2,}`
- Platform-specific sanitization patterns for Windows/macOS/Linux

### Edge Cases
- SRT blocks with < 3 lines: silently skipped
- VTT with optional cue identifiers: handles both layouts
- ASS Comment lines: explicitly skipped (only Dialogue processed)
- Empty sanitized names: falls back to "audiobook"
- Text before first timestamp: assigned start time 0.0
- No timestamps in file: entire content = single entry at 0.0
