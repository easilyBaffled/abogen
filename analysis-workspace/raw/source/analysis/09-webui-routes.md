# Comprehensive Analysis: WebUI Routes (Chunk 9)

## 1. Package Structure

`abogen/webui/routes/__init__.py` exports 7 blueprints:
- `main_bp` (from `main.py`)
- `jobs_bp` (from `jobs.py`)
- `settings_bp` (from `settings.py`)
- `voices_bp` (from `voices.py`)
- `entities_bp` (from `entities.py`)
- `books_bp` (from `books.py`)
- `api_bp` (from `api.py`)

---

## 2. Blueprint: `main_bp` (prefix: none)

### Template Filters (app-wide)

| Filter | Signature | Behavior |
|--------|-----------|----------|
| `datetimeformat` | `(value: float, fmt="%Y-%m-%d %H:%M:%S") -> str` | Formats unix timestamp; returns em-dash if falsy |
| `durationformat` | `(value: Optional[float]) -> str` | Formats seconds into `Xs`, `Xm Ys`, or `Xh Ym` |

### Endpoints

| Method | Path | Function | Parameters | Behavior |
|--------|------|----------|------------|----------|
| GET | `/` | `index()` | Query: `pending_id` | If pending job exists, redirects to wizard step. Otherwise renders `index.html` with job stats, options, settings, jobs panel |
| GET | `/wizard` | `wizard_start()` | Query: `pending_id`, `step` (default "book") | Redirects to `/wizard/<step>` with optional pending_id |
| GET | `/wizard/<step>` | `wizard_step(step)` | Path: `step`; Query: `pending_id` | Normalizes step name; if JSON requested returns wizard JSON response; otherwise renders `index.html` in wizard mode |
| POST | `/wizard/upload` | `wizard_upload()` | Form: `pending_id`, `file`/`source_file`, all book step form fields | Case 1: existing pending + no new file -> updates settings, advances to "chapters". Case 2: new file -> saves to upload dir, extracts, builds pending job, advances to "chapters" |
| POST | `/wizard/text` | `wizard_text()` | Form: `text`, `title` (default "Pasted Text") | Saves text as .txt, extracts, builds pending job, advances to "chapters" |
| POST | `/wizard/update` | `wizard_update()` | Form/Query: `pending_id`; Form: `step`, `next_step`, all form fields | Updates pending job based on current step, stores, advances |
| POST | `/wizard/cancel` | `wizard_cancel()` | Form/Query: `pending_id` | Removes pending job; redirects or returns JSON `{"status":"cancelled","redirect_url":...}` |
| POST | `/wizard/finish` | `wizard_finish()` | Form/Query: `pending_id` | Final apply, calls `submit_job`, returns JSON with `job_id`, `redirect_url`, `jobs_panel` or redirects |

### Templates Referenced
- `index.html`

### htmx Patterns
- JSON responses via `wants_wizard_json()` (detects `Accept: application/json`, `X-Requested-With: xmlhttprequest|fetch`, `X-Abogen-Wizard: json`, query `?format=json`)

---

## 3. Blueprint: `jobs_bp` (prefix: /jobs)

### Endpoints

