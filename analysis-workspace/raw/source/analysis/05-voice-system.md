# Voice System Analysis

Chunk: voice-system
Files analyzed:
- `abogen/voice_formulas.py` (82 lines)
- `abogen/voice_profiles.py` (230 lines)
- `abogen/voice_cache.py` (146 lines)
- `abogen/tts_supertonic.py` (276 lines)

---

## 1. Voice Formula Parsing Syntax and Blending Algorithm

Source: `abogen/voice_formulas.py`

### Formula Syntax

The formula format is a `+`-delimited list of `voice*weight` terms:

```
<voice_id>*<weight> + <voice_id>*<weight> [+ ...]
```

Example: `af_bella*0.7 + am_adam*0.3`

Each segment MUST contain a `*` character; otherwise a ValueError is raised with message "Each component must be in the form voice*weight".

Voice names are validated against the `VOICES_INTERNAL` constant list. Unknown voices raise `ValueError("Unknown voice: {voice_name}")`.

Weights must be positive floats (> 0); zero or negative weights are rejected.

### Blending Algorithm

`parse_voice_formula(pipeline, formula)` implements blending:

1. Parse all terms via `parse_formula_terms(formula)`.
2. Compute `total_weight = sum(weight for _, weight in terms)`.
3. For each voice, compute `normalized_weight = weight / total_weight`.
4. Load the voice tensor from the pipeline via `pipeline.load_single_voice(voice_name)`.
5. Accumulate: `weighted_sum += normalized_weight * voice_tensor`.
6. Return the blended tensor.

This is a simple weighted linear interpolation in tensor space with normalization to sum=1.

### GPU Loading (Disabled)

`get_new_voice` calls `parse_voice_formula`, then moves the tensor to a device. GPU ("cuda") is commented out due to a known issue: `split_with_sizes(): argument 'split_sizes' (position 2)` error. The device is hardcoded to "cpu".

### Utility Functions

- `calculate_sum_from_formula(formula)` uses regex `r"\* *([\d.]+)"` to extract and sum all weights (without validation against known voices).
- `extract_voice_ids(formula)` returns the list of voice identifier strings from a formula.

---

## 2. Voice Profile CRUD Operations and Storage Format

Source: `abogen/voice_profiles.py`

### Storage Location

Profiles are stored in `voice_profiles.json` in the same directory as the user config file. The config directory is resolved via `get_user_config_path()` which returns `<settings_dir>/config.json`, so profiles live at `<settings_dir>/voice_profiles.json`.

The settings directory is determined (in order of priority) by:
1. `ABOGEN_SETTINGS_DIR` environment variable
2. `ABOGEN_DATA`/`ABOGEN_DATA_DIR` env var + `/settings`
3. `/data/settings` (if `/data` directory exists)
4. Legacy `~/.config/abogen` (non-Windows, if it exists)
5. `platformdirs.user_config_dir("abogen")`

### Storage Format

The JSON file always uses a top-level wrapper key:

```json
{
  "abogen_voice_profiles": {
    "<profile_name>": { ... }
  }
}
```

Each profile entry supports two providers:

**Kokoro provider** (legacy and current):
```json
{
  "provider": "kokoro",
  "language": "a",
  "voices": [["af_bella", 0.7], ["am_adam", 0.3]]
}
```

**SuperTonic provider**:
```json
{
  "provider": "supertonic",
  "language": "a",
  "voice": "M1",
  "total_steps": 5,
  "speed": 1.0
}
```

### CRUD Operations

| Operation | Function | Behavior |
|-----------|----------|----------|
| Create/Update | `save_profile(name, *, language, voices)` | Validates name non-empty, normalizes voices, persists as kokoro provider |
| Read all | `load_profiles()` | Returns dict from JSON, handles `abogen_voice_profiles` wrapper or bare dict fallback |
| Delete | `delete_profile(name)` / `remove_profile(name)` | Removes key from dict and persists |
| Duplicate | `duplicate_profile(src, dest)` | Copies entry under new key |
| Export | `export_profiles(export_path)` / `export_profiles_payload(names)` | Writes all profiles to a file path or returns a filtered subset dict |
| Import | `import_profiles_data(data, *, replace_existing=False)` | Merges incoming profiles; skips duplicates unless `replace_existing=True`; returns list of added/updated names |
| Serialize | `serialize_profiles()` | Returns raw profiles dict |

