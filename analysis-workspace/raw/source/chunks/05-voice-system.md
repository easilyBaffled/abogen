# Chunk 5: voice-system

## Files
- `abogen/voice_formulas.py`
- `abogen/voice_profiles.py`
- `abogen/voice_cache.py`
- `abogen/tts_supertonic.py`

## Description
Voice management. `voice_formulas.py` parses weighted voice blending formulas (e.g., "af_heart*0.7 + am_echo*0.3") and loads/blends Kokoro voice tensors. `voice_profiles.py` provides CRUD for named voice profiles (JSON persistence with provider support for Kokoro and SuperTonic). `voice_cache.py` ensures Kokoro voice weight files are downloaded from HuggingFace. `tts_supertonic.py` is an adapter for the SuperTonic TTS engine (ONNX-based, supports GPU via CUDAExecutionProvider, handles unsupported character recovery).

## Internal Relationships
- `voice_formulas.py` and `voice_profiles.py` both import VOICES_INTERNAL from constants
- `voice_profiles.py` also imports DEFAULT_SUPERTONIC_VOICES from `tts_supertonic`
- `voice_cache.py` uses `huggingface_hub` for downloads
- The conversion runner imports all four modules
