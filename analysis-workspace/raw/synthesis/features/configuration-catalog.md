# Abogen Configuration Catalog

Complete reference of every configurable option in the abogen system.

---

## 1. Environment Variables

### 1.1 Application Directory Layout

| Variable | Default | Type | Module | Effect |
|----------|---------|------|--------|--------|
| `ABOGEN_SETTINGS_DIR` | platformdirs user_config_dir("abogen") | str (path) | `utils.py:get_user_settings_dir()` | Overrides settings directory for config.json, voice_profiles.json, speaker_configs.json, overrides.json |
| `ABOGEN_DATA` / `ABOGEN_DATA_DIR` | None | str (path) | `utils.py:get_user_settings_dir()`, `get_user_cache_root()` | Data root; settings at `{value}/settings`, cache at `{value}/cache` |
| `ABOGEN_TEMP_DIR` | platform cache dir | str (path) | `utils.py:get_user_cache_root()` | Overrides cache/temp directory; also configures HF_HOME, XDG_CACHE_HOME, TRANSFORMERS_CACHE |
| `ABOGEN_OUTPUT_DIR` / `ABOGEN_OUTPUT_ROOT` | `{cache_root}/outputs` | str (path) | `utils.py:get_user_output_root()` | Overrides output directory for generated audio and subtitles |
| `ABOGEN_INTERNAL_CACHE_ROOT` | XDG_CACHE_HOME or $HOME/.cache | str (path) | `utils.py:get_internal_cache_root()` | Internal cache marker for application-managed cache |
| `ABOGEN_UPLOAD_ROOT` | platform cache `web/uploads` | str (path) | `webui/app.py:_default_dirs()` | Web UI upload directory |
| `ABOGEN_ENV_FILE` | None | str (path) | `utils.py:_load_environment()` | Custom .env file path; if unset, uses find_dotenv(usecwd=True) |
| `ABOGEN_VOICE_CACHE_DIR` | HuggingFace default cache | str (path) | `voice_cache.py` | Overrides voice weight file cache directory |
| `ABOGEN_QUEUE_STATE_PATH` | None | str (path) | `webui/service.py:_determine_state_path` | Exact file path for queue state persistence |
| `ABOGEN_QUEUE_STATE_DIR` | `{settings_dir}/queue/` | str (path) | `webui/service.py:_determine_state_path` | Directory for queue state file |
| `HOME` | OS default | str (path) | `utils.py` | Home directory fallback |
| `XDG_CACHE_HOME` | `$HOME/.cache` | str (path) | `utils.py` | Cache base; read and set by cache resolution |

### 1.2 Web Server

| Variable | Default | Type | Module | Effect |
|----------|---------|------|--------|--------|
| `ABOGEN_HOST` | `0.0.0.0` | str | `webui/app.py:main()` | Flask server bind address |
| `ABOGEN_PORT` | `8808` | int (as str) | `webui/app.py:main()` | Flask server HTTP port |
| `ABOGEN_DEBUG` | `false` | bool (as str) | `webui/app.py:main()` | Flask debug mode toggle |
| `ABOGEN_SECRET_KEY` | auto-generated persistent file | str | `webui/app.py:_get_secret_key()` | Flask session secret key |

### 1.3 LLM Configuration

| Variable | Default | Type | Module | Effect |
|----------|---------|------|--------|--------|
| `ABOGEN_LLM_BASE_URL` | None | str (URL) | `normalization_settings.py` | OpenAI-compatible API endpoint |
| `ABOGEN_LLM_API_KEY` | None | str | `normalization_settings.py` | API key; "ollama" disables Authorization header |
| `ABOGEN_LLM_MODEL` | None | str | `normalization_settings.py` | Model identifier |
| `ABOGEN_LLM_TIMEOUT` | `30` | float (as str) | `normalization_settings.py` | Request timeout in seconds |
| `ABOGEN_LLM_PROMPT` | None | str | `normalization_settings.py` | Custom normalization prompt template |
| `ABOGEN_LLM_CONTEXT_MODE` | `"sentence"` | str | `normalization_settings.py` | Context scope: sentence, paragraph, document |

### 1.4 NLP / spaCy

| Variable | Default | Type | Module | Effect |
|----------|---------|------|--------|--------|
| `ABOGEN_SPACY_MODEL` | `en_core_web_sm` | str | `normalization_settings.py`, `entity_analysis.py` | spaCy model name override |

### 1.5 Hugging Face Hub (set at startup via setdefault)

