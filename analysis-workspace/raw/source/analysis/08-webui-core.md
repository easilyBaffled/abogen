# Chunk Analysis: webui-core

## File: `abogen/webui/__init__.py`

### Public Functions/Classes
- **`create_app`** (exported via `__all__` and lazy `__getattr__`): Package's sole public export. Loaded lazily on first attribute access.

---

## File: `abogen/webui/app.py`

### Public Functions/Classes
1. **`create_app(config: Optional[dict[str, Any]] = None) -> Flask`** (line 73): Factory that builds and configures Flask application.
2. **`main() -> None`** (line 126): Entry point that creates app and starts dev server.

### Internal Helpers
- **`_SuppressSuccessfulAccessFilter`** (line 17): Logging filter suppressing werkzeug access log entries with HTTP 200/201/204.
- **`_default_dirs() -> tuple[Path, Path]`** (line 34): Resolves upload and output directories from env vars, falling back to platform-specific user cache/output paths.
- **`_get_secret_key() -> str`** (line 53): Retrieves or generates persistent Flask secret key. Checks env var first, then reads/writes from settings file, with random ephemeral fallback.

### App Construction (exact order)
1. Resolve upload/output directories via `_default_dirs()`.
2. Create Flask instance with `static_folder="static"`, `template_folder="templates"`.
3. Set config: `SECRET_KEY`, `UPLOAD_FOLDER`, `OUTPUT_FOLDER`, `MAX_CONTENT_LENGTH` (400 MB).
4. Merge caller-provided `config` dict (overrides base).
5. Build `ConversionService` via `build_service(runner=run_conversion_job, ...)` and store as `app.extensions["conversion_service"]`.
6. Register 7 blueprints: `main_bp` (/), `jobs_bp` (/jobs), `settings_bp` (/settings), `voices_bp` (/voices), `entities_bp` (/overrides), `books_bp` (/find-books), `api_bp` (/api).
7. Register `atexit` handler calling `service.shutdown()`.
8. Attach werkzeug access log filter (once per process via module-level flag).
9. Return app.

### Configuration Options Consumed
| Source | Key | Default |
|--------|-----|---------|
| Env | `ABOGEN_UPLOAD_ROOT` | platform cache `web/uploads` |
| Env | `ABOGEN_OUTPUT_ROOT` | platform output `web` |
| Env | `ABOGEN_SECRET_KEY` | auto-generated persistent file |
| Env | `ABOGEN_HOST` | `0.0.0.0` |
| Env | `ABOGEN_PORT` | `8808` |
| Env | `ABOGEN_DEBUG` | `false` |
| Flask config | `MAX_CONTENT_LENGTH` | 400 MB |

---

## File: `abogen/webui/service.py`

### Public Functions/Classes
1. **`build_service(runner, *, output_root, uploads_root) -> ConversionService`** (line 1606): Factory constructing `ConversionService`.
2. **`ConversionService`** (line 581): Core job management and execution service.
3. **`Job`** (line 98): Dataclass representing single conversion job (all state, progress, results).
4. **`PendingJob`** (line 527): Dataclass for pre-enqueue wizard state.
5. **`JobStatus`** (line 73): Enum: `PENDING`, `RUNNING`, `PAUSED`, `COMPLETED`, `FAILED`, `CANCELLED`.
6. **`JobLog`** (line 83): Dataclass for timestamped log entry.
7. **`JobResult`** (line 90): Dataclass holding output paths: `audio_path`, `subtitle_paths`, `artifacts`, `epub_path`.
8. **`build_audiobookshelf_metadata(job: Job) -> Dict[str, Any]`** (line 370): Constructs Audiobookshelf-compatible metadata from job tags.
9. **`load_audiobookshelf_chapters(job: Job) -> Optional[List[Dict[str, Any]]]`** (line 485): Loads chapter markers from metadata JSON artifact.

### Job Lifecycle

**Creation:**
1. `ConversionService.enqueue(...)` generates UUID hex id.
2. Normalizes metadata tags, chapters, and chunks.
3. Constructs `Job` dataclass with status `PENDING`.
4. Under lock: adds job to `_jobs` dict, appends id to `_queue` list, updates queue positions, signals `_wake_event`.
5. Calls `_ensure_worker()` to guarantee worker thread running.
6. Logs "Job queued".

**Execution:**
1. Worker loop dequeues first item from `_queue`.
2. Checks `cancel_requested`; if true, marks `CANCELLED` immediately.
3. `_run_job(job)`: Sets `pause_event`, clears pause/cancel flags, sets status `RUNNING`, records `started_at`, persists state.
4. Calls `self._runner(job)` (injected `run_conversion_job` function).
5. On success: if `cancel_requested` during run, marks `CANCELLED`; otherwise marks `COMPLETED` and triggers `_post_completion_hooks`.
6. On exception: sets `error`, status `FAILED`, logs traceback (up to 20 lines).
7. In all cases: sets `pause_event`, persists state, updates queue positions.

