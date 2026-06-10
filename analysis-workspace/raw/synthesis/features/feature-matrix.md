# Feature Matrix: Abogen

## 1. Input Format Support

| Format | Extensions | Parser Module | Metadata Extracted | Chapter Detection Method | Limitations |
|--------|-----------|---------------|-------------------|--------------------------|-------------|
| EPUB | `.epub` | `text_extractor.py` (Web), `book_parser.py` (Desktop) | title, author, series, series_index, cover image (Web only) | HTML nav (`<nav epub:type="toc">`), NCX fallback, heuristic doc scan | Malformed EPUBs may lose nav; Desktop uses monkey-patches, Web does not |
| PDF | `.pdf` | `text_extractor.py` (Web), `book_parser.py` (Desktop) | title, author, subject, keywords, creator (Web); title, author (Desktop) | PDF TOC (`doc.get_toc()`); no-TOC fallback = single "Pages" parent | Footnote markers `[12]` cleaned; no OCR for scanned PDFs |
| Markdown | `.md` | `text_extractor.py` (Web), `book_parser.py` (Desktop) | YAML frontmatter (Web only) | Splits on `# ` headings (level-1 only) | Only level-1 headings detected as chapters |
| Plain Text | `.txt` | `text_extractor.py` (Web only) | None | `<<CHAPTER_MARKER:Title>>` tags; single chapter if no markers | Not supported in Desktop GUI parser |
| SRT Subtitles | `.srt` | `subtitle_utils.py` | None | Timestamp entries as segments | Blocks with < 3 lines silently skipped |
| ASS Subtitles | `.ass` | `subtitle_utils.py` | None | Dialogue events with timing | Only Dialogue lines processed; Comment lines skipped |
| VTT Subtitles | `.vtt` | `subtitle_utils.py` | None | Timestamp entries as segments | STYLE/NOTE blocks skipped; supports MM:SS short format |

---

## 2. Output Format Support

| Format | Encoder | Container | Metadata Support | Chapter Support | Quality Settings |
|--------|---------|-----------|-----------------|-----------------|-----------------|
| WAV | soundfile (float32, 24kHz mono) | RIFF/WAV | None | Separate files only | Uncompressed; highest fidelity |
| FLAC | soundfile (int, 24kHz mono) | FLAC | None | Separate files only | Lossless compression |
| MP3 | ffmpeg libmp3lame | MPEG Audio Layer 3 | ffmetadata tags | Separate files only | VBR quality 2 |
| Opus | ffmpeg libopus | Ogg/Opus | ffmetadata tags | Separate files only | 24kbps constant |
| M4B | ffmpeg AAC + mutagen | MPEG-4 Audio | ffmetadata tags + cover art + MP4Chapter atoms | Embedded chapter markers (requires merge) | 192kbps AAC, faststart |

### Subtitle Output Formats

| Format | Variants | Time Resolution | Special Features |
|--------|----------|-----------------|-----------------|
| SRT | Standard | Milliseconds (`HH:MM:SS,mmm`) | Index numbers, plain text |
| ASS | Wide, Narrow, Centered Wide, Centered Narrow | Centiseconds (`H:MM:SS.cc`) | Styled dialogue, karaoke highlighting (Desktop) |
| VTT | Standard | Milliseconds (`HH:MM:SS.mmm`) | Web-standard format |

---

## 3. TTS Engine Comparison

