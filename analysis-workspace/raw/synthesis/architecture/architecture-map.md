# Architecture Map: abogen

## 1. System Layers (bottom up)

### Layer 1: Platform Abstraction and Core Utilities

Responsibility: Environment detection, directory resolution, configuration I/O, hardware probing, sleep prevention, encoding detection.

Constituent modules:
- `abogen/utils.py` -- environment loading, directory hierarchy resolution (`get_user_settings_dir`, `get_user_cache_root`, `get_user_output_root`), config I/O (`load_config`, `save_config`), GPU detection (`get_gpu_acceleration`), sleep prevention (`prevent_sleep_start`/`prevent_sleep_end`), subprocess creation (`create_process`)
- `abogen/constants.py` -- application metadata, format lists, voice lists, language mappings, color palette
- `abogen/is_nvidia.py` -- NVIDIA GPU detection via gpustat
- `abogen/check_cuda.py` -- CUDA availability with Windows DLL fix
- `abogen/hf_tracker.py` -- HuggingFace Hub download tracking (monkey-patches `hf_hub_download`)

---

### Layer 2: Text Processing Pipeline

Responsibility: Input parsing, text normalization, chunking, subtitle generation, NLP analysis.

Constituent modules:
- `abogen/text_extractor.py` -- refactored parser for Web UI (`extract_from_path` -> `ExtractionResult`)
- `abogen/book_parser.py` -- legacy parser for PyQt GUI (`get_book_parser` -> `BaseBookParser` subclasses)
- `abogen/kokoro_text_normalization.py` -- full normalization engine (`normalize_for_pipeline`)
- `abogen/normalization_settings.py` -- settings extraction, `ApostropheConfig` construction
- `abogen/spacy_contraction_resolver.py` -- spaCy-based contraction disambiguation
- `abogen/word_substitution.py` -- marker-preserving word/phrase replacement
- `abogen/heteronym_overrides.py` -- heteronym detection and phonetic replacement tokens
- `abogen/chunking.py` -- text segmentation (`chunk_text` -> list of `Chunk` dataclasses)
- `abogen/subtitle_utils.py` -- subtitle parsing/generation, voice marker parsing, filename sanitization
- `abogen/speaker_analysis.py` -- rule-based speaker detection (`analyze_speakers` -> `SpeakerAnalysis`)
- `abogen/entity_analysis.py` -- spaCy NER extraction (`extract_entities` -> `EntityExtractionResult`)
- `abogen/spacy_utils.py` -- sentence segmentation utilities (`segment_sentences`)

---

### Layer 3: Voice and TTS Engine

Responsibility: Voice management, TTS synthesis, audio encoding.

Constituent modules:
- `abogen/voice_formulas.py` -- formula parsing and tensor blending (`parse_voice_formula`, `get_new_voice`)
- `abogen/voice_profiles.py` -- profile CRUD, persistence to `voice_profiles.json` (`save_profile`, `load_profiles`)
- `abogen/voice_cache.py` -- on-demand voice weight download from HuggingFace (`ensure_voice_assets`, `bootstrap_voice_cache`)
- `abogen/tts_supertonic.py` -- SuperTonic ONNX-based TTS adapter (`SupertonicTTS.__call__` -> `Iterator[SupertonicSegment]`)
- `abogen/llm_client.py` -- stdlib-only OpenAI-compatible client (`generate_completion`, `list_models`)
- `abogen/pronunciation_store.py` -- persistent pronunciation overrides (`save_override`, `load_overrides`)
- `abogen/speaker_configs.py` -- persistent speaker-voice configuration presets (`upsert_config`, `load_configs`)

---

### Layer 4: Integrations and Export

Responsibility: External service communication, output packaging.

Constituent modules:
- `abogen/integrations/audiobookshelf.py` -- `AudiobookshelfClient` (upload, find, delete audiobooks)
- `abogen/integrations/calibre_opds.py` -- `CalibreOPDSClient` (OPDS catalog browsing, search, download)
- `abogen/epub3/exporter.py` -- `EPUB3PackageBuilder` (SMIL-synchronized EPUB 3 generation)

---

### Layer 5: Application Services

Responsibility: Job orchestration, conversion pipeline execution, state persistence.

