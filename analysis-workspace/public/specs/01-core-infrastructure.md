# Behavioral Specification: Core Infrastructure

## Module Identity

**Paths**: `abogen/main.py`, `abogen/utils.py`, `abogen/constants.py`, `abogen/is_nvidia.py`, `abogen/check_cuda.py`, `abogen/hf_tracker.py`  
**Role**: Platform abstraction, environment detection, directory resolution, configuration I/O, hardware probing, sleep prevention, encoding detection  
**Boundaries**: Foundation layer with no upward dependencies. All other modules depend on this layer.  
**Does NOT**: Perform TTS, text processing, job management, or UI rendering

---

## Public Interface

### utils.py — Directory Resolution

#### get_user_settings_dir() -> str
**Given** environment and filesystem state  
**When** called  
**Then** resolves settings directory via priority: (1) `ABOGEN_SETTINGS_DIR`, (2) `ABOGEN_DATA`/`ABOGEN_DATA_DIR` + `/settings`, (3) `/data/settings` if `/data` exists (Docker), (4) legacy `~/.config/abogen` if exists (non-Windows), (5) `platformdirs.user_config_dir("abogen")`; result cached via `@lru_cache(maxsize=1)`  
**Source**: `utils.py:get_user_settings_dir`

#### get_user_cache_root() -> str
**Given** environment state  
**When** called  
**Then** resolves cache root; sets `HF_HOME`, `HUGGINGFACE_HUB_CACHE`, `TRANSFORMERS_CACHE`, `XDG_CACHE_HOME` as side effects; cached  
**Source**: `utils.py:get_user_cache_root`

#### get_user_output_root() -> str
**Given** environment state  
**When** called  
**Then** resolves via `ABOGEN_OUTPUT_DIR`/`ABOGEN_OUTPUT_ROOT` or `{cache_root}/outputs`  
**Source**: `utils.py:get_user_output_root`

#### get_user_config_path() -> str
**Given** settings directory resolved  
**When** called  
**Then** returns `{settings_dir}/config.json`  
**Source**: `utils.py:get_user_config_path`

### utils.py — Configuration I/O

#### load_config() -> dict
**Given** config.json may or may not exist  
**When** called  
**Then** reads JSON from `get_user_config_path()`; returns `{}` on missing file or parse error  
**Source**: `utils.py:load_config`

#### save_config(config: dict) -> None
**Given** a config dictionary  
**When** called  
**Then** writes JSON with indent=2 to config path; creates parent dirs if needed  
**Source**: `utils.py:save_config`

### utils.py — GPU Detection

#### get_gpu_acceleration() -> str
**Given** system hardware  
**When** called  
**Then** returns one of: `"cuda"` (NVIDIA), `"mps"` (macOS ARM), `"rocm"` (AMD), `"cpu"` (fallback)  
**Source**: `utils.py:get_gpu_acceleration`

### utils.py — Sleep Prevention

#### prevent_sleep_start() -> None
**Given** OS platform  
**When** called  
**Then** Windows: `SetThreadExecutionState`; macOS: spawns `caffeinate` subprocess; Linux: spawns `systemd-inhibit` subprocess  
**Source**: `utils.py:prevent_sleep_start`

#### prevent_sleep_end() -> None
**Given** sleep prevention active  
**When** called  
**Then** Windows: resets execution state; macOS/Linux: terminates subprocess  
**Source**: `utils.py:prevent_sleep_end`

### utils.py — Encoding Detection

#### detect_encoding(file_path) -> str
**Given** a file path  
**When** called  
**Then** attempts charset detection; defaults to `"utf-8"` on failure  
**Source**: `utils.py:detect_encoding`

### utils.py — Process Creation

#### create_process(cmd, **kwargs) -> subprocess.Popen
**Given** a command list  
**When** called  
**Then** creates subprocess with platform-appropriate flags (Windows: CREATE_NO_WINDOW); pipes stderr  
**Source**: `utils.py:create_process`

### utils.py — Pipeline Loading

#### LoadPipelineThread(Thread)
**Given** need for background model loading  
**When** started  
**Then** loads numpy + KPipeline in background thread; emits `finished` signal with pipeline instance  
**Source**: `utils.py:LoadPipelineThread`

---

## Configuration Contracts

### Environment Variables Read

| Variable | Reader | Default |
|----------|--------|---------|
| `ABOGEN_SETTINGS_DIR` | `get_user_settings_dir()` | platformdirs |
| `ABOGEN_DATA` / `ABOGEN_DATA_DIR` | `get_user_settings_dir()`, `get_user_cache_root()` | None |
| `ABOGEN_TEMP_DIR` | `get_user_cache_root()` | platform cache |
| `ABOGEN_OUTPUT_DIR` / `ABOGEN_OUTPUT_ROOT` | `get_user_output_root()` | `{cache}/outputs` |
| `ABOGEN_INTERNAL_CACHE_ROOT` | `get_internal_cache_root()` | XDG_CACHE_HOME |
| `ABOGEN_ENV_FILE` | `_load_environment()` | find_dotenv |

