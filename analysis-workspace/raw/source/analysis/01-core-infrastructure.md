# Core Infrastructure Analysis

Chunk: core-infrastructure
Files analyzed:
- abogen/main.py
- abogen/constants.py
- abogen/utils.py
- abogen/hf_tracker.py
- abogen/is_nvidia.py
- abogen/check_cuda.py

---

## File: abogen/main.py

### Purpose
Backwards-compatible entry point that launches the Flask-based web UI. Configures environment variables for Hugging Face Hub, ROCm, and Apple Silicon MPS before delegating to the web UI module.

### Public Functions/Classes

**`main() -> None`**
- Launches the Flask-based web UI by calling `_run_web_ui()` (imported from `abogen.webui.app`).

**`_cleanup_sleep(signum, _frame)`**
- Signal handler for SIGINT and SIGTERM that calls `prevent_sleep_end()` then exits with code 0.

### Configuration Mechanisms

| Mechanism | Source | Behavior |
|-----------|--------|----------|
| `HF_HUB_DISABLE_TELEMETRY` | env var (setdefault) | Set to `"1"` |
| `HF_HUB_ETAG_TIMEOUT` | env var (setdefault) | Set to `"10"` seconds |
| `HF_HUB_DOWNLOAD_TIMEOUT` | env var (setdefault) | Set to `"10"` seconds |
| `HF_HUB_DISABLE_SYMLINKS_WARNING` | env var (setdefault) | Set to `"1"` |
| `HF_HUB_OFFLINE` | env var (conditional set) | Set to `"1"` if config key `disable_kokoro_internet` is True |
| `MIOPEN_FIND_MODE` | env var (setdefault) | Set to `"FAST"` for ROCm tuning |
| `MIOPEN_CONV_PRECISE_ROCM_TUNING` | env var (setdefault) | Set to `"0"` |
| `PYTORCH_ENABLE_MPS_FALLBACK` | env var (conditional setdefault) | Set to `"1"` on Darwin/arm |

### Error Conditions and Edge Cases
- If `load_config()` raises, it returns `{}`, so `disable_kokoro_internet` defaults to `False`.
- `platform.processor() == "arm"` may not match all Apple Silicon identifiers (e.g., empty string in some Docker environments).

### State Management
- Registers `prevent_sleep_end` as an `atexit` handler.
- Installs signal handlers for SIGINT and SIGTERM at module import time.

### External Dependencies
- `abogen.utils` (load_config, prevent_sleep_end)
- `abogen.webui.app` (main as _run_web_ui)
- Standard library: atexit, os, platform, signal, sys

---

## File: abogen/constants.py

### Purpose
Defines application-wide constants: program metadata, supported formats, language mappings, voice lists, UI colors, and subtitle configuration.

### Constants Defined

| Constant | Value/Type | Description |
|----------|------------|-------------|
| `PROGRAM_NAME` | `"abogen"` | Application name |
| `PROGRAM_DESCRIPTION` | string | Human-readable app description |
| `GITHUB_URL` | `"https://github.com/denizsafak/abogen"` | Project URL |
| `VERSION` | dynamic via `get_version()` | Current version string |
| `CHAPTER_OPTIONS_COUNTDOWN` | `30` | Seconds for chapter options countdown |
| `SUBTITLE_FORMATS` | list of 5 tuples | srt, ass_wide, ass_narrow, ass_centered_wide, ass_centered_narrow |
| `LANGUAGE_DESCRIPTIONS` | dict (9 entries) | Single-char codes to language names: a=American English, b=British English, e=Spanish, f=French, h=Hindi, i=Italian, j=Japanese, p=Brazilian Portuguese, z=Mandarin Chinese |
| `SUPPORTED_SOUND_FORMATS` | `["wav", "mp3", "opus", "m4b", "flac"]` | Output audio formats |
| `SUPPORTED_SUBTITLE_FORMATS` | `["srt", "ass", "vtt"]` | Subtitle formats |
| `SUPPORTED_INPUT_FORMATS` | `["epub", "pdf", "txt", "srt", "ass", "vtt"]` | Input formats |
| `SUPPORTED_LANGUAGES_FOR_SUBTITLE_GENERATION` | all 9 language keys | Note: only 'a' and 'b' produce timestamps in Kokoro |
| `VOICES_INTERNAL` | list of 52 voice IDs | Convention: `{lang}{gender}_{name}` |
| `SAMPLE_VOICE_TEXTS` | dict (9 entries) | Sample text per language |
| `COLORS` | dict (20 entries) | UI color palette for dark/light modes |

### Error Conditions
- If `get_version()` fails, VERSION will be `"Unknown"`.