| Feature | Kokoro-82M | SuperTonic |
|---------|-----------|------------|
| **Module** | kokoro (KPipeline) | `abogen/tts_supertonic.py` |
| **Model Source** | HuggingFace Hub (`hexgrad/Kokoro-82M`) | supertonic package (auto-download) |
| **GPU Support** | CUDA, MPS (voice blending CPU-only due to `split_with_sizes` bug) | ONNX: CUDAExecutionProvider with CPUExecutionProvider fallback |
| **Voice Mixing** | Yes -- weighted linear interpolation of voice tensors | No -- single voice selection only |
| **Voice Count** | 52 voices across 9 languages | 10 voices (M1-M5, F1-F5) |
| **Languages** | American English, British English, Spanish, French, Hindi, Italian, Japanese, Brazilian Portuguese, Mandarin Chinese | Not language-specific |
| **Quality Settings** | Speed 0.1x-2.0x | Steps [2-15] (default 5), Speed [0.7-2.0] |
| **Streaming** | Iterator of audio segments per generate() call | Iterator of SupertonicSegment per __call__ |
| **Timestamp Tokens** | English only (word-level timing for subtitles) | No timestamp tokens |
| **Max Chunk Length** | Pipeline-managed | 300 characters (configurable) |
| **Retry Logic** | None | Up to 3 retries stripping unsupported characters |
| **UI Availability** | Both Web and Desktop | Web UI only |
| **Sample Rate** | 24,000 Hz | Inferred from audio duration, resampled linearly |

---

## 4. Text Processing Features

| Feature | Module | Configurable | Default Behavior | Dependencies |
|---------|--------|-------------|------------------|-------------|
| Apostrophe/Contraction Expansion | `kokoro_text_normalization.py` | Yes (`normalization_apostrophe_mode`: off/spacy/llm) | spaCy mode with rule-based + contextual disambiguation | spaCy (optional), num2words (optional) |
| Number-to-Words Conversion | `kokoro_text_normalization.py` | Yes (`normalization_numbers`) | Enabled; converts integers, decimals, fractions, ranges, grouped numbers | num2words |
| Date Expansion | `kokoro_text_normalization.py` | Yes (via `normalization_numbers`) | ISO/MDY dates expanded to spoken form; locale-aware US/non-US | None |
| Time Normalization | `kokoro_text_normalization.py` | Yes (via `normalization_numbers`) | Time expressions normalized to spoken form | None |
| Roman Numeral Expansion | `kokoro_text_normalization.py` | Yes (via `normalization_numbers`) | Cardinal for context words (chapter, act); ordinal for name sequences | None |
| Currency Conversion | `kokoro_text_normalization.py` | Yes (`convert_currency` in ApostropheConfig) | Enabled; $, GBP, EUR, JPY with magnitude support | num2words |
| Title/Suffix Expansion | `kokoro_text_normalization.py` | Yes (`normalization_titles`) | Mr./Mrs./Dr./etc. expanded to full words | None |
| Address Abbreviation Expansion | `kokoro_text_normalization.py` | Yes (`normalization_titles`) | St./Rd./Ave./Blvd. expanded | None |
| Internet Slang Expansion | `kokoro_text_normalization.py` | Yes (`normalization_internet_slang`) | pls/plz -> please, etc. | None |
| Terminal Punctuation | `kokoro_text_normalization.py` | Yes (`normalization_terminal`) | Appends "." to segments lacking terminal punctuation | None |
| ALL CAPS Quote Normalization | `kokoro_text_normalization.py` | Yes (`normalization_caps_quotes`) | Lowercases ALL-CAPS text within quotation marks | None |
| Year Pronunciation | `kokoro_text_normalization.py` | Yes (`year_pronunciation_mode`: off/american) | American-style (e.g., 1924 -> "nineteen twenty-four") | None |
| Phoneme Hints (IZ marker) | `kokoro_text_normalization.py` | Yes (`add_phoneme_hints`) | Sibilant possessives get IZ marker for TTS | None |
| Word Substitution | `word_substitution.py` | Yes (user-defined rules) | Marker-preserving find/replace, caps lowering, numeral conversion | num2words (optional) |
| Heteronym Detection | `heteronym_overrides.py` | Yes (part of entity override system) | Replaces ambiguous words (wind, read, tear, close, lead) with phonetic tokens | spaCy (optional) |
| Sentence Splitting | `chunking.py` | Level config (paragraph/sentence) | Regex `[.!?][\s\n]+` with abbreviation merging | spaCy (optional via `spacy_utils.py`) |
| Paragraph Splitting | `chunking.py` | Level config | Split on `(?:\r?\n){2,}` | None |
| LLM-Assisted Normalization | `kokoro_text_normalization.py` | Yes (`normalization_apostrophe_mode: "llm"`) | Sends sentences to LLM for regex replacements | LLM endpoint |
| spaCy Contraction Resolution | `spacy_contraction_resolver.py` | Yes (activated by spaCy mode) | Disambiguates 's (is/has/possessive) and 'd (would/had) | spaCy |
| Footnote Removal | `kokoro_text_normalization.py` | Yes (`remove_footnotes`) | Strips `[12]` patterns from text | None |

