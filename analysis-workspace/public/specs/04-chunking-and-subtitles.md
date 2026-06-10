# Behavioral Specification: Chunking and Subtitles

## Module Identity

**Paths**: `abogen/chunking.py`, `abogen/subtitle_utils.py`  
**Role**: Text segmentation into TTS-ready chunks + subtitle file parsing/generation + voice marker handling + filename sanitization  
**Boundaries**: Receives pre-extracted chapter text, produces ordered chunk sequences. Parses subtitle files for re-synthesis input. Does NOT perform TTS, audio encoding, or timing generation.  
**Dependencies**: `kokoro_text_normalization` (normalization pipeline), `normalization_settings`, `utils` (detect_encoding, load_config), `constants` (VOICES_INTERNAL, SAMPLE_VOICE_TEXTS)

---

## Public Interface: chunking.py

### Chunk (frozen dataclass)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | str | required | Unique chunk identifier |
| `chapter_index` | int | required | Parent chapter index |
| `chunk_index` | int | required | Sequential index within chapter |
| `level` | ChunkLevel | required | "paragraph" or "sentence" |
| `text` | str | required | Normalized whitespace text |
| `speaker_id` | str | "narrator" | Speaker assignment |
| `voice` | Optional[str] | None | Voice ID |
| `voice_profile` | Optional[str] | None | Profile name |
| `voice_formula` | Optional[str] | None | Formula string |
| `display_text` | Optional[str] | None | Original text for display |

### chunk_text(...) -> List[Dict[str, object]]

**Signature**: `chunk_text(*, chapter_index, chapter_title, text, level, speaker_id="narrator", voice=None, voice_profile=None, voice_formula=None, chunk_prefix=None)`

### build_chunks_for_chapters(chapters, *, level, speaker_id="narrator") -> List[Dict[str, object]]

Iterates chapter dicts, calls `chunk_text` for each with text content.

---

## Public Interface: subtitle_utils.py

| Function | Signature | Returns |
|----------|-----------|---------|
| `clean_subtitle_text(text)` | str -> str | Text with markers/metadata stripped |
| `calculate_text_length(text)` | str -> int | Character count excluding markers/newlines |
| `clean_text(text)` | str -> str | Whitespace-normalized, optionally newline-collapsed |
| `parse_srt_file(file_path)` | path -> List[(float, float, str)] | Parsed SRT entries |
| `parse_vtt_file(file_path)` | path -> List[(float, float, str)] | Parsed VTT entries |
| `parse_ass_file(file_path)` | path -> List[(float, float, str)] | Parsed ASS entries |
| `detect_timestamps_in_text(file_path)` | path -> bool | Whether file has inline timestamps |
| `parse_timestamp_text_file(file_path)` | path -> List[(float, float\|None, str)] | Timestamp-text entries |
| `get_sample_voice_text(lang_code)` | str -> str | Sample text for voice preview |
| `sanitize_name_for_os(name, is_folder=True)` | str -> str | OS-safe filename |
| `validate_voice_name(voice_name)` | str -> (bool, Optional[str]) | Validity + first invalid voice |
| `split_text_by_voice_markers(text, default_voice)` | str, str -> (list, str, int, int) | Segments + last voice + counts |

---

## Chunking Behaviors

### Paragraph Level Splitting
**Given** text with double-newline separators and `level="paragraph"`  
**When** `chunk_text()` is called  
**Then** splits on `(?:\r?\n){2,}`, strips each segment, skips empty results  
**Source**: `chunking.py:53-57` (`_iter_paragraphs`)

### Sentence Level Splitting
**Given** text with sentence-ending punctuation and `level="sentence"`  
**When** `chunk_text()` is called  
**Then** first splits into paragraphs, then splits each on `[.!?][\s\n]+` (not after single uppercase letter), yields (normalized, raw) pairs  
**Source**: `chunking.py:60-74` (`_iter_sentences`)

### Abbreviation Merging
**Given** a sentence ending with an abbreviation (Mr., Mrs., Dr., Prof., etc.)  
**When** sentence splitting occurs  
**Then** merges that sentence with the next one to avoid incorrect splits  
**Source**: `chunking.py:107-108` (checks `_ABBREVIATION_END_RE`)

### Paragraph Fallback on Empty
**Given** text that yields no paragraphs after splitting  
**When** `chunk_text()` is called with `level="paragraph"`  
**Then** uses `[text.strip()]` as single-element fallback  
**Source**: `chunking.py:138`