Constituent modules:
- `abogen/webui/service.py` -- `ConversionService` (job lifecycle, queue, worker thread, state persistence)
- `abogen/webui/conversion_runner.py` -- `run_conversion_job` (complete book-to-audio pipeline)
- `abogen/webui/debug_tts_runner.py` -- debug WAV generation (`run_debug_tts_wavs`)
- `abogen/pyqt/conversion.py` -- `ConversionThread` (desktop pipeline equivalent)

---

### Layer 6: User Interface

Responsibility: User interaction, request handling, presentation.

Constituent modules:
- `abogen/webui/app.py` -- Flask factory (`create_app`)
- `abogen/webui/routes/` -- 7 blueprints (`main_bp`, `jobs_bp`, `settings_bp`, `voices_bp`, `entities_bp`, `books_bp`, `api_bp`)
- `abogen/webui/routes/utils/` -- route utilities (common, entity, epub, form, preview, service, settings, voice)
- `abogen/pyqt/gui.py` -- main desktop widget (`abogen(QWidget)`)
- `abogen/pyqt/book_handler.py` -- `HandlerDialog` (chapter selection)
- `abogen/pyqt/voice_formula_gui.py` -- `VoiceFormulaDialog`
- `abogen/pyqt/queue_manager_gui.py` -- `QueueManager`
- `abogen/pyqt/predownload_gui.py` -- `PreDownloadDialog`

---

## 2. Module Dependency Graph

```
=== Layer 6 (UI) -> Layer 5 (Services) ===
webui/routes/*         -> webui/service (ConversionService)
webui/routes/main_bp   -> webui/routes/utils/form (build_pending_job_from_extraction)
webui/routes/main_bp   -> text_extractor (extract_from_path)
webui/routes/api_bp    -> llm_client, kokoro_text_normalization, voice_profiles
webui/routes/api_bp    -> integrations/audiobookshelf, integrations/calibre_opds
webui/routes/entities  -> pronunciation_store, entity_analysis
webui/routes/voices    -> speaker_configs, constants
webui/routes/settings  -> debug_tts_runner
webui/routes/utils/preview -> tts_supertonic, voice_formulas, kokoro_text_normalization
webui/routes/utils/voice -> speaker_analysis, speaker_configs, voice_profiles
webui/routes/utils/entity -> entity_analysis, pronunciation_store, heteronym_overrides
webui/routes/utils/form -> chunking, constants, voice_profiles, speaker_configs

pyqt/gui               -> book_parser (get_book_parser)
pyqt/gui               -> utils, constants, subtitle_utils, hf_tracker
pyqt/gui               -> voice_profiles (load_profiles)
pyqt/conversion        -> word_substitution, subtitle_utils, spacy_utils
pyqt/book_handler      -> book_parser
pyqt/voice_formula_gui -> voice_profiles, voice_formulas, constants

=== Layer 5 (Services) -> Layer 4 (Integrations) ===
webui/service          -> integrations/audiobookshelf (post-completion hook)
webui/conversion_runner -> epub3/exporter (EPUB3PackageBuilder)

=== Layer 5 (Services) -> Layer 3 (Voice/TTS) ===
webui/conversion_runner -> voice_formulas (get_new_voice)
webui/conversion_runner -> voice_cache (ensure_voice_assets)
webui/conversion_runner -> voice_profiles (load_profiles)
webui/conversion_runner -> tts_supertonic (SupertonicTTS)
webui/conversion_runner -> pronunciation_store (load_overrides, increment_usage)
webui/conversion_runner -> speaker_configs (load_configs)

pyqt/conversion        -> voice_formulas (parse_voice_formula)

=== Layer 5 (Services) -> Layer 2 (Text Processing) ===
webui/conversion_runner -> text_extractor (extract_from_path)
webui/conversion_runner -> kokoro_text_normalization (normalize_for_pipeline)
webui/conversion_runner -> normalization_settings (build_apostrophe_config, get_runtime_settings)
webui/conversion_runner -> chunking (chunk_text)
webui/conversion_runner -> heteronym_overrides

pyqt/conversion        -> subtitle_utils (split_text_by_voice_markers)
pyqt/conversion        -> word_substitution

=== Layer 3 (Voice/TTS) -> Layer 1 (Core) ===
voice_profiles         -> utils (get_user_config_path)
voice_cache            -> constants (VOICES_INTERNAL)
voice_formulas         -> constants (VOICES_INTERNAL)
pronunciation_store    -> utils (get_user_settings_dir, get_internal_cache_path)
speaker_configs        -> utils (get_user_config_path), constants (LANGUAGE_DESCRIPTIONS)
llm_client             -> (stdlib only, no internal deps)

=== Layer 2 (Text Processing) -> Layer 1 (Core) ===
text_extractor         -> utils (detect_encoding, clean_text, calculate_text_length)
book_parser            -> subtitle_utils (clean text utilities)
kokoro_text_normalization -> normalization_settings, spacy_contraction_resolver, llm_client
chunking               -> kokoro_text_normalization (normalize_for_pipeline)
chunking               -> normalization_settings (build_apostrophe_config, get_runtime_settings)
entity_analysis        -> (spaCy external only)
speaker_analysis       -> (no internal dependencies)
pronunciation_store    -> entity_analysis (normalize_token)
subtitle_utils         -> utils (detect_encoding), constants (VOICES_INTERNAL)

=== Layer 1 (Core) internal ===
constants              -> utils (get_version)
main.py                -> utils (load_config, prevent_sleep_end)
main.py                -> webui/app (main)
```