---

## 5. UI Feature Comparison (Web vs Desktop)

| Feature | Web UI | Desktop (PyQt6) | Notes |
|---------|--------|-----------------|-------|
| File input | Upload + drag-and-drop | Drag-and-drop + file dialog | Web has wizard-based multi-step flow |
| Direct text input | POST /wizard/text | TextboxDialog with marker insertion | Desktop allows inline editing |
| Voice selection | Dropdown + profile system | Dropdown + VoiceFormulaDialog | Same 52 Kokoro voices |
| Voice mixing/formulas | API + profile CRUD | VoiceMixer widget with sliders | Same `voice_formulas.py` |
| Voice profiles | Import/export/duplicate via API | Import/export via dialog | Same JSON storage |
| Voice preview | WAV synthesis via API endpoint | SHA256-cached WAV via pygame | Desktop caches by hash |
| Speed adjustment | Form field (0.5-2.0) | QSlider (0.1-2.0) | Desktop has wider range |
| Chapter selection | Wizard step with checkboxes | HandlerDialog tree + context menu | Desktop has auto-accept countdown |
| Subtitle generation | SRT/ASS writer in runner | Line/Sentence/+Comma/+Highlighting/N-word | Desktop has more modes |
| Output format selection | Form dropdown | QComboBox | Same formats |
| Job queue | Persistent JSON, pause/resume/cancel/retry | In-memory QueueManager | Web survives restarts |
| Progress tracking | SSE log streaming + htmx polling | QThread -> QProgressBar + ETR | Web has live streaming |
| GPU acceleration | Auto-detected | Checkbox + auto-detect | Same `get_gpu_acceleration()` |
| Theme support | Not implemented | Dark/Light/System | Desktop only |
| Entity analysis (NER) | Full wizard step | Not available | Web only |
| Speaker analysis | Multi-voice with gender detection | Not available | Web only |
| Pronunciation overrides | Persistent store with CRUD API | Not available | Web only |
| Heteronym detection | Integrated in entity step | Not available | Web only |
| Text normalization pipeline | Full pipeline | Word substitution only | Desktop uses simpler system |
| LLM-assisted normalization | Settings + preview | Not available | Web only |
| Audiobookshelf integration | Settings + auto/manual upload | Not available | Web only |
| Calibre OPDS browsing | Find Books page + import | Not available | Web only |
| EPUB 3 export (with SMIL) | Configurable per job | Not available | Web only |
| SuperTonic TTS | Available as provider option | Not available | Web only |
| Debug TTS testing | Settings page with WAV generation | Not available | Web only |
| Pre-download models | Not available | PreDownloadDialog | Desktop only |
| OS sleep prevention | Service lifecycle | prevent_sleep during conversion | Both |
| Update checking | Not available | Settings menu | Desktop only |
| Word substitutions | Via normalization pipeline | Dedicated dialog | Different mechanisms |
| Save mode | timestamped/simple/direct/custom | Project/separate/merge | Web more granular |

---

## 6. Integration Capabilities