### Normalization

`normalize_profile_entry(entry)` handles backwards compatibility:
- Determines provider (defaults to "kokoro" if missing or invalid; only "kokoro" and "supertonic" recognized).
- For supertonic: normalizes voice name (upper-cased, must be in `DEFAULT_SUPERTONIC_VOICES`, else defaults to "M1"), clamps `total_steps` to [2, 15] (default 5), clamps `speed` to [0.7, 2.0] (default 1.0).
- For kokoro: normalizes voice entries list.

`_normalize_voice_entries(entries)` accepts items as dicts (`{id/voice, weight}`) or tuples/lists. Each voice must exist in `VOICES_INTERNAL`, weight must be a positive float.

---

## 3. Cache Management

Source: `abogen/voice_cache.py`

### Architecture

The cache system downloads Kokoro voice weight files (`.pt` tensors) from HuggingFace Hub on demand.

### Download Source

Default repository: `hexgrad/Kokoro-82M`. Files are stored at path `voices/{voice_id}.pt` within the HF hub cache structure.

### Cache Directory Resolution

Priority:
1. Explicit `cache_dir` parameter
2. `ABOGEN_VOICE_CACHE_DIR` environment variable
3. Default HuggingFace cache (managed by `hf_hub_download`)

### Download Strategy

`_ensure_single_voice_asset(voice_id, repo_id, cache_dir)`:
1. First attempts `hf_hub_download(local_files_only=True, ...)` -- checks if file already exists locally.
2. If `LocalEntryNotFoundError` is raised, downloads with `resume_download=True`.
3. Returns `True` if a download occurred, `False` if already cached.

### In-Memory Tracking

Module-level state:
- `_CACHED_VOICES: Set[str]` -- tracks voice IDs known to be locally cached.
- `_CACHE_LOCK: threading.Lock()` -- protects concurrent access.
- `_BOOTSTRAPPED: bool` -- ensures one-time bootstrap.
- `_BOOTSTRAP_LOCK: threading.Lock()` -- protects bootstrap flag.

### Public API

`ensure_voice_assets(voices, *, repo_id, cache_dir, on_progress)`:
- Normalizes target voices (defaults to ALL `VOICES_INTERNAL` if `None` passed).
- Iterates over voices not yet in `_CACHED_VOICES`.
- Returns `(downloaded: Set[str], errors: Dict[str, str])`.

`bootstrap_voice_cache(...)`:
- Wraps `ensure_voice_assets` with a one-time guard.
- Subsequent calls are no-ops returning empty structures.

### Invalidation

There is NO explicit cache invalidation mechanism. Once a voice ID is added to `_CACHED_VOICES`, it remains for the process lifetime.

---

## 4. SuperTonic TTS Adapter

Source: `abogen/tts_supertonic.py`

### Initialization

Parameters:
- `sample_rate: int` -- target output sample rate.
- `auto_download: bool = True` -- passed to `supertonic.TTS`.
- `total_steps: int = 5` -- default inference steps (quality knob).
- `max_chunk_length: int = 300` -- maximum characters per text chunk.

### GPU Detection and Configuration

`_configure_supertonic_gpu()`:
1. Imports `onnxruntime` and queries `get_available_providers()`.
2. Builds provider list: prefers `CUDAExecutionProvider`, always appends `CPUExecutionProvider`.
3. Explicitly skips `TensorrtExecutionProvider` (may list as available but fail at runtime).
4. Patches `supertonic.config.DEFAULT_ONNX_PROVIDERS` and `supertonic.loader.DEFAULT_ONNX_PROVIDERS`.
5. On any exception, logs a warning and falls back to CPU.