### Sentence Fallback on Empty
**Given** a paragraph that yields no sentences after splitting  
**When** processing at sentence level  
**Then** uses `[(normalized_para, paragraph)]` as single-sentence fallback  
**Source**: `chunking.py:169`

### Chunk ID Format (Paragraph)
**Given** `chunk_prefix` or default `chap{index:04d}`  
**When** paragraph chunks are generated  
**Then** IDs follow pattern `{prefix}_p{para_index:04d}`  
**Source**: `chunking.py:143`

### Chunk ID Format (Sentence)
**Given** paragraph and sentence indices  
**When** sentence chunks are generated  
**Then** IDs follow pattern `{prefix}_p{para_index:04d}_s{sent_local_index:04d}`  
**Source**: `chunking.py:176`

### Normalization Integration
**Given** a raw chunk text  
**When** `_normalize_chunk_text()` is called  
**Then** calls `get_runtime_settings()` -> `build_apostrophe_config()` -> `normalize_for_pipeline()` -> whitespace normalization  
**Source**: `chunking.py:81-87`

### Display Text Attachment
**Given** generated chunks and original source text  
**When** `_attach_display_text()` runs  
**Then** searches source text for each chunk's normalized text using regex pattern matching, assigns matched span as `display_text` and `original_text`  
**Source**: `chunking.py:224-242`

### Display Text Search with Cursor
**Given** chunks processed in order  
**When** searching for display text spans  
**Then** searches forward from cursor position; if not found, retries from position 0  
**Source**: `chunking.py:232-234`

### Whitespace Normalization
**Given** any text segment  
**When** `_normalize_whitespace()` is called  
**Then** collapses all whitespace sequences to single space, strips edges  
**Source**: `chunking.py:77-78`

### build_chunks_for_chapters Delegation
**Given** an iterable of chapter dicts with `text`, `voice`, `voice_profile`, `voice_formula`, `id`, `title` keys  
**When** `build_chunks_for_chapters()` is called  
**Then** iterates chapters, skips non-dicts and empty text, calls `chunk_text()` for each, extends results  
**Source**: `chunking.py:245-275`

---

## Subtitle Parsing

### SRT Parsing
**Given** an SRT file with index/timestamp/text blocks  
**When** `parse_srt_file(file_path)` is called  
**Then** detects encoding, splits on double-newline, requires ≥3 lines per block, parses `HH:MM:SS,mmm --> HH:MM:SS,mmm` timestamps, strips HTML tags, cleans markers  
**Source**: `subtitle_utils.py:77-136`

### SRT Block Minimum Lines
**Given** an SRT block with fewer than 3 lines  
**When** parsing  
**Then** block is silently skipped  
**Source**: `subtitle_utils.py:101`

### VTT Parsing
**Given** a WebVTT file  
**When** `parse_vtt_file(file_path)` is called  
**Then** removes WEBVTT header, STYLE blocks, NOTE blocks; supports both `HH:MM:SS.mmm` and `MM:SS.mmm` formats; strips HTML and voice tags  
**Source**: `subtitle_utils.py:139-222`

### VTT Short Timestamp Format
**Given** a VTT timestamp with only `MM:SS.mmm` (2 parts)  
**When** parsing time  
**Then** treats as minutes:seconds (no hours)  
**Source**: `subtitle_utils.py:202-205`

### VTT Optional Identifier
**Given** a VTT block where first line is NOT a timestamp  
**When** parsing  
**Then** checks second line for `-->` separator; uses line after timestamp as text start  
**Source**: `subtitle_utils.py:175-181`

### ASS Parsing
**Given** an ASS/SSA subtitle file  
**When** `parse_ass_file(file_path)` is called  
**Then** finds `[Events]` section, parses Format line for column positions, processes Dialogue lines, skips Comment lines, uses centisecond timestamps (`H:MM:SS.CS`), removes `{styling}` tags, converts `\N`/`\n` to newlines  
**Source**: `subtitle_utils.py:317-396`

### ASS Comment Lines Skipped
**Given** an ASS line starting with "Comment:"  
**When** in Events section  
**Then** line is skipped entirely  
**Source**: `subtitle_utils.py:354-355`

### ASS Section Boundary
**Given** a new section header (starting with `[`) after `[Events]`  
**When** parsing  
**Then** stops processing events  
**Source**: `subtitle_utils.py:342-343`