| Variable | Default | Type | Module | Effect |
|----------|---------|------|--------|--------|
| `HF_HUB_DISABLE_TELEMETRY` | `"1"` | str | `main.py` | Disables HF Hub telemetry |
| `HF_HUB_ETAG_TIMEOUT` | `"10"` | str | `main.py` | ETag check timeout |
| `HF_HUB_DOWNLOAD_TIMEOUT` | `"10"` | str | `main.py` | Download timeout |
| `HF_HUB_DISABLE_SYMLINKS_WARNING` | `"1"` | str | `main.py` | Suppresses symlink warnings |
| `HF_HUB_OFFLINE` | `"1"` (conditional) | str | `main.py` | Set only if config `disable_kokoro_internet` is True |
| `HF_HOME` | derived from cache root | str (path) | `utils.py:get_user_cache_root()` | Hugging Face home directory |
| `HUGGINGFACE_HUB_CACHE` | derived from HF_HOME | str (path) | `utils.py:get_user_cache_root()` | Hub cache path |
| `TRANSFORMERS_CACHE` | derived from cache root | str (path) | `utils.py:get_user_cache_root()` | Transformers cache path |

### 1.6 GPU / Hardware Tuning

| Variable | Default | Type | Module | Effect |
|----------|---------|------|--------|--------|
| `MIOPEN_FIND_MODE` | `"FAST"` | str | `main.py` | ROCm MIOpen tuning mode |
| `MIOPEN_CONV_PRECISE_ROCM_TUNING` | `"0"` | str | `main.py` | Disables precise ROCm tuning |
| `PYTORCH_ENABLE_MPS_FALLBACK` | `"1"` (Darwin/arm only) | str | `main.py` | MPS operation fallback on Apple Silicon |

### 1.7 Locale Detection (read-only)

| Variable | Default | Type | Module | Effect |
|----------|---------|------|--------|--------|
| `LC_ALL` | OS default | str | `kokoro_text_normalization.py` | US locale detection for date format |
| `LC_TIME` | OS default | str | `kokoro_text_normalization.py` | US locale detection for date format |
| `LANG` | OS default | str | `kokoro_text_normalization.py` | US locale detection for date format |

### 1.8 Docker-Specific

| Variable | Default | Type | Module | Effect |
|----------|---------|------|--------|--------|
| `ABOGEN_UID` | `1000` | int | Docker entrypoint | Container process UID |
| `ABOGEN_GID` | `1000` | int | Docker entrypoint | Container process GID |

---

## 2. Configuration File Settings

Stored in `{settings_dir}/config.json`. Loaded via `load_config()`, saved via `save_config()`.

### 2.1 Web UI Settings (40+ keys)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `language` | str | `"a"` | Default TTS language code |
| `voice` | str | `"af_heart"` | Default Kokoro voice ID |
| `voice_profile` | str | `""` | Named voice profile |
| `custom_voice_formula` | str | `""` | Voice blending formula |
| `speed` | float | `1.0` | Speech speed multiplier |
| `sound_format` | str | `"mp3"` | Output format: wav/mp3/opus/m4b/flac |
| `subtitle_format` | str | `"srt"` | Subtitle format: srt/ass/vtt |
| `subtitle_style` | str | `"srt"` | Subtitle style variant |
| `save_mode` | str | `"timestamped"` | Output naming: timestamped/simple/direct/custom |
| `save_chapters_separately` | bool | `true` | Separate audio per chapter |
| `create_merged_version` | bool | `true` | Also produce merged file |
| `silence_between_chapters` | float | `2.0` | Seconds of silence between chapters |
| `replace_single_newlines` | bool | `true` | Replace single newlines with spaces |
| `use_gpu` | bool | `true` | Enable GPU acceleration |
| `read_chapter_titles` | bool | `true` | Speak chapter titles |
| `read_closing_outro` | bool | `true` | Speak "The end of..." outro |
| `generate_epub3` | bool | `false` | Generate EPUB 3 media overlay |
| `speaker_mode` | str | `"single"` | Speaker mode: single/multi |
| `spacy_sentence_segmentation` | bool | `false` | Use spaCy for sentence splitting |
| `max_words_per_subtitle` | int | `12` | Max words per subtitle entry |
| `subtitle_generation_mode` | str | `"sentence"` | Mode: disabled/line/sentence/etc. |

### 2.2 Normalization Settings

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `normalization_numbers` | bool | `true` | Number-to-word normalization |
| `normalization_titles` | bool | `true` | Title/address abbreviation expansion |
| `normalization_internet_slang` | bool | `true` | Internet abbreviation expansion |
| `normalization_terminal` | bool | `true` | Terminal punctuation enforcement |
| `normalization_caps_quotes` | bool | `true` | ALL-CAPS quote lowercasing |
| `normalization_apostrophe_mode` | str | `"spacy"` | Mode: off/spacy/llm |
| `add_phoneme_hints` | bool | `true` | Phoneme markers for TTS |