### External Dependencies
- `abogen.utils.get_version`

---

## File: abogen/utils.py

### Purpose
Central utility module: environment loading, encoding detection, resource path resolution, version retrieval, directory management, configuration I/O, GPU detection, sleep prevention, text cleaning, process creation, and pipeline loading.

### Public Functions/Classes

**`_load_environment() -> None`** -- Loads .env files. Checks `ABOGEN_ENV_FILE` first; otherwise uses `find_dotenv(usecwd=True)`. Called at import time.

**`detect_encoding(file_path) -> str`** -- Detects encoding using charset_normalizer or chardet (both optional). Falls back to "utf-8".

**`get_resource_path(package, resource) -> Optional[str]`** -- Three-strategy resource resolution: importlib.resources, relative path from utils.py, subdirectory fallback. Returns None if not found.

**`get_version() -> str`** -- Reads VERSION file. Returns "Unknown" on failure.

**`ensure_directory(path) -> str`** -- Expands/resolves path, creates with makedirs, returns absolute path.

**`get_user_settings_dir() -> str`** (cached) -- Priority: ABOGEN_SETTINGS_DIR > ABOGEN_DATA/settings > /data/settings > legacy ~/.config/abogen > platformdirs.

**`get_user_config_path() -> str`** -- Returns `{settings_dir}/config.json`.

**`get_user_cache_root() -> str`** (cached) -- Priority: ABOGEN_TEMP_DIR > platformdirs > ABOGEN_DATA/cache > /data/cache > /tmp/abogen-cache > /tmp/abogen-cache-{pid}. Also configures HF_HOME, XDG_CACHE_HOME, TRANSFORMERS_CACHE, etc.

**`get_internal_cache_root() -> str`** -- From ABOGEN_INTERNAL_CACHE_ROOT or XDG_CACHE_HOME or $HOME/.cache.

**`get_internal_cache_path(folder=None) -> str`** -- Internal cache base or subfolder.

**`get_user_cache_path(folder=None) -> str`** -- User cache base or subfolder.

**`get_user_output_root() -> str`** (cached) -- ABOGEN_OUTPUT_DIR/ABOGEN_OUTPUT_ROOT or {cache_root}/outputs.

**`get_user_output_path(folder=None) -> str`** -- Output base or subfolder.

**`clean_text(text, *args, **kwargs) -> str`** -- Normalizes whitespace, limits consecutive newlines to 2. Optionally replaces single newlines with spaces (config-driven).

**`create_process(cmd, stdin=None, text=True, capture_output=False) -> subprocess.Popen`** -- Creates subprocess with real-time output streaming via daemon thread. Warns on shell=True. Hides window on Windows.

**`load_config() -> dict`** -- Reads config.json, returns {} on failure.

**`save_config(config) -> None`** -- Writes config.json with indent=2. Silent on failure.

**`calculate_text_length(text) -> int`** -- Strips chapter markers, metadata patterns, newlines; returns character count.

**`get_gpu_acceleration(enabled) -> tuple[str, bool]`** -- Returns (message, is_gpu_enabled). Checks MPS then CUDA.

**`prevent_sleep_start() -> None`** -- Windows: SetThreadExecutionState. Darwin: caffeinate. Linux: systemd-inhibit.

**`prevent_sleep_end() -> None`** -- Reverses sleep prevention for all platforms.

**`load_numpy_kpipeline() -> tuple`** -- Imports and returns (numpy, KPipeline).

**`class LoadPipelineThread(Thread)`** -- Loads numpy+KPipeline in background thread, calls callback(np, KPipeline, error).

### Module-Level Constants
- `default_encoding` = `sys.getfilesystemencoding()`
- `_sleep_procs` = `{"Darwin": None, "Linux": None}`

### Configuration Mechanisms (Environment Variables)

| Variable | Read/Set | Purpose |
|----------|----------|---------|
| `ABOGEN_ENV_FILE` | read | Custom .env path |
| `ABOGEN_SETTINGS_DIR` | read | Settings dir override |
| `ABOGEN_DATA` / `ABOGEN_DATA_DIR` | read | Data root |
| `ABOGEN_TEMP_DIR` | read | Cache dir override |
| `ABOGEN_OUTPUT_DIR` / `ABOGEN_OUTPUT_ROOT` | read | Output dir override |
| `ABOGEN_INTERNAL_CACHE_ROOT` | set | Internal cache marker |
| `HOME` | read/set (fallback) | Home directory |
| `XDG_CACHE_HOME` | read/set | Cache base |
| `HF_HOME` | read/set | Hugging Face home |
| `HUGGINGFACE_HUB_CACHE` | setdefault | HF hub cache |
| `TRANSFORMERS_CACHE` | setdefault | Transformers cache |