| Service | Operations Supported | Auth Method | Error Handling |
|---------|---------------------|-------------|----------------|
| **Audiobookshelf** | Upload audiobook (audio + cover + subtitles + chapters), list libraries, list/resolve folders, find existing items, delete items | Bearer token | `AudiobookshelfUploadError`; 4 endpoint variants tried; no retry |
| **Calibre OPDS** | Fetch feed, search (4 strategies), browse by letter, download resource | HTTP Basic Auth (optional) | `CalibreOPDSError`; graceful degradation through search strategies |
| **LLM (OpenAI-compatible)** | List models, generate completions | API key Bearer header (or "ollama" = skip auth) | `LLMClientError`; timeout configurable (30s default); no retry |
| **HuggingFace Hub** | Download voice assets (`.pt`), download model weights | Env vars for private repos | Monkey-patched download; `resume_download=True`; telemetry disabled |
| **espeak-ng** | Phoneme generation (Kokoro backend) | System dependency | Errors propagate from pipeline |
| **FFmpeg** | Audio encoding, metadata embedding, time-stretch, M4B chapters | System dependency via `static_ffmpeg` | Subprocess errors propagate; M4B failures -> `RuntimeError` |

---

## 7. Subtitle/Timing Features

| Format | Generation Modes | Timing Source | Special Features |
|--------|-----------------|---------------|-----------------|
| **SRT** | Line, Sentence, Sentence+Comma, N-word (Desktop); Sentence-level (Web) | Kokoro timestamp tokens (English); duration-based fallback (non-English) | Index numbers; millisecond precision |
| **ASS** | Same + Sentence+Highlighting (Desktop karaoke) | Same as SRT | Style definitions; wide/narrow/centered variants |
| **VTT** | Parsing only (input) | N/A | STYLE/NOTE blocks; voice tags; MM:SS short format |

### Timing Mechanisms

| Mechanism | Availability | Accuracy |
|-----------|-------------|----------|
| Kokoro word-level timestamps | English only (codes `a`, `b`) | Word-level precision |
| Duration-based fallback | All languages | Approximate sentence/line |
| Input timestamp preservation | Subtitle file inputs | Preserves original |
| SMIL overlays (EPUB 3) | Web UI + EPUB3 export | Chunk-level |
| M4B chapter markers | M4B format | Chapter-level |
| FFmetadata chapters | All compressed formats | Chapter-level |

---

## Implementation Location Cross-Reference

| Behavioral Claim | Implementation | Test Coverage |
|-----------------|----------------|---------------|
| EPUB/PDF/text/markdown to audio | `text_extractor.py`, `book_parser.py` | test_book_parser.py, test_text_extractor.py |
| Kokoro-82M TTS | `conversion_runner.py` (pipeline.generate) | No end-to-end TTS test |
| Voice mixing by weighted addition | `voice_formulas.py:parse_voice_formula()` | test_voice_cache.py |
| M4B with chapters | `conversion_runner.py` (ffmetadata + mutagen) | test_ffmetadata |
| spaCy sentence segmentation | `spacy_utils.py:segment_sentences()` | Conditionally tested |
| Entity analysis with NER | `entity_analysis.py:extract_entities()` | Referenced in utility tests |
| Speaker detection/gender inference | `speaker_analysis.py:analyze_speakers()` | test_speaker_analysis.py |
| Audiobookshelf push | `integrations/audiobookshelf.py` | test_audiobookshelf_client.py |
| Calibre OPDS browse/search | `integrations/calibre_opds.py` | test_calibre_opds.py |
| EPUB 3 with media overlays | `epub3/exporter.py:EPUB3PackageBuilder.build()` | test_epub_exporter.py |
| CPU fallback | `utils.py:get_gpu_acceleration()`, `tts_supertonic.py` | Not directly tested |
| Queue pause/resume/cancel | `webui/service.py:ConversionService` | test_service.py |
| Jobs survive restarts | `webui/service.py:_persist_state/_load_state` | test_service.py |
