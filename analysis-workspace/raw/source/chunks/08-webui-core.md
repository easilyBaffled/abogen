# Chunk 8: webui-core

## Files
- `abogen/webui/__init__.py`
- `abogen/webui/app.py`
- `abogen/webui/service.py`
- `abogen/webui/conversion_runner.py`
- `abogen/webui/debug_tts_runner.py`

## Description
Flask web application core. `app.py` is the app factory (blueprint registration, secret key, directory setup). `service.py` defines Job/JobStatus and the ConversionService (thread-pool job execution, state persistence, pause/resume/cancel, Audiobookshelf upload). `conversion_runner.py` is the main pipeline: text extraction, chunking, voice resolution (profiles/formulas/speaker configs), Kokoro/SuperTonic synthesis loop, WAV concatenation, ffmpeg encoding (mp3/opus/m4b/flac), subtitle generation, EPUB3 media overlay export. `debug_tts_runner.py` generates per-phoneme debug audio samples.

## Internal Relationships
- `app.py` imports `service.build_service` and `conversion_runner.run_conversion_job`
- `service.py` imports from `utils`, `voice_cache`, and `integrations.audiobookshelf`
- `conversion_runner.py` is the heaviest importer: pulls from nearly every core module (text_extractor, chunking, normalization, voice_formulas, voice_profiles, voice_cache, tts_supertonic, epub3, entity_analysis, pronunciation_store, llm_client)