### Environment Variables Set (side effects)

| Variable | Set By | Value |
|----------|--------|-------|
| `HF_HOME` | `get_user_cache_root()` | `{cache}/huggingface` |
| `HUGGINGFACE_HUB_CACHE` | `get_user_cache_root()` | `{HF_HOME}/hub` |
| `TRANSFORMERS_CACHE` | `get_user_cache_root()` | derived |
| `XDG_CACHE_HOME` | `get_user_cache_root()` | set if not exists |
| `HF_HUB_DISABLE_TELEMETRY` | `main.py` | `"1"` |
| `HF_HUB_OFFLINE` | `main.py` | `"1"` (conditional) |
| `MIOPEN_FIND_MODE` | `main.py` | `"FAST"` |
| `PYTORCH_ENABLE_MPS_FALLBACK` | `main.py` | `"1"` (Darwin/arm) |

---

## Platform Behaviors

### macOS (Darwin)
- GPU: MPS acceleration on Apple Silicon (arm processor)
- Sleep: `caffeinate` subprocess
- Config: Legacy `~/.config/abogen` path checked
- Env: `PYTORCH_ENABLE_MPS_FALLBACK=1` set

### Windows
- GPU: CUDA via gpustat + torch.cuda
- Sleep: `SetThreadExecutionState(ES_CONTINUOUS | ES_SYSTEM_REQUIRED)`
- Process: `CREATE_NO_WINDOW` flag on subprocesses
- Config: platformdirs Windows paths

### Linux
- GPU: CUDA (NVIDIA) or ROCm (AMD)
- Sleep: `systemd-inhibit` subprocess
- Docker: `/data` directory detection for container paths
- Env: `MIOPEN_*` vars for ROCm tuning

---

## State Management

### Mutable Global State
- `_sleep_process`: subprocess handle for sleep prevention
- `_pipeline_cache`: dict for LoadPipelineThread results
- Directory resolution functions use `@lru_cache(maxsize=1)`

### Persisted Files
- `config.json`: user preferences (40+ keys)
- `secret_key`: Flask session key (auto-generated if missing)

### Import-Time Side Effects (main.py)
- Registers `atexit` handler for `prevent_sleep_end`
- Installs SIGINT/SIGTERM signal handlers
- Sets environment variables via `os.environ.setdefault()`

---

## Error Behaviors

### Missing config.json
**Given** config path doesn't exist  
**When** `load_config()` called  
**Then** returns `{}`  
**Source**: `utils.py:load_config`

### Corrupted config.json
**Given** invalid JSON in config file  
**When** `load_config()` catches JSONDecodeError  
**Then** returns `{}`  
**Source**: `utils.py:load_config`

### GPU Detection Failure
**Given** gpustat/torch unavailable or error  
**When** `get_gpu_acceleration()` catches exception  
**Then** falls back to `"cpu"`  
**Source**: `utils.py:get_gpu_acceleration`

### Sleep Prevention Failure
**Given** caffeinate/systemd-inhibit not available  
**When** subprocess fails  
**Then** silently continues (no sleep prevention)  
**Source**: `utils.py:prevent_sleep_start`

### Encoding Detection Failure
**Given** charset detection library unavailable or fails  
**When** `detect_encoding()` catches exception  
**Then** returns `"utf-8"` default  
**Source**: `utils.py:detect_encoding`

---

## Thread Safety

| Function | Thread-Safe | Notes |
|----------|-------------|-------|
| `load_config()` | No | File I/O without lock |
| `save_config()` | No | File I/O without lock |
| `get_user_settings_dir()` | Yes | `@lru_cache` + immutable result |
| `get_user_cache_root()` | Yes | `@lru_cache` + immutable result |
| `get_gpu_acceleration()` | Yes | Stateless computation |
| `prevent_sleep_start/end` | No | Mutates `_sleep_process` global |
| `LoadPipelineThread` | Yes | QThread with signal communication |

---

## Invariants

1. **Directory resolution cached**: Once resolved, directory paths never change within process lifetime
2. **Config always returns dict**: `load_config()` never raises, always returns dict (empty on error)
3. **GPU detection never raises**: Always returns one of 4 valid strings
4. **HF env vars set before model loading**: `get_user_cache_root()` side effects guarantee HF paths configured
5. **Sleep prevention paired**: `prevent_sleep_start()` must be paired with `prevent_sleep_end()` (atexit handler provides safety net)
6. **Constants immutable**: All values in `constants.py` are module-level and never mutated
7. **VERSION fallback**: If `get_version()` fails, returns `"Unknown"`
8. **VOICES_INTERNAL fixed**: 52 voice IDs, never modified at runtime
9. **Platform detection via stdlib**: Uses only `platform.system()`, `platform.processor()`, `sys.platform`
10. **setdefault semantics**: main.py env vars don't overwrite user-set values