### Timestamp-Text File Detection
**Given** a text file  
**When** `detect_timestamps_in_text(file_path)` is called  
**Then** checks first 50 non-empty lines for timestamp-only lines (`HH:MM:SS` or `HH:MM:SS,ms`); returns True if ≥2 timestamp lines AND >5% of total lines  
**Source**: `subtitle_utils.py:225-243`

### Timestamp-Text File Parsing
**Given** a file with inline timestamps  
**When** `parse_timestamp_text_file(file_path)` is called  
**Then** splits by timestamp regex, associates text after each timestamp with that time; text before first timestamp gets time 0.0; end times derived from next entry's start  
**Source**: `subtitle_utils.py:247-314`

### Encoding Detection
**Given** any subtitle file path  
**When** parsing is initiated  
**Then** calls `detect_encoding(file_path)` and opens with that encoding + `errors="replace"`  
**Source**: `subtitle_utils.py:87-88, 149-150, 251-252, 328-329`

---

## Voice Marker Handling

### Voice Marker Pattern
**Given** text containing `<<VOICE:name>>` patterns  
**When** `split_text_by_voice_markers(text, default_voice)` is called  
**Then** regex `<<VOICE:(.*?)>>` identifies markers  
**Source**: `subtitle_utils.py:19, 523`

### No Markers Present
**Given** text with no `<<VOICE:...>>` markers  
**When** splitting  
**Then** returns `[(default_voice, text)]`, last_voice=default_voice, valid=0, invalid=0  
**Source**: `subtitle_utils.py:525-527`

### Text Before First Marker
**Given** text exists before the first voice marker  
**When** splitting  
**Then** assigns that text to default_voice as first segment  
**Source**: `subtitle_utils.py:535-539`

### Voice Validation
**Given** a voice marker with a name  
**When** validating  
**Then** checks against VOICES_INTERNAL (case-insensitive); formulas validated per-voice in each term  
**Source**: `subtitle_utils.py:468-501`

### Invalid Voice Handling
**Given** a voice marker with an unrecognized voice name  
**When** splitting  
**Then** increments invalid_markers count, keeps previous voice for that segment  
**Source**: `subtitle_utils.py:576-578`

### Voice Formula Normalization
**Given** a valid formula like `"af_heart*0.5 + am_echo*0.5"`  
**When** processing voice markers  
**Then** normalizes each voice part to canonical lowercase from VOICES_INTERNAL, reconstructs formula with ` + ` separators  
**Source**: `subtitle_utils.py:553-567`

### Voice Persistence Across Segments
**Given** multiple voice markers  
**When** splitting completes  
**Then** returns `last_voice_used` as second element for persistence across chapters  
**Source**: `subtitle_utils.py:583`

---

## Text Cleaning

### clean_subtitle_text
**Given** text with `<<METADATA_*:*>>`, `<<CHAPTER_MARKER:*>>`, `<<VOICE:*>>` tags  
**When** `clean_subtitle_text(text)` is called  
**Then** removes all three marker types, strips result  
**Source**: `subtitle_utils.py:35-41`

### calculate_text_length
**Given** text with markers and newlines  
**When** `calculate_text_length(text)` is called  
**Then** removes markers, removes newlines, strips, returns `len()`  
**Source**: `subtitle_utils.py:44-54`

### clean_text Whitespace Handling
**Given** text with irregular whitespace  
**When** `clean_text(text)` is called  
**Then** collapses intra-line whitespace to single space per line, normalizes multiple newlines to exactly two (paragraph breaks), optionally replaces single newlines with spaces (config-driven: `replace_single_newlines`)  
**Source**: `subtitle_utils.py:57-74`

### clean_text Config Loading
**Given** any call to `clean_text()`  
**When** processing  
**Then** loads `replace_single_newlines` from `load_config()` on each call (not cached)  
**Source**: `subtitle_utils.py:61-62`

---

## Filename Sanitization

### sanitize_name_for_os Platform Dispatch
**Given** a filename string  
**When** `sanitize_name_for_os(name, is_folder=True)` is called  
**Then** dispatches to platform-specific rules based on `platform.system()`  
**Source**: `subtitle_utils.py:417`

### Windows Sanitization
**Given** platform is Windows  
**When** sanitizing  
**Then** replaces `<>:"/\|?*` with `_`, removes control chars 0-31, strips trailing `. `, prepends `_` to reserved names (CON, PRN, AUX, NUL, COM1-9, LPT1-9)  
**Source**: `subtitle_utils.py:419-435`