### 2.3 LLM Settings

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `llm_base_url` | str | `""` | API base URL |
| `llm_api_key` | str | `""` | Authentication key |
| `llm_model` | str | `""` | Model identifier |
| `llm_timeout` | float | `30.0` | Request timeout |
| `llm_prompt` | str | `""` | Custom prompt template |
| `llm_context_mode` | str | `"sentence"` | Context scope |

### 2.4 Integration Settings

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `audiobookshelf_enabled` | bool | `false` | Enable integration |
| `audiobookshelf_auto_send` | bool | `false` | Auto-upload completed jobs |
| `audiobookshelf_base_url` | str | `""` | Server URL |
| `audiobookshelf_api_token` | str | `""` | Bearer token |
| `audiobookshelf_library_id` | str | `""` | Target library |
| `audiobookshelf_folder_id` | str | `""` | Target folder |
| `audiobookshelf_collection_id` | str | `""` | Optional collection |
| `audiobookshelf_verify_ssl` | bool | `true` | TLS verification |
| `audiobookshelf_send_cover` | bool | `true` | Include cover |
| `audiobookshelf_send_chapters` | bool | `true` | Include chapters |
| `audiobookshelf_send_subtitles` | bool | `true` | Include subtitles |
| `audiobookshelf_timeout` | float | `3600.0` | Upload timeout |
| `calibre_opds_base_url` | str | `""` | OPDS catalog URL |
| `calibre_opds_username` | str | `""` | Basic auth username |
| `calibre_opds_password` | str | `""` | Basic auth password |
| `calibre_opds_timeout` | float | `15.0` | Request timeout |
| `calibre_opds_verify_ssl` | bool | `true` | TLS verification |

---

## 3. Integration Settings

### 3.1 Audiobookshelf (AudiobookshelfConfig dataclass)

| Field | Type | Default | Required | Description |
|-------|------|---------|----------|-------------|
| `base_url` | str | -- | Yes | Server URL; trailing `/api` auto-stripped |
| `api_token` | str | -- | Yes | Bearer token |
| `library_id` | str | None | No | Target library |
| `collection_id` | str | None | No | Collection for upload |
| `folder_id` | str | None | No | Folder resolved by: ID, name, path, tail-segment |
| `verify_ssl` | bool | True | No | TLS verification |
| `send_cover` | bool | True | No | Include cover image |
| `send_chapters` | bool | True | No | Include chapter metadata |
| `send_subtitles` | bool | True | No | Include subtitle files |
| `timeout` | float | 3600.0 | No | Upload timeout (1 hour) |

### 3.2 Calibre OPDS (CalibreOPDSClient)

| Field | Type | Default | Required | Description |
|-------|------|---------|----------|-------------|
| `base_url` | str | -- | Yes | OPDS catalog URL |
| `username` | str | None | No | Basic auth username |
| `password` | str | None | No | Basic auth password |
| `timeout` | float | 15.0 | No | HTTP timeout |
| `verify` | bool | True | No | TLS verification |

---

## 4. Normalization Settings (ApostropheConfig)

| Field | Type | Default | Effect |
|-------|------|---------|--------|
| `contraction_mode` | str | `"expand"` | expand/collapse/keep contractions |
| `possessive_mode` | str | `"keep"` | Singular possessive handling |
| `plural_possessive_mode` | str | `"collapse"` | Plural possessive: remove trailing apostrophe |
| `sibilant_possessive_mode` | str | `"mark"` | Sibilant possessive: insert IZ marker |
| `fantasy_mode` | str | `"keep"` | Fantasy apostrophes: keep/mark/collapse |
| `acronym_possessive_mode` | str | `"keep"` | Acronym possessive handling |
| `decades_mode` | str | `"expand"` | '90s -> nineties |
| `leading_elision_mode` | str | `"expand"` | 'tis -> it is |
| `ambiguous_past_modal_mode` | str | `"contextual"` | Ambiguous 'd: keep/expand_prefer_would/expand_prefer_had/contextual |
| `add_phoneme_hints` | bool | `true` | Emit phoneme markers |
| `protect_cultural_names` | bool | `true` | Preserve O'Brien, D'Angelo |
| `convert_numbers` | bool | `true` | Grouped numbers to words |
| `convert_currency` | bool | `true` | Currency symbols to words |
| `remove_footnotes` | bool | `true` | Strip footnote indicators |
| `number_lang` | str | `"en"` | num2words language |
| `year_pronunciation_mode` | str | `"american"` | off/american |

### Contraction Categories (independently toggleable)