**Circular dependencies**: None detected. Strictly acyclic. Closest potential issue: `pronunciation_store` -> `entity_analysis` (for `normalize_token`) -- narrow, function-level dependency.

---

## 3. Data Flow: Book-to-Audio Pipeline

Web UI pipeline within `run_conversion_job()` (`abogen/webui/conversion_runner.py`):

```
INPUT FILE (.epub/.pdf/.txt/.md)
       |
       v
[1] extract_from_path(path: Path) -> ExtractionResult
    (abogen/text_extractor.py)
    Outputs: ExtractionResult.chapters (List[ExtractedChapter]), .metadata, .combined_text
       |
       v
[2] Chapter Selection & Filtering
    (conversion_runner.py: auto-select or user-specified chapter overrides)
    Filters short/empty chapters; merges user metadata
       |
       v
[3] chunk_text(chapter_index, chapter_title, text, level, speaker_id, voice, ...)
    (abogen/chunking.py)
    Internally calls: _normalize_chunk_text(value) which calls:
      -> get_runtime_settings() (normalization_settings.py)
      -> build_apostrophe_config(settings, base) (normalization_settings.py)
      -> normalize_for_pipeline(value, config, settings) (kokoro_text_normalization.py)
    Outputs: List[Chunk] with .text (normalized) and .display_text (original)
       |
       v
[4] Pronunciation Override Application
    (conversion_runner.py: regex replacement rules from pronunciation_store + heteronym_overrides)
    Compiled rules sorted longest-first, applied to each chunk text
       |
       v
[5] Voice Resolution per chunk
    _resolve_voice(pipeline, voice_spec, use_gpu)
    resolve_voice_target(raw_spec) -- handles "speaker:<name>" and "profile:<name>" references
    get_new_voice(pipeline, formula) -- for formula specs (voice_formulas.py)
    Voice cache lookup/store by "{provider}:{spec}" key
       |
       v
[6] TTS Synthesis
    Kokoro: pipeline.generate(text, voice=resolved_voice, speed=speed, split_pattern=SPLIT_PATTERN)
    SuperTonic: SupertonicTTS.__call__(text, voice=voice, speed=speed, total_steps=steps)
    Outputs: Iterator of audio segments (numpy float32 arrays at 24kHz)
       |
       v
[7] Audio Encoding
    WAV/FLAC: soundfile.SoundFile.write(audio_array) -- direct write
    MP3/Opus/M4B: create_process(ffmpeg ...) with raw f32le PCM piped via stdin
      MP3: libmp3lame VBR quality 2
      Opus: libopus 24kbps
      M4B: AAC 192kbps with faststart
    Chapter silence: zero-valued float32 numpy arrays inserted between chapters
       |
       v
[8] Subtitle Writing (concurrent with audio)
    SubtitleWriter writes SRT (HH:MM:SS,mmm) or ASS (H:MM:SS.cc) entries
    Timestamps derived from TTS segment durations (cumulative offset tracking)
       |
       v
[9] Post-Processing
    M4B: ffmpeg remux with ffmetadata (chapters + tags + cover) + mutagen MP4Chapter atoms
    EPUB3: EPUB3PackageBuilder.build() (abogen/epub3/exporter.py)
    metadata.json: chapter markers, chunk markers, speaker data
       |
       v
[10] Optional Integration
    _maybe_send_to_audiobookshelf(job):
      AudiobookshelfClient.upload_audiobook(audio, metadata, cover, chapters, subtitles)
```