| Method | Path | Function | Parameters | Behavior |
|--------|------|----------|------------|----------|
| GET | `/<job_id>` | `job_detail(job_id)` | Path: `job_id` | Returns `job_detail.html`. Not found -> `job_not_found.html` with 200 |
| POST | `/<job_id>/pause` | `pause_job(job_id)` | Path: `job_id` | Pauses; htmx -> jobs panel partial; else redirect |
| POST | `/<job_id>/resume` | `resume_job(job_id)` | Path: `job_id` | Resumes; htmx -> jobs panel partial; else redirect |
| POST | `/<job_id>/cancel` | `cancel_job(job_id)` | Path: `job_id` | Cancels; htmx -> jobs panel partial; else redirect |
| POST | `/<job_id>/delete` | `delete_job(job_id)` | Path: `job_id` | Deletes; htmx -> jobs panel partial; else redirect to index |
| POST | `/<job_id>/retry` | `retry_job(job_id)` | Path: `job_id` | Retries; htmx -> jobs panel partial; else redirect to new job |
| POST | `/<job_id>/audiobookshelf` | `send_job_to_audiobookshelf(job_id)` | Form/Query: `overwrite` | Uploads to Audiobookshelf. Overwrite confirmation via HX-Trigger `audiobookshelf-overwrite-prompt` |
| POST | `/clear-finished` | `clear_finished_jobs()` | None | Clears finished; htmx -> panel; else redirect |
| GET | `/<job_id>/epub` | `job_epub(job_id)` | Path: `job_id` | Downloads EPUB file |
| GET | `/<job_id>/download/<file_type>` | `download_file(job_id, file_type)` | file_type: "audio" | Downloads audio file |
| GET | `/<job_id>/logs` | `job_logs(job_id)` | Path: `job_id` | Returns `job_logs_static.html` |
| GET | `/<job_id>/logs/partial` | `job_logs_partial(job_id)` | Path: `job_id` | Returns `partials/logs_section.html` |
| GET | `/<job_id>/logs/stream` | `stream_logs(job_id)` | Path: `job_id` | SSE stream; yields JSON log entries every 0.5s until terminal status |
| GET | `/<job_id>/reader` | `job_reader(job_id)` | Path: `job_id` | Returns `reader_embed.html` |
| GET | `/queue` | `queue_page()` | None | Returns `queue.html` |
| GET | `/partial` | `jobs_partial()` | None | Returns rendered jobs panel partial |

### Templates Referenced
- `job_detail.html`, `job_not_found.html`, `job_logs_static.html`, `job_logs_missing.html`
- `reader_embed.html`, `queue.html`
- `partials/jobs.html`, `partials/logs_section.html`, `partials/logs_section_missing.html`

---

## 4. Blueprint: `settings_bp` (prefix: /settings)

### Endpoints

| Method | Path | Function | Parameters | Behavior |
|--------|------|----------|------------|----------|
| GET/POST | `/` | `settings_page()` | GET Query: `debug_run_id`; POST delegates | Renders `settings.html` with all settings |
| POST | `/update` | `update_settings()` | Form: 40+ settings keys | Validates and saves all settings |
| POST | `/debug/run` | `run_debug_wavs()` | None | Runs debug TTS WAV generation |
| GET | `/debug/<run_id>` | `debug_wavs_page(run_id)` | Path: `run_id` | Renders `debug_wavs.html` |
| GET | `/debug/<run_id>/<filename>` | `download_debug_wav(run_id, filename)` | Query: `download` | Serves WAV/manifest file |

---

## 5. Blueprint: `voices_bp` (prefix: /voices)

### Endpoints

| Method | Path | Function | Parameters | Behavior |
|--------|------|----------|------------|----------|
| GET | `/` | `voice_profiles()` | None | Renders `voices.html` |
| POST | `/test` | `test_voice()` | Form: `text`, `voice`, `speed` | Synthesizes preview; returns WAV |
| GET | `/configs` | `speaker_configs()` | None | Returns JSON `{"configs": [...]}` |
| POST | `/configs/save` | `save_speaker_config()` | JSON: `name`, `config` | Saves; returns `{"status":"saved","configs":[...]}` |
| POST | `/configs/delete` | `delete_speaker_config()` | JSON: `name` | Deletes; returns `{"status":"deleted","configs":[...]}` |
| GET/POST | `/presets` | `speaker_configs_page()` | GET Query: `config`; POST Form: config fields | Renders `speakers.html`; POST saves preset |
| POST | `/presets/<name>/delete` | `delete_speaker_config_named(name)` | Path: `name` | Deletes; redirects |

---

## 6. Blueprint: `entities_bp` (prefix: /overrides)

### Endpoints