### macOS Sanitization
**Given** platform is Darwin  
**When** sanitizing  
**Then** replaces `:` with `_`, removes control chars, prepends `_` if folder starts with `.`  
**Source**: `subtitle_utils.py:436-445`

### Linux Sanitization
**Given** platform is Linux  
**When** sanitizing  
**Then** replaces `/` and null with `_`, removes control chars 1-31, prepends `_` if folder starts with `.`  
**Source**: `subtitle_utils.py:446-455`

### Empty Name Fallback
**Given** name is empty or becomes empty/whitespace after sanitization  
**When** sanitizing  
**Then** returns `"audiobook"`  
**Source**: `subtitle_utils.py:414-415, 458-459`

### Length Limit
**Given** sanitized name exceeds 255 characters  
**When** sanitizing  
**Then** truncates to 255 and strips trailing `. `  
**Source**: `subtitle_utils.py:462-463`

---

## Configuration

### Chunking Level
**Given** `level` parameter  
**When** chunk_text is called  
**Then** "paragraph" splits on double-newline only; "sentence" splits paragraphs into sentences  
**Source**: `chunking.py:137, 162`

### Normalization Settings
**Given** chunking occurs  
**When** `_normalize_chunk_text()` runs  
**Then** uses `get_runtime_settings()` to load current config-based settings; constructs `ApostropheConfig` from defaults + settings  
**Source**: `chunking.py:82-86`

### Newline Replacement Config
**Given** `replace_single_newlines` in config.json  
**When** `clean_text()` runs  
**Then** if True (default), single newlines become spaces; if False, preserved  
**Source**: `subtitle_utils.py:71-73`

### Chunk Prefix Override
**Given** `chunk_prefix` parameter  
**When** chunk_text is called  
**Then** uses provided prefix instead of default `chap{index:04d}`  
**Source**: `chunking.py:134`

### Voice Propagation
**Given** voice/voice_profile/voice_formula parameters  
**When** chunks are generated  
**Then** all chunks inherit the same voice settings from parameters  
**Source**: `chunking.py:149-153, 183-187`

---

## Error Behaviors

### Empty Text Input
**Given** empty or whitespace-only text  
**When** `chunk_text()` is called  
**Then** paragraph fallback yields `[text.strip()]` which is `""`, whitespace normalization produces empty string, chunk is skipped (line 142: `if not normalized: continue`)  
**Source**: `chunking.py:138, 141-142`

### Malformed SRT Blocks
**Given** SRT block with fewer than 3 lines or unparseable timestamps  
**When** `parse_srt_file()` processes block  
**Then** block silently skipped via `continue`  
**Source**: `subtitle_utils.py:101, 110-111, 133-134`

### Encoding Detection Failure
**Given** file with unknown encoding  
**When** subtitle parser opens file  
**Then** uses `errors="replace"` to substitute undecodable bytes with replacement character  
**Source**: `subtitle_utils.py:88, 150, 252, 329`

### Invalid Voice in Marker
**Given** `<<VOICE:nonexistent_voice>>` marker  
**When** `split_text_by_voice_markers()` processes it  
**Then** validation fails, `invalid_markers` incremented, previous voice retained for segment  
**Source**: `subtitle_utils.py:576-578`

### Timestamp Detection Exception
**Given** any exception during `detect_timestamps_in_text()`  
**When** error occurs  
**Then** returns `False` (bare except)  
**Source**: `subtitle_utils.py:243-244`

---

## Invariants

1. **Chunk ordering**: `chunk_index` is sequential within a chapter (0-based, gapless)
2. **Paragraph ordering**: Paragraphs maintain source order
3. **Sentence ordering**: Sentences maintain source order within paragraphs, paragraphs maintain source order
4. **Non-empty chunks**: Empty/whitespace-only segments are always skipped
5. **ID uniqueness**: Chunk IDs are unique within a chapter due to index-based naming
6. **Timestamp monotonicity**: Subtitle parsers preserve source ordering (entries appended in file order)
7. **Text completeness**: Every non-empty text between markers becomes a segment
8. **Voice persistence**: `last_voice_used` tracks state for cross-chapter continuity
9. **Normalization applied once**: `_normalize_chunk_text()` runs per chunk, stores result in `normalized_text` field
10. **Display text traceability**: `display_text` maps back to exact span in source text
11. **Frozen Chunk dataclass**: Chunk instances are immutable after creation (output dicts are mutable copies via `as_dict()`)