### Synthesis Loop

`__call__(text, *, voice, speed, split_pattern, total_steps)` returns `Iterator[SupertonicSegment]`:

1. Voice normalization: defaults to "M1" if empty.
2. Parameter clamping: steps in [2, 15], speed in [0.7, 2.0].
3. Voice style loading: `self._tts.get_voice_style(voice_name=voice_name)`.
4. Text splitting via `_split_text`.
5. Per-chunk synthesis with retry loop (max 3 attempts) for unsupported characters.
6. Audio post-processing: float32 mono conversion, sample rate inference from duration, linear resampling.
7. Yields `SupertonicSegment(graphemes=chunk_to_speak, audio=audio)`.

### Text Splitting

`_split_text(text, split_pattern, max_chunk_length)`:
- Uses `re.split(split_pattern, text)` if pattern provided; falls back on regex error.
- Hard-splits long parts at whitespace boundaries (minimum 40 chars before seeking split).

---

## 5. All Voice Identifiers and Naming Conventions

### Kokoro Voices (VOICES_INTERNAL)

56 voice identifiers following pattern `{language_code}{gender}_{name}`:

| Prefix | Language | Gender | Count |
|--------|----------|--------|-------|
| `af_` | American English | Female | 11 |
| `am_` | American English | Male | 9 |
| `bf_` | British English | Female | 4 |
| `bm_` | British English | Male | 4 |
| `ef_` | Spanish | Female | 1 |
| `em_` | Spanish | Male | 2 |
| `ff_` | French | Female | 1 |
| `hf_` | Hindi | Female | 2 |
| `hm_` | Hindi | Male | 2 |
| `if_` | Italian | Female | 1 |
| `im_` | Italian | Male | 1 |
| `jf_` | Japanese | Female | 4 |
| `jm_` | Japanese | Male | 1 |
| `pf_` | Portuguese | Female | 1 |
| `pm_` | Portuguese | Male | 2 |
| `zf_` | Chinese | Female | 4 |
| `zm_` | Chinese | Male | 4 |

### SuperTonic Voices

`DEFAULT_SUPERTONIC_VOICES = ("M1", "M2", "M3", "M4", "M5", "F1", "F2", "F3", "F4", "F5")`

Naming convention: `{M|F}{1-5}` -- gender (Male/Female) + numeric index. Always uppercase.

---

## 6. Error Handling

### Missing/Unknown Voices

| Location | Behavior |
|----------|----------|
| `voice_formulas.py` | `ValueError("Unknown voice: {voice_name}")` |
| `voice_formulas.py` | `ValueError("Empty voice formula")` |
| `voice_formulas.py` | `ValueError("Each component must be in the form voice*weight")` |
| `voice_profiles.py` | Silently skips unknown voices during normalization |
| `voice_profiles.py` | Normalizes unknown SuperTonic voice to "M1" default |
| `voice_cache.py` | Records error in dict and continues |

### GPU Fallback

| Location | Behavior |
|----------|----------|
| `voice_formulas.py` | GPU explicitly disabled; always CPU due to known bug |
| `tts_supertonic.py` | CUDA attempted; falls back to CPU on exception |

### Unsupported Characters (SuperTonic)

Recovery mechanism:
1. Regex extracts unsupported chars from error message.
2. Strips offending chars and retries (up to 3 times).
3. If chunk becomes empty, drops it with warning.
4. If retries exhausted or unrecognized error format, re-raises.

---

## Key Architectural Observations

1. Two TTS providers coexist: Kokoro (tensor-based voice blending) and SuperTonic (ONNX-based, single voice selection).
2. Voice blending is CPU-only due to a known CUDA bug with `split_with_sizes`.
3. No cache invalidation exists -- voice downloads tracked in-memory per process only.
4. SuperTonic has sophisticated unsupported-character retry logic (strip and retry up to 3x).
5. Linear resampling trades audio quality for simplicity and zero external dependencies.