| Method | Path | Function | Parameters | Behavior |
|--------|------|----------|------------|----------|
| POST | `/analyze` | `analyze_entities()` | Form/Query: `pending_id` | Refreshes entity summary; returns JSON |
| GET | `/pending/<pending_id>` | `get_entities(pending_id)` | Query: `refresh`, `cache_key` | Returns entities payload; optionally refreshes |
| POST | `/pending/<pending_id>/refresh` | `refresh_entities(pending_id)` | None | Force refresh; returns JSON |
| GET | `/pending/<pending_id>/overrides` | `list_manual_overrides(pending_id)` | None | Returns overrides payload |
| POST | `/pending/<pending_id>/overrides` | `upsert_override(pending_id)` | JSON: `token`, `pronunciation`, `voice`, etc. | Upserts; returns override + entities payload |
| DELETE | `/pending/<pending_id>/overrides/<override_id>` | `delete_override(pending_id, override_id)` | None | Deletes; returns `{"deleted":true, ...}` |
| GET | `/pending/<pending_id>/overrides/search` | `search_candidates(pending_id)` | Query: `q`/`query`, `limit` | Returns search results |
| POST | `/overrides` | `upsert_global_override()` | Form: `action`, `lang`, `token`, `pronunciation`, `voice` | CRUD global override; redirects |
| GET | `/` | `entities_page()` | Query: `lang`, `voice`, `pronunciation` | Renders `entities.html` |

---

## 7. Blueprint: `books_bp` (prefix: /find-books)

### Endpoints

| Method | Path | Function | Behavior |
|--------|------|----------|----------|
| GET | `/` | `find_books_page()` | Renders `find_books.html` |
| GET | `/search` | `search_books()` | Same render |

---

## 8. Blueprint: `api_bp` (prefix: /api)

### Endpoints

| Method | Path | Function | Parameters | Response |
|--------|------|----------|------------|----------|
| GET | `/voice-profiles` | `api_get_voice_profiles()` | None | All profiles JSON |
| POST | `/voice-profiles` | `api_save_voice_profile()` | JSON: `name`, `profile`/fields | `{"success":true, "profile":"name", "profiles":{...}}` |
| DELETE | `/voice-profiles/<name>` | `api_delete_voice_profile(name)` | Path: `name` | `{"success":true, "profiles":{...}}` |
| POST | `/voice-profiles/<name>/duplicate` | `api_duplicate_voice_profile(name)` | JSON: `name` (new) | `{"success":true, "profile":"new", "profiles":{...}}` |
| POST | `/voice-profiles/import` | `api_import_voice_profiles()` | JSON: `data`, `replace_existing` | `{"success":true, "imported":N, "profiles":{...}}` |
| GET | `/voice-profiles/export` | `api_export_voice_profiles()` | Query: `names` | JSON file download |
| POST | `/voice-profiles/preview` | `api_voice_profiles_preview()` | JSON: `text`, `language`, `speed`, `formula`/`profile`/`voice`, etc. | WAV audio response |
| POST | `/speaker-preview` | `api_speaker_preview()` | JSON: `pending_id`, `text`, `voice`, `language`, `speed`, etc. | WAV with overrides applied |
| GET | `/integrations/calibre-opds/feed` | `api_calibre_opds_feed()` | Query: `href`, `q`, `letter` | `{"feed":{...}, "href":"", "query":""}` |
| POST | `/integrations/audiobookshelf/folders` | `api_abs_folders()` | JSON: connection params | `{"folders":[...]}` |
| POST | `/integrations/audiobookshelf/test` | `api_abs_test()` | JSON: connection params | `{"success":true, "message":"..."}` |
| POST | `/integrations/calibre-opds/test` | `api_calibre_opds_test()` | JSON: connection params | `{"success":true, "message":"..."}` |
| POST | `/integrations/calibre-opds/import` | `api_calibre_opds_import()` | JSON: `href`, `metadata` | `{"success":true, "pending_id":"...", "redirect_url":"..."}` |
| POST | `/llm/models` | `api_llm_models()` | JSON: `base_url`, `api_key`, `timeout` | `{"models":[...]}` |
| POST | `/llm/preview` | `api_llm_preview()` | JSON: `text`, LLM config fields | `{"text":"...", "normalized_text":"..."}` |
| POST | `/normalization/preview` | `api_normalization_preview()` | JSON: `text` | `{"text":"...", "normalized_text":"..."}` |
| POST | `/entity-pronunciation/preview` | `api_entity_pronunciation_preview()` | JSON: `token`, `pronunciation`, `voice`, `language` | `{"audio_base64":"..."}` |