**Completion Hooks:**
- `_maybe_send_to_audiobookshelf(job)`: Reads audiobookshelf integration config. If enabled + auto_send: uploads audio, cover, subtitles, chapters. Deletes pre-existing items with same title before uploading.

### Thread Management and Concurrency Model
- **Single worker thread** (`abogen-conversion-worker`): daemon thread, processes one job at a time sequentially.
- **`_lock`**: `threading.RLock` protects `_jobs` dict, `_queue` list, and all mutations.
- **`_wake_event`**: `threading.Event` wakes worker when new jobs enqueued.
- **`_stop_event`**: `threading.Event` signals worker to terminate (used in `shutdown()`).
- **`_poll_interval`**: 0.5 seconds default wait timeout when queue empty.
- **Per-job `pause_event`**: `threading.Event` (initially set). When cleared, conversion runner blocks at checkpoints. Resume sets it again.
- Jobs processed **FIFO** from `_queue`.

### State Persistence
- **`STATE_VERSION = 8`**: Current serialization version.
- **State file location** (determined by `_determine_state_path`):
  1. `ABOGEN_QUEUE_STATE_PATH` env var (exact file).
  2. `ABOGEN_QUEUE_STATE_DIR` env var / `<settings_dir>/queue/`.
  3. Fallback: internal cache `jobs/queue_state.json`.
  4. Migrates legacy path if new path does not exist.
- **Serialization**: JSON with `version`, `jobs` array, `queue` array. Written atomically via `.tmp` + `os.replace`. Logs truncated to last 500 entries.
- **Load on startup** (`_load_state`): Accepts current version and version-1. Jobs in `RUNNING`/`PAUSED` state reset to `PENDING` with progress zeroed and re-queued. Worker started if queue non-empty.
- **Persistence trigger**: Called after every state mutation.

### Error Handling and Recovery
- Persistence failures silently swallowed.
- Voice cache bootstrap failures logged as warnings but don't prevent service start.
- Deserialization errors for individual jobs caught and job skipped.
- Running/Paused jobs found on reload reset to PENDING (crash recovery).
- Audiobookshelf hook failures don't change job status (logged as errors).

### Pause/Resume/Cancel
- **Cancel**: Sets `cancel_requested=True`, clears pause, sets `pause_event`. If PENDING, immediately marks CANCELLED and removes from queue.
- **Pause**: Sets `pause_requested=True`. If PENDING, removes from queue and marks PAUSED immediately. If RUNNING, runner checks at each chunk boundary.
- **Resume**: Clears `pause_requested`/`paused`, sets `pause_event`. If paused-before-start, re-inserts at front of queue. If paused-while-running, sets status back to RUNNING.

### Retry and Delete
- **`retry(job_id)`**: Only for COMPLETED/FAILED/CANCELLED jobs. Creates new job via `enqueue` with same parameters. Removes old job. Returns new Job.
- **`delete(job_id)`**: Only for non-RUNNING jobs. Removes from `_jobs` and `_queue`.
- **`clear_finished(statuses)`**: Bulk removes all jobs in given terminal statuses.

---

## File: `abogen/webui/conversion_runner.py`

### Public Functions/Classes
1. **`run_conversion_job(job: Job) -> None`** (line 1517): Main conversion entry point, invoked by service worker.

### Constants
- `SPLIT_PATTERN = r"\n+"` (line 54): Regex for splitting text into segments for TTS.
- `SAMPLE_RATE = 24000` (line 55): Audio sample rate.

### Conversion Pipeline (exact order)
1. **Normalization settings**: Load runtime settings; apply job-level `normalization_overrides`; build `ApostropheConfig`.
2. **Validate LLM config** if apostrophe mode is "llm".
3. **Load voice profiles** from saved profile store.
4. **Extract text** from input file (`extract_from_path`).
5. **Infer file type** (epub/markdown/pdf/txt).
6. **Compile pronunciation overrides**: Merge manual + pronunciation + speaker overrides. Build regex-based replacement rules sorted longest-first.
7. **Compile heteronym sentence rules**: Pattern-based sentence-level replacements.
8. **Auto-select chapters** (if no explicit chapter overrides): Filter out short/empty chapters based on file-type thresholds.
9. **Apply chapter overrides** (if job has explicit chapters): Match by index/title, apply enable/disable, merge metadata.
10. **Merge metadata**: Combine extracted document metadata with user overrides.
11. **Calculate total characters**.
12. **Apply newline policy** (`replace_single_newlines`).
13. **Prepare output directory** based on `save_mode`.
14. **Prepare project layout** (timestamped folder; optionally with audio/subtitles/metadata subdirs).
15. **Force merge for m4b format** (m4b cannot be split into chapters without merging).
16. **Open merged audio sink** (WAV/FLAC via soundfile, or MP3/Opus/M4B via ffmpeg pipe).
17. **Create subtitle writer** (SRT or ASS format).
18. **Resolve base voice** and pre-cache it.
19. **Initialize voice cache** (download required voice assets).
20. **Build title intro text** (from metadata: series, title, subtitle, author).
21. **Iterate chapters**:
    - Emit book intro (first chapter only).
    - Emit spoken chapter heading.
    - Strip duplicate heading from body text.
    - Normalize uppercase chapter opening.
    - If chunk-level data exists: iterate chunks with per-chunk voice resolution.
    - Otherwise: emit full chapter body text.
    - Insert silence between chapters.
    - Record chapter markers (start/end times).