---

## 4. Concurrency Model

### Web UI Concurrency

| Mechanism | Location | Purpose |
|-----------|----------|---------|
| Single daemon worker thread (`abogen-conversion-worker`) | `ConversionService` in `webui/service.py` | Processes jobs sequentially from FIFO queue |
| `threading.RLock` (`_lock`) | `ConversionService` | Protects `_jobs` dict, `_queue` list, all state mutations |
| `threading.Event` (`_wake_event`) | `ConversionService` | Wakes worker when jobs enqueued; blocks with 0.5s timeout when idle |
| `threading.Event` (`_stop_event`) | `ConversionService` | Signals worker to terminate during `shutdown()` |
| Per-job `threading.Event` (`pause_event`) | `Job` dataclass | Initially set; cleared to pause runner at chunk boundaries; set to resume |
| `threading.Lock` (`_CACHE_LOCK`) | `voice_cache.py` | Protects `_CACHED_VOICES` set |
| `threading.Lock` (`_BOOTSTRAP_LOCK`) | `voice_cache.py` | Guards one-time bootstrap flag |
| `threading.RLock` (`_MODEL_LOCK`) | `entity_analysis.py` | Thread-safe spaCy model loading/caching |
| `threading.RLock` (`_DB_LOCK`) | `pronunciation_store.py` | Thread-safe pronunciation override file access |
| `LoadPipelineThread(Thread)` | `utils.py` | Background loading of numpy + KPipeline |
| Preview pipeline cache | `webui/routes/utils/preview.py` | Thread-safe cached KPipeline instances |

### PyQt Desktop Concurrency

| Mechanism | Location | Purpose |
|-----------|----------|---------|
| `ConversionThread(QThread)` | `pyqt/conversion.py` | TTS conversion in background |
| `VoicePreviewThread(QThread)` | `pyqt/conversion.py` | Voice preview generation |
| `PlayAudioThread(QThread)` | `pyqt/conversion.py` | Audio playback via pygame.mixer |
| `_LoaderThread(QThread)` | `pyqt/book_handler.py` | EPUB/PDF parsing in background |
| `PreDownloadWorker(QThread)` | `pyqt/predownload_gui.py` | Model/voice download |
| `StatusCheckWorker(QThread)` | `pyqt/predownload_gui.py` | Per-category status checks |
| Cancel flag (boolean) | `ConversionThread` | Set by main thread; checked at loop boundaries |

**Notable gap**: `spacy_utils.py` module-level `_nlp_cache` dict NOT thread-protected (unlike `entity_analysis.py` which uses `_MODEL_LOCK`).

---

## 5. Storage Architecture

### Configuration Files

| File | Location Resolution | Format | Content |
|------|-------------------|--------|---------|
| `config.json` | `get_user_config_path()` = `{settings_dir}/config.json` | JSON indent=2 | All user preferences (40+ keys) |
| `voice_profiles.json` | Same dir as `config.json` | JSON with `abogen_voice_profiles` wrapper | Voice profile definitions |
| `speaker_configs.json` | Same dir as `config.json` | JSON with `abogen_speaker_configs` wrapper | Named speaker-voice mapping presets |
| `overrides.json` | `get_user_settings_dir()` or `get_internal_cache_path("pronunciations")` | JSON with version field | Per-language pronunciation overrides |
| `secret_key` | `{settings_dir}/secret_key` | Plain text | Flask session signing key |

### Settings Directory Resolution (`get_user_settings_dir`, cached)

1. `ABOGEN_SETTINGS_DIR` env var
2. `ABOGEN_DATA`/`ABOGEN_DATA_DIR` env var + `/settings`
3. `/data/settings` (if `/data` exists -- Docker)
4. Legacy `~/.config/abogen` (non-Windows, if exists)
5. `platformdirs.user_config_dir("abogen")`

### State Files

| File | Location Resolution | Format | Content |
|------|-------------------|--------|---------|
| `queue_state.json` | `ABOGEN_QUEUE_STATE_PATH` > `ABOGEN_QUEUE_STATE_DIR`/`<settings_dir>/queue/` > `get_internal_cache_path("jobs")` | JSON (version 8) | All jobs + queue order; atomic write via `.tmp` + `os.replace` |

### Cache Directories