### Error Conditions and Edge Cases
- `detect_encoding`: Falls back to utf-8 if no detection library installed.
- `get_resource_path`: Returns None on complete failure.
- `get_user_cache_root`: Multi-level fallback; raises RuntimeError only as final safety (should never trigger).
- `create_process`: Character-by-character reading is CPU-intensive but ensures real-time output.
- `load_config`/`save_config`: Silently swallow all exceptions.
- `prevent_sleep_start` on Linux: Handles missing systemd-inhibit gracefully.
- Import-time side effects: loads .env, suppresses all warnings.

### State Management
- `_sleep_procs`: Mutable dict tracking sleep-prevention subprocesses.
- `lru_cache` on `get_user_settings_dir`, `get_user_cache_root`, `get_user_output_root`: Computed once per process lifetime.
- Global warning suppression at import time.

### External Dependencies
- python-dotenv, chardet (optional), charset_normalizer (optional), platformdirs, torch (optional), kokoro (optional), numpy (optional)

---

## File: abogen/hf_tracker.py

### Purpose
Monkey-patches `huggingface_hub.hf_hub_download` to add download tracking with logging and UI warning signals.

### Public Functions/Classes

**`set_log_callback(cb) -> None`** -- Sets global log_callback for download messages.

**`set_show_warning_signal_emitter(emitter) -> None`** -- Sets global UI signal emitter for download warnings.

**`tracked_hf_hub_download(*args, **kwargs) -> path`** -- Wraps hf_hub_download: tries local_files_only first; if exception (not cached), logs notification then downloads. For .pth files, emits detailed "Downloading model" warning.

### State Management
- Two mutable globals: `log_callback`, `show_warning_signal_emitter`.
- Monkey-patches `huggingface_hub.hf_hub_download` at module import time (permanent process-wide mutation).

### Error Conditions
- Catches all exceptions from local-only download attempt broadly.
- `kwargs.get("repo_id")` / `kwargs.get("filename")` may miss positional args, resulting in `<unknown repo>`/`<unknown file>` in messages.

### External Dependencies
- `huggingface_hub`

---

## File: abogen/is_nvidia.py

### Purpose
Detects NVIDIA GPU presence via gpustat.

### Public Functions/Classes

**`check() -> bool`** -- Queries GPUs via gpustat. Returns True if any GPU name contains: nvidia, rtx, gtx, quadro, tesla, titan, mx. Returns False on gpustat failure or no match.

### Error Conditions
- gpustat failure (no nvidia-smi, no drivers) returns False silently.
- "mx" keyword is broad and could theoretically match non-NVIDIA GPUs.

### External Dependencies
- `gpustat`

---

## File: abogen/check_cuda.py

### Purpose
Standalone script checking CUDA availability with Windows DLL fix for [WinError 1114].

### Public Functions/Classes

**`check_cuda_with_fix() -> None`** -- On Windows: pre-loads torch/lib/c10.dll via ctypes. Then prints torch.cuda.is_available() result. Prints "False" if torch missing.

### Error Conditions
- DLL preload errors silently caught.
- Missing torch prints "False" rather than raising.
- Designed to be called as subprocess (output captured from stdout).

### External Dependencies
- `torch` (optional)
- Standard library: sys, os, platform, ctypes, importlib.util

---

## Cross-Cutting Concerns

### Import-Time Side Effects
1. `utils.py`: Calls `_load_environment()` (loads .env), suppresses all warnings.
2. `hf_tracker.py`: Monkey-patches `huggingface_hub.hf_hub_download` globally.
3. `main.py`: Sets multiple env vars, registers atexit handler, installs signal handlers.
4. `constants.py`: Calls `get_version()` which triggers file I/O.

### Hardware Detection Strategy (layered)
1. `is_nvidia.py`: gpustat (wraps nvidia-smi) for NVIDIA detection
2. `check_cuda.py`: PyTorch torch.cuda.is_available() with Windows DLL fix
3. `utils.py:get_gpu_acceleration()`: Unified check: MPS (Apple Silicon) then CUDA with diagnostics

### Directory Hierarchy (resolved at runtime)
- Settings: ABOGEN_SETTINGS_DIR > ABOGEN_DATA/settings > /data/settings > platform config dir
- Cache: ABOGEN_TEMP_DIR > platform cache dir > ABOGEN_DATA/cache > /data/cache > /tmp/abogen-cache
- Output: ABOGEN_OUTPUT_DIR > {cache_root}/outputs
- Internal cache: ABOGEN_INTERNAL_CACHE_ROOT > XDG_CACHE_HOME > $HOME/.cache
