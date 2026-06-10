# Behavioral Specification: Voice System

## Module Identity

**Paths**: `abogen/voice_formulas.py`, `abogen/voice_profiles.py`, `abogen/voice_cache.py`, `abogen/tts_supertonic.py`  
**Role**: Voice management (formulas, profiles, caching) + SuperTonic TTS adapter  
**Boundaries**: Manages voice tensor blending, profile persistence, HuggingFace download, and SuperTonic ONNX synthesis. Does NOT handle Kokoro pipeline directly (that's in conversion_runner). Does NOT handle audio encoding.  
**TTS Engines**: Kokoro-82M (primary, via KPipeline in conversion_runner) and SuperTonic (ONNX-based, adapter here)

---

## Public Interface

### voice_formulas.py

| Function | Signature | Returns |
|----------|-----------|---------|
| `get_new_voice(pipeline, formula, use_gpu)` | KPipeline, str, bool -> tensor | Blended voice tensor (always on CPU) |
| `parse_formula_terms(formula)` | str -> List[Tuple[str, float]] | Validated (voice_name, weight) pairs |
| `parse_voice_formula(pipeline, formula)` | KPipeline, str -> tensor | Weighted sum of voice tensors |
| `calculate_sum_from_formula(formula)` | str -> float | Sum of weights via regex |
| `extract_voice_ids(formula)` | str -> List[str] | Voice IDs from formula |

### voice_profiles.py

| Function | Signature | Returns |
|----------|-----------|---------|
| `load_profiles()` | () -> dict | All profiles from JSON file |
| `save_profiles(profiles)` | dict -> None | Writes profiles to JSON |
| `delete_profile(name)` | str -> None | Removes profile by name |
| `duplicate_profile(src, dest)` | str, str -> None | Copies profile |
| `export_profiles(export_path)` | str -> None | Writes all profiles to path |
| `serialize_profiles()` | () -> dict | Alias for load_profiles() |
| `normalize_profile_entry(entry)` | Any -> dict | Normalizes stored profile |

### voice_cache.py

| Function | Signature | Returns |
|----------|-----------|---------|
| `ensure_voice_assets(voices, *, repo_id, cache_dir, on_progress)` | -> (Set[str], Dict[str, str]) | (downloaded, errors) |
| `bootstrap_voice_cache(voices, *, repo_id, cache_dir, on_progress)` | -> (Set[str], Dict[str, str]) | One-time bootstrap |

### tts_supertonic.py

| Class/Function | Description |
|----------------|-------------|
| `SupertonicPipeline` | ONNX TTS adapter with retry and resampling |
| `SupertonicSegment` | Dataclass: graphemes + audio array |
| `DEFAULT_SUPERTONIC_VOICES` | Tuple: ("M1"-"M5", "F1"-"F5") |

---

## Voice Formula System

### Formula Parsing
**Given** a formula string like `"af_heart*0.5 + am_echo*0.3 + bf_emma*0.2"`  
**When** `parse_formula_terms(formula)` is called  
**Then** splits on `+`, splits each term on `*` (first occurrence), validates voice_name against VOICES_INTERNAL, parses weight as float  
**Source**: `voice_formulas.py:20-46`

### Empty Formula
**Given** empty or whitespace-only formula  
**When** `parse_formula_terms()` is called  
**Then** raises `ValueError("Empty voice formula")`  
**Source**: `voice_formulas.py:21-22`

### Missing Weight Separator
**Given** a term without `*`  
**When** parsing formula  
**Then** raises `ValueError("Each component must be in the form voice*weight")`  
**Source**: `voice_formulas.py:30`

### Unknown Voice
**Given** a voice name not in VOICES_INTERNAL  
**When** parsing formula  
**Then** raises `ValueError(f"Unknown voice: {voice_name}")`  
**Source**: `voice_formulas.py:33`

### Non-Positive Weight
**Given** a weight ≤ 0  
**When** parsing formula  
**Then** raises `ValueError(f"Weight for {voice_name} must be positive")`  
**Source**: `voice_formulas.py:39`

### Weight Normalization
**Given** parsed terms with total_weight > 0  
**When** `parse_voice_formula(pipeline, formula)` is called  
**Then** normalizes each weight by dividing by total_weight before applying to tensor  
**Source**: `voice_formulas.py:59`

### Tensor Blending Algorithm
**Given** normalized weights and voice tensors  
**When** blending  
**Then** accumulates via `weighted_sum = sum(normalized_weight_i * tensor_i)` (linear interpolation)  
**Source**: `voice_formulas.py:63-66`

### GPU Disabled for Blending
**Given** `use_gpu=True`  
**When** `get_new_voice()` is called  
**Then** always uses `device="cpu"` due to `split_with_sizes` CUDA bug  
**Source**: `voice_formulas.py:14` (comment explains bug)

### get_new_voice Error Wrapping
**Given** any exception during formula parsing or blending  
**When** `get_new_voice()` catches it  
**Then** raises `ValueError(f"Failed to create voice: {str(e)}")`  
**Source**: `voice_formulas.py:17`

---

## Voice Profile Management

### Profile Storage Location
**Given** application configuration  
**When** profile path is resolved  
**Then** uses `voice_profiles.json` in same directory as `config.json`  
**Source**: `voice_profiles.py:11-13`

### Profile File Format
**Given** profiles are saved  
**When** writing to disk  
**Then** wraps in `{"abogen_voice_profiles": {...}}` JSON with indent=2  
**Source**: `voice_profiles.py:40`

### Load with Wrapper Detection
**Given** a profiles JSON file  
**When** `load_profiles()` reads it  
**Then** checks for `abogen_voice_profiles` key first; falls back to treating entire dict as profiles  
**Source**: `voice_profiles.py:24-28`

### Load File Missing
**Given** profiles file doesn't exist  
**When** `load_profiles()` is called  
**Then** returns empty dict `{}`  
**Source**: `voice_profiles.py:31`

### Load Parse Error
**Given** corrupted JSON in profiles file  
**When** `load_profiles()` raises Exception  
**Then** catches and returns empty dict `{}`  
**Source**: `voice_profiles.py:29-30`

### Profile Normalization (Kokoro)
**Given** a profile entry with `provider="kokoro"` (or absent)  
**When** `normalize_profile_entry()` is called  
**Then** returns `{provider: "kokoro", language: str, voices: [(id, weight), ...]}` ; returns `{}` if no valid voices  
**Source**: `voice_profiles.py:126-133`

### Profile Normalization (SuperTonic)
**Given** a profile entry with `provider="supertonic"`  
**When** `normalize_profile_entry()` is called  
**Then** returns `{provider: "supertonic", language: str, voice: str, total_steps: int[2-15], speed: float[0.7-2.0]}`  
**Source**: `voice_profiles.py:109-124`

### SuperTonic Voice Clamping
**Given** `total_steps` value  
**When** normalizing  
**Then** clamps to range [2, 15], defaults to 5  
**Source**: `voice_profiles.py:76-81`

### SuperTonic Speed Clamping
**Given** `speed` value  
**When** normalizing  
**Then** clamps to range [0.7, 2.0], defaults to 1.0  
**Source**: `voice_profiles.py:84-89`

### Voice Entry Validation
**Given** voice entries in Kokoro profile  
**When** normalizing voices list  
**Then** accepts `[voice_id, weight]` tuples or `{id/voice, weight}` dicts; skips voices not in VOICES_INTERNAL; skips None weights; clamps weight ≥ 0  
**Source**: `voice_profiles.py:136-150+`

---

## Voice Cache

### Cache Directory Resolution
**Given** no explicit `cache_dir` parameter  
**When** `ensure_voice_assets()` resolves cache dir  
**Then** checks `ABOGEN_VOICE_CACHE_DIR` env var; if empty, uses None (HuggingFace default)  
**Source**: `voice_cache.py:59-62`

### Missing huggingface_hub
**Given** `huggingface_hub` import failed  
**When** `ensure_voice_assets()` or `_ensure_single_voice_asset()` is called  
**Then** raises `RuntimeError("huggingface_hub is required to cache voices")`  
**Source**: `voice_cache.py:56-57, 127-128`

### Target Normalization
**Given** `voices=None`  
**When** normalizing targets  
**Then** returns ALL voices from VOICES_INTERNAL  
**Source**: `voice_cache.py:29-30`

### Target Filtering
**Given** a list of voice IDs  
**When** normalizing targets  
**Then** only includes voices that exist in VOICES_INTERNAL; skips empty strings  
**Source**: `voice_cache.py:32-40`

### Local-First Check
**Given** a voice to download  
**When** `_ensure_single_voice_asset()` is called  
**Then** first tries `hf_hub_download(local_files_only=True)`; if `LocalEntryNotFoundError`, downloads with `resume_download=True`  
**Source**: `voice_cache.py:138-144`

### Download Returns Flag
**Given** voice download attempt  
**When** `_ensure_single_voice_asset()` completes  
**Then** returns `False` if already cached locally, `True` if newly downloaded  
**Source**: `voice_cache.py:140, 145`

### Bootstrap Once-Only
**Given** `bootstrap_voice_cache()` called multiple times  
**When** `_BOOTSTRAPPED` is already True  
**Then** returns `(set(), {})` immediately without re-downloading  
**Source**: `voice_cache.py:109-110`

### Download Error Collection
**Given** download fails for a specific voice  
**When** exception is caught  
**Then** records `errors[voice_id] = str(exc)` and continues to next voice  
**Source**: `voice_cache.py:83-85`

### File Path Pattern
**Given** a voice_id to download  
**When** constructing HuggingFace path  
**Then** uses `voices/{voice_id}.pt` from repo `hexgrad/Kokoro-82M`  
**Source**: `voice_cache.py:130`

---

## SuperTonic TTS

### Pipeline Initialization
**Given** SupertonicPipeline instantiated  
**When** `__init__()` runs  
**Then** configures GPU providers BEFORE importing TTS; creates TTS instance with auto_download  
**Source**: `tts_supertonic.py:174-184`

### GPU Provider Selection
**Given** onnxruntime available  
**When** `_configure_supertonic_gpu()` runs  
**Then** checks `CUDAExecutionProvider` availability; always appends `CPUExecutionProvider` as fallback; patches both `supertonic.config` and `supertonic.loader`  
**Source**: `tts_supertonic.py:132-156`

### GPU Config Failure
**Given** exception during GPU configuration  
**When** `_configure_supertonic_gpu()` catches it  
**Then** logs warning and continues (CPU fallback)  
**Source**: `tts_supertonic.py:155-156`

### Text Splitting
**Given** text longer than `max_chunk_length` (default 300)  
**When** `_split_text()` processes it  
**Then** splits on `split_pattern` regex first; hard-splits remaining long chunks at whitespace boundaries (preferring split after position 40)  
**Source**: `tts_supertonic.py:50-83`

### Voice Defaulting
**Given** empty or None voice  
**When** `__call__()` processes voice parameter  
**Then** defaults to `"M1"`  
**Source**: `tts_supertonic.py:195`

### Steps Clamping
**Given** `total_steps` parameter  
**When** `__call__()` processes it  
**Then** clamps to [2, 15]; defaults to instance's `self.total_steps`  
**Source**: `tts_supertonic.py:196-197`

### Speed Clamping
**Given** `speed` parameter  
**When** `__call__()` processes it  
**Then** clamps to [0.7, 2.0]; defaults to 1.0  
**Source**: `tts_supertonic.py:198-199`

### Retry on Unsupported Characters
**Given** SuperTonic raises ValueError with "unsupported character(s)"  
**When** synthesis is attempted  
**Then** parses error for character list, strips those characters, retries up to 3 attempts total  
**Source**: `tts_supertonic.py:211-254`

### Retry Termination (No Progress)
**Given** character removal doesn't change the text  
**When** retry would loop  
**Then** re-raises the original ValueError  
**Source**: `tts_supertonic.py:235-236`

### Retry Termination (Empty Text)
**Given** all characters removed  
**When** text becomes empty after stripping  
**Then** logs warning, breaks loop, skips chunk (no yield)  
**Source**: `tts_supertonic.py:239-244, 256-257`

### Audio Normalization
**Given** raw wav output from SuperTonic  
**When** `_ensure_float32_mono()` processes it  
**Then** converts to float32 numpy array, handles 2D arrays by taking first column, reshapes to 1D  
**Source**: `tts_supertonic.py:25-33`

### Sample Rate Inference and Resampling
**Given** duration value from SuperTonic  
**When** processing audio  
**Then** infers source sample rate as `audio.size / duration`; if between 8kHz-96kHz and differs from target, resamples via linear interpolation  
**Source**: `tts_supertonic.py:262-273`

### Linear Resampling
**Given** source and destination sample rates differ  
**When** `_resample_linear()` is called  
**Then** uses `numpy.interp()` with linearly-spaced points for resampling  
**Source**: `tts_supertonic.py:36-47`

---

## Thread Safety

### Voice Cache Locks
**Given** concurrent access to voice cache  
**When** checking/updating `_CACHED_VOICES`  
**Then** protected by `_CACHE_LOCK` (threading.Lock)  
**Source**: `voice_cache.py:22-23, 68-69, 89-90`

### Bootstrap Lock
**Given** concurrent bootstrap calls  
**When** checking/setting `_BOOTSTRAPPED` flag  
**Then** protected by `_BOOTSTRAP_LOCK` (threading.Lock)  
**Source**: `voice_cache.py:24-25, 108-109`

### Profile Operations Not Thread-Safe
**Given** concurrent profile read/write  
**When** multiple threads access `load_profiles()`/`save_profiles()`  
**Then** NO lock protection — potential race condition on file I/O  
**Source**: `voice_profiles.py` (no threading imports)

### Voice Formula Stateless
**Given** concurrent formula parsing  
**When** multiple threads call `parse_voice_formula()`  
**Then** thread-safe — no mutable module state; `pipeline.load_single_voice()` safety depends on KPipeline implementation  
**Source**: `voice_formulas.py` (no module-level mutable state)

---

## Invariants

1. **Weights always normalized**: Total weights sum to 1.0 after normalization in blending
2. **Blending always on CPU**: Due to CUDA bug, `get_new_voice` forces `device="cpu"`
3. **Profiles always wrapped**: Saved with `abogen_voice_profiles` envelope
4. **Cache is additive**: Voices only added to `_CACHED_VOICES`, never removed
5. **Bootstrap is idempotent**: After first call, returns empty results without side effects
6. **SuperTonic max 3 attempts**: Retry loop bounded at 3 iterations
7. **Speed bounded [0.7, 2.0]**: Both profile normalization and synthesis enforce same range
8. **Steps bounded [2, 15]**: Both profile normalization and synthesis enforce same range
9. **Audio always float32 mono**: `_ensure_float32_mono()` guarantees output shape
10. **Formula requires positive weights**: Zero/negative weights rejected at parse time