---

## 9. Utility Module: `utils/common.py`

| Function | Behavior |
|----------|----------|
| `split_profile_spec(value)` | Parses "profile:Name"/"speaker:Name" prefixes |
| `split_speaker_spec(value)` | Alias for `split_profile_spec` |
| `existing_paths(paths)` | Filters to paths that exist on disk |

---

## 10. Utility Module: `utils/entity.py`

| Function | Behavior |
|----------|----------|
| `collect_pronunciation_overrides(pending)` | Aggregates overrides from entity_summary, speakers, manual_overrides; deduplicates by normalized token |
| `sync_pronunciation_overrides(pending)` | Assigns collected overrides to pending; syncs back into entity_summary |
| `refresh_entity_summary(pending, chapters)` | Runs entity extraction + heteronym detection; merges pronunciation store |
| `find_manual_override(pending, identifier)` | Finds by id or normalized token |
| `upsert_manual_override(pending, payload)` | Creates/updates; persists to global store; syncs |
| `delete_manual_override(pending, override_id)` | Removes; deletes from store; syncs |
| `search_manual_override_candidates(pending, query, *, limit=15)` | Searches entity index + store + manual overrides |
| `pending_entities_payload(pending)` | Standard entities response dict |

---

## 11. Utility Module: `utils/epub.py`

Key functions: `normalize_epub_path`, `decode_text`, `load_job_metadata`, `resolve_book_title`, `extract_epub_chapters`, `read_epub_bytes`, `locate_job_epub`, `locate_job_m4b`, `locate_job_audio`, `job_download_flags`.

Audio search priority: .m4b, .mp3, .flac, .opus, .ogg, .m4a, .wav.

---

## 12. Utility Module: `utils/form.py`

Key functions:
- `build_pending_job_from_extraction(...)` - Full pending job construction
- `apply_prepare_form(pending, form)` - Processes "chapters" step
- `apply_book_step_form(pending, form, *, settings, profiles)` - Processes "book" step
- `render_jobs_panel()` - Renders `partials/jobs.html`
- `wizard_json_response(pending, step, ...)` - Wizard JSON response builder

### Wizard Step Metadata
```
_WIZARD_STEP_ORDER = ["book", "chapters", "entities"]
```

### Wizard JSON Response Schema
```json
{
  "step": "chapters",
  "step_index": 2,
  "total_steps": 3,
  "title": "Select chapters",
  "hint": "...",
  "html": "<rendered partial>",
  "completed_steps": ["book", "chapters"],
  "pending_id": "abc123",
  "filename": "book.epub",
  "error": "",
  "notice": ""
}
```

---

## 13. Utility Module: `utils/preview.py`

| Function | Behavior |
|----------|----------|
| `get_preview_pipeline(language, device)` | Thread-safe cached KPipeline instances |
| `generate_preview_audio(text, voice_spec, language, speed, use_gpu, ...)` | Full audio generation with normalization; returns WAV bytes |
| `synthesize_preview(...)` | Wraps above in Flask `send_file` response |

---

## 14. Utility Module: `utils/service.py`

| Function | Behavior |
|----------|----------|
| `get_service()` | Gets ConversionService from Flask app extensions |
| `require_pending_job(pending_id)` | Gets pending or aborts 404 |
| `remove_pending_job(pending_id)` | Pops pending from service |
| `submit_job(pending)` | Enqueues full conversion job; returns job.id |

---

## 15. Utility Module: `utils/settings.py`

