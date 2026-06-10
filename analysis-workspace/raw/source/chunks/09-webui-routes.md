# Chunk 9: webui-routes

## Files
- `abogen/webui/routes/__init__.py`
- `abogen/webui/routes/main.py`
- `abogen/webui/routes/jobs.py`
- `abogen/webui/routes/settings.py`
- `abogen/webui/routes/voices.py`
- `abogen/webui/routes/entities.py`
- `abogen/webui/routes/books.py`
- `abogen/webui/routes/api.py`
- `abogen/webui/routes/utils/common.py`
- `abogen/webui/routes/utils/entity.py`
- `abogen/webui/routes/utils/epub.py`
- `abogen/webui/routes/utils/form.py`
- `abogen/webui/routes/utils/preview.py`
- `abogen/webui/routes/utils/service.py`
- `abogen/webui/routes/utils/settings.py`
- `abogen/webui/routes/utils/voice.py`

## Description
Flask route blueprints and utilities. `main.py` serves the index/upload page. `jobs.py` handles job CRUD and output download. `settings.py` manages normalization/LLM settings. `voices.py` manages voice profiles. `entities.py` handles entity overrides/pronunciation. `books.py` integrates Calibre OPDS book browsing. `api.py` provides REST endpoints. Route utils extract common patterns (form parsing, voice resolution, settings validation, entity operations, EPUB preview, service access).

## Internal Relationships
- All route modules access ConversionService via `app.extensions["conversion_service"]`
- Import from core modules for validation (voice_formulas, voice_profiles, speaker_configs, normalization_settings, entity_analysis, text_extractor, llm_client, integrations)