| Category | Default | Covers |
|----------|---------|--------|
| `contraction_aux_be` | `true` | I'm, 're, 's (is) |
| `contraction_aux_have` | `true` | 've, 's (has), 'd (had) |
| `contraction_modal_will` | `true` | 'll |
| `contraction_modal_would` | `true` | 'd (would) |
| `contraction_negation_not` | `true` | All n't contractions |
| `contraction_let_us` | `true` | let's |

---

## 5. Job Parameters

### 5.1 Voice Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `language` | str | `"a"` | Language code |
| `voice` | str | `"af_heart"` | Base voice ID or formula |
| `voice_profile` | str | `""` | Named profile |
| `custom_voice_formula` | str | `""` | Blending formula |
| `speed` | float | `1.0` | Speed multiplier |
| `speaker_mode` | str | `"single"` | single/multi |
| `speakers` | dict | `{}` | Per-speaker assignments |

### 5.2 Output Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `sound_format` | str | `"mp3"` | wav/mp3/opus/m4b/flac |
| `subtitle_format` | str | `"srt"` | srt/ass/vtt |
| `save_mode` | str | `"timestamped"` | Output naming |
| `save_chapters_separately` | bool | `true` | Per-chapter files |
| `create_merged_version` | bool | `true` | Merged file |
| `silence_between_chapters` | float | `2.0` | Inter-chapter silence |
| `generate_epub3` | bool | `false` | EPUB 3 output |

### 5.3 Audio Encoding (hardcoded)

| Format | Codec | Settings |
|--------|-------|----------|
| WAV | PCM | 24kHz, mono, float32 |
| FLAC | FLAC | 24kHz, mono, int |
| MP3 | libmp3lame | VBR quality 2 |
| Opus | libopus | 24kbps |
| M4B | AAC | 192kbps, faststart |

---

## 6. Voice System Configuration

### 6.1 Voice Profiles (`voice_profiles.json`)

```json
{
  "abogen_voice_profiles": {
    "<profile_name>": {
      "provider": "kokoro|supertonic",
      "language": "<lang_code>",
      "voices": [["voice_id", weight], ...],  // kokoro
      "voice": "M1",                          // supertonic
      "total_steps": 5,                       // supertonic [2-15]
      "speed": 1.0                            // supertonic [0.7-2.0]
    }
  }
}
```

### 6.2 Speaker Configs (`speaker_configs.json`)

```json
{
  "abogen_speaker_configs": {
    "<config_name>": {
      "language": "a",
      "languages": ["a", "b"],
      "default_voice": "voice_id",
      "speakers": {
        "<slug>": {
          "id": "slug", "label": "Name",
          "gender": "male|female|unknown",
          "voice": "id", "voice_profile": "name",
          "voice_formula": "formula", "resolved_voice": "final"
        }
      },
      "version": 1
    }
  }
}
```

### 6.3 Pronunciation Overrides (`overrides.json`)

```json
{
  "version": 1,
  "overrides": {
    "<language>": {
      "<normalized_token>": {
        "id": "uuid", "token": "Original",
        "pronunciation": "IPA", "voice": "voice_id",
        "usage_count": 0, "created_at": 0.0, "updated_at": 0.0
      }
    }
  }
}
```

Thread-safe (RLock). Atomic writes (.tmp + shutil.move). Legacy SQLite auto-migrated.

---

## 7. Platform-Specific Configuration

### 7.1 GPU Detection

| Platform | Method | Acceleration |
|----------|--------|-------------|
| macOS (ARM) | processor check + Darwin | MPS |
| Linux/Windows (NVIDIA) | gpustat + torch.cuda | CUDA |
| Linux (AMD) | ROCm env vars | ROCm |
| Any (fallback) | -- | CPU |

### 7.2 Sleep Prevention

| Platform | Mechanism |
|----------|-----------|
| Windows | `SetThreadExecutionState` |
| macOS | `caffeinate` subprocess |
| Linux | `systemd-inhibit` subprocess |

### 7.3 spaCy Model Mapping

| Code | Model | Language |
|------|-------|----------|
| `a` | `en_core_web_sm` | American English |
| `b` | `en_core_web_sm` | British English |
| `e` | `es_core_news_sm` | Spanish |
| `f` | `fr_core_news_sm` | French |
| `i` | `it_core_news_sm` | Italian |
| `p` | `pt_core_news_sm` | Portuguese |
| `z` | `zh_core_web_sm` | Chinese |
| `j` | `ja_core_news_sm` | Japanese |
| `h` | `xx_sent_ud_sm` | Hindi |

### 7.4 State Persistence

Queue state version: 8. Accepts current and version-1. RUNNING/PAUSED jobs reset to PENDING on restart.