Key functions: `settings_defaults()` (40+ keys), `load_settings()`, `save_settings()`, `load_integration_settings()`, `build_audiobookshelf_config()`, `build_calibre_client()`, `apply_integration_form()`, type coercion helpers.

### Save Mode Labels
4 save modes: timestamped, simple, direct, custom.

---

## 16. Utility Module: `utils/voice.py`

Key functions:
- `build_narrator_roster(voice, voice_profile, existing)` - Narrator-only roster
- `build_speaker_roster(analysis, base_voice, voice_profile, existing, order)` - Full multi-speaker roster
- `apply_speaker_config_to_roster(roster, config, ...)` - Applies saved config
- `prepare_speaker_metadata(...)` - Full speaker analysis pipeline
- `template_options()` - Comprehensive template context
- `resolve_voice_choice(language, base_voice, profile_name, custom_formula, profiles)` - Full voice resolution
- `parse_voice_formula(formula)` - Parses weighted voice formulas

---

## 17. All Flask Templates (Consolidated)

| Template | Used By |
|----------|---------|
| `index.html` | main_bp |
| `job_detail.html` | jobs_bp |
| `job_not_found.html` | jobs_bp |
| `job_logs_static.html` | jobs_bp |
| `job_logs_missing.html` | jobs_bp |
| `reader_embed.html` | jobs_bp |
| `queue.html` | jobs_bp |
| `settings.html` | settings_bp |
| `debug_wavs.html` | settings_bp |
| `voices.html` | voices_bp |
| `speakers.html` | voices_bp |
| `entities.html` | entities_bp |
| `find_books.html` | books_bp |
| `partials/jobs.html` | render_jobs_panel() |
| `partials/logs_section.html` | jobs_bp |
| `partials/logs_section_missing.html` | jobs_bp |
| `partials/new_job_step_book.html` | wizard (step="book") |
| `partials/new_job_step_chapters.html` | wizard (step="chapters") |
| `partials/new_job_step_entities.html` | wizard (step="entities") |

---

## 18. htmx Patterns Summary

| Pattern | Location | Behavior |
|---------|----------|----------|
| `HX-Request` header check | All jobs_bp POST | Returns partial HTML instead of redirect |
| `HX-Trigger` response header | audiobookshelf upload | Triggers `audiobookshelf-overwrite-prompt` event |
| `HX-Target` header reading | audiobookshelf upload | Reads target element ID |
| JSON wizard responses | main_bp wizard endpoints | Via `wants_wizard_json()` |
| SSE streaming | `stream_logs` | `text/event-stream` with JSON data lines |
| Partial HTML responses | `jobs_partial`, `job_logs_partial` | Direct partials for htmx polling |

---

## 19. Cross-Reference: Core Module Dependencies

| Route/Utility | Core Modules Used |
|---------------|-------------------|
| main.py | webui.service, text_extractor, voice_profiles |
| jobs.py | webui.service, integrations.audiobookshelf |
| settings.py | debug_tts_runner, debug_tts_samples, utils |
| voices.py | speaker_configs, constants |
| entities.py | pronunciation_store, entity_analysis (via utils) |
| books.py | settings/voice utils only |
| api.py | voice_profiles, normalization_settings, llm_client, kokoro_text_normalization, integrations.audiobookshelf, integrations.calibre_opds, text_extractor |
| utils/entity.py | entity_analysis, pronunciation_store, heteronym_overrides |
| utils/epub.py | webui.service (Job, JobStatus) |
| utils/form.py | utils, voice_profiles, chunking, constants, speaker_configs, kokoro_text_normalization |
| utils/preview.py | utils, tts_supertonic, voice_formulas, kokoro_text_normalization, webui.conversion_runner |
| utils/service.py | webui.service |
| utils/settings.py | constants, normalization_settings, utils, integrations.calibre_opds, integrations.audiobookshelf |
| utils/voice.py | speaker_configs, speaker_analysis, voice_profiles, voice_formulas, constants, utils, webui.conversion_runner |