22. **Emit outro** ("The end of...") using metadata if `read_closing_outro` enabled.
23. **Record pronunciation override usage** in persistent store.
24. **Write metadata.json** artifact (chapters, chunks, speakers, metadata).
25. **Generate EPUB 3 package** if `generate_epub3` enabled.
26. **Set progress to 1.0**.

**Finally block** (always runs):
- Close audio sink and subtitle writer.
- Clear TTS pipelines, set to None, call `gc.collect()`, and `torch.cuda.empty_cache()` if available.
- **Embed M4B metadata** via ffmpeg (chapters, cover art, metadata tags, movflags) + mutagen (MP4Chapter atoms).

### Audio Processing
**WAV/FLAC**: Direct write via `soundfile.SoundFile` (24kHz, mono, float32 for WAV, int for FLAC).

**Compressed formats (MP3/Opus/M4B)**: Audio piped as raw `f32le` PCM to ffmpeg subprocess via stdin:
- MP3: libmp3lame, VBR quality 2
- Opus: libopus, 24kbps
- M4B: AAC, 192kbps, faststart+use_metadata_tags

**Concatenation**: All chapter audio written sequentially to same sink (no intermediate files for merged output). Silence = zero-valued float32 numpy arrays.

**Chapter-level output**: Separate `AudioSink` per chapter (opened/closed in per-chapter `ExitStack`).

**M4B metadata embedding** (post-conversion): ffmpeg remuxes with ffmetadata (chapters + tags + cover art as attached_pic), then mutagen writes MP4Chapter atoms.

### Voice Resolution
- **`_resolve_voice(pipeline, voice_spec, use_gpu)`**: If spec contains `*` (formula), calls `get_new_voice` to mix voice tensors. Otherwise returns spec as-is.
- **`resolve_voice_target(raw_spec)`**: Checks for `speaker:<name>` or `profile:<name>` references. Resolves via loaded profiles. Infers provider (kokoro vs supertonic) from spec.
- **Voice cache** (`Dict[str, Any]`): Memoizes resolved voice tensors/IDs by `{provider}:{spec}` key within single job execution.

### Subtitle Generation
- **`SubtitleWriter`**: Writes SRT or ASS format. SRT: index, timestamps, text. ASS: full header with style, dialogue events.
- **Timestamps**: HH:MM:SS,mmm (SRT) or H:MM:SS.cc (ASS).

### Error Handling
- `_JobCancelled` exception: Raised by `_make_canceller()` closure; caught to set status CANCELLED.
- General exceptions: Set `job.error`, status FAILED, log traceback + diagnostic context.
- LLM normalization failures (`LLMClientError`): Re-raised to fail job.
- ffmpeg metadata embedding failures: Re-raised as `RuntimeError`.
- CUDA initialization failures: Falls back to CPU with warning log.

---

## File: `abogen/webui/debug_tts_runner.py`

### Public Functions/Classes
1. **`run_debug_tts_wavs(*, output_root, settings, epub_path=None) -> Dict[str, Any]`** (line 84): Generates debug WAV artifacts for TTS quality testing.
2. **`DebugWavArtifact`** (line 24): Frozen dataclass holding artifact metadata (label, filename, code, text).

### Exact Behavior
1. Creates `output_root/debug/<random_uuid>/` directory.
2. If no `epub_path` provided, generates debug EPUB via `build_debug_epub`.
3. Extracts text from EPUB.
4. Parses marker codes from text using regex.
5. Cross-checks found codes against expected codes; raises `RuntimeError` if any missing.
6. Resolves language (with alias mapping for ISO codes to Kokoro short codes).
7. Resolves voice spec (handles `profile:<name>` references).
8. Ensures voice assets cached.
9. Loads Kokoro pipeline.
10. For each sample case: synthesizes code identifier (letter-by-letter) + sample text, concatenates with 1s silence gap.
11. Concatenates all cases with 0.35s silence between them into `overall.wav`.
12. Writes `manifest.json` with run_id, epub path, artifact list, sample_rate.
13. Returns manifest dict.

---

## Cross-Cutting Observations

### Concurrency Summary
Entire system uses **single-threaded sequential worker** model. Only one job runs at a time. Flask request threads interact with service via `_lock` (RLock) for reads/writes to job dict and queue.

### State Persistence Summary
Jobs survive restarts because `_persist_state()` writes full job list + queue to JSON after every mutation. On reload, RUNNING/PAUSED jobs reset to PENDING and re-queued.

### Memory Management
Runner explicitly clears pipeline references, calls `gc.collect()`, and empties CUDA cache after each job completes. Prevents GPU memory accumulation across sequential jobs.