| Cache | Location Resolution | Content |
|-------|-------------------|---------|
| User cache root | `ABOGEN_TEMP_DIR` > platformdirs > `ABOGEN_DATA/cache` > `/data/cache` > `/tmp/abogen-cache` | Parent for all caches |
| HuggingFace cache | `HF_HOME` (set by `get_user_cache_root` side-effect) | Model weights, voice `.pt` files |
| Voice cache | `ABOGEN_VOICE_CACHE_DIR` > default HF cache structure | `voices/{voice_id}.pt` from `hexgrad/Kokoro-82M` |
| Internal cache | `ABOGEN_INTERNAL_CACHE_ROOT` > `XDG_CACHE_HOME` > `$HOME/.cache` | Subfolders: `pronunciations`, `jobs` |
| Preview cache | `{user_cache}/preview_cache/` | SHA256-keyed WAV files (PyQt only) |
| Web uploads | `ABOGEN_UPLOAD_ROOT` > `{user_cache}/web/uploads` | Uploaded input files |

### Output Directories

| Output | Location Resolution | Content |
|--------|-------------------|---------|
| Output root | `ABOGEN_OUTPUT_DIR`/`ABOGEN_OUTPUT_ROOT` > `{cache_root}/outputs` | Parent for all output |
| Web output | `ABOGEN_OUTPUT_ROOT` > `{user_output_root}/web` | Converted audiobooks |
| Debug output | `{output_root}/debug/{uuid}/` | Debug WAV artifacts + manifest.json |

---

## 6. Dual-UI Architecture

### Shared Core Logic (both UIs use)

| Module | Shared Usage |
|--------|-------------|
| `abogen/utils.py` | Config I/O, directory resolution, GPU detection, sleep prevention |
| `abogen/constants.py` | All constants (voices, formats, languages, colors) |
| `abogen/voice_profiles.py` | Profile CRUD (same JSON file) |
| `abogen/voice_formulas.py` | Formula parsing and tensor blending |
| `abogen/voice_cache.py` | Voice weight downloading |
| `abogen/subtitle_utils.py` | Text cleaning, voice marker parsing, filename sanitization |
| `abogen/hf_tracker.py` | Download tracking |

### Divergent Implementations

| Concern | PyQt Desktop | Web UI |
|---------|-------------|--------|
| Book parsing | `book_parser.py` (class hierarchy, `BaseBookParser` ABC) | `text_extractor.py` (functional, `extract_from_path` -> `ExtractionResult`) |
| Text normalization | `word_substitution.py` only (simple find/replace) | Full `kokoro_text_normalization.py` pipeline + `normalization_settings.py` + `spacy_contraction_resolver.py` |
| Chunking | Implicit (split on chapter/voice markers) | Explicit `chunking.py` with `Chunk` dataclass, paragraph/sentence levels |
| NLP analysis | None | `entity_analysis.py` + `speaker_analysis.py` + `heteronym_overrides.py` |
| LLM integration | None | `llm_client.py` for normalization mode |
| TTS engines | Kokoro only | Kokoro + SuperTonic |
| Job management | In-memory queue (`QueuedItem` list), sequential | Persistent `ConversionService` with state file, pause/resume/cancel |
| Integrations | None | Audiobookshelf upload, Calibre OPDS browsing |
| Export | Audio + subtitles only | Audio + subtitles + EPUB3 with SMIL + metadata.json |
| Pronunciation | None | Full pronunciation store with entity-based overrides |
| Speaker configs | None | `speaker_configs.py` with multi-voice assignment |

### Duplicated Logic

1. **Audio encoding via ffmpeg** -- Both build ffmpeg subprocess commands independently
2. **M4B metadata embedding** -- Both implement ffmpeg remux + mutagen chapter atoms
3. **Chapter marker parsing** -- Both split on `<<CHAPTER_MARKER:Title>>` pattern
4. **Voice marker splitting** -- Both call `split_text_by_voice_markers` from `subtitle_utils.py`
5. **Sleep prevention** -- Both call `prevent_sleep_start`/`prevent_sleep_end`
6. **Voice formula resolution** -- Both use `voice_formulas.py` for tensor blending

### Architecture Note

Web UI is actively-developed frontend with comprehensive features. PyQt GUI is legacy/alternative with simpler pipeline. They share low-level utilities and voice infrastructure but diverge at orchestration layer. Cannot safely run simultaneously against same state files (no cross-process locking on config).
