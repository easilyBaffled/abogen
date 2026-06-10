# Chunk 7: llm-client

## Files
- `abogen/llm_client.py`

## Description
Minimal HTTP client for OpenAI-compatible LLM APIs using only stdlib (urllib). Provides `LLMConfiguration` dataclass, `list_models()`, and `generate_completion()` with tool-call support. URL normalization handles trailing slashes and v1 prefix deduplication. Authentication via Bearer token (skipped for "ollama" key).

## Internal Relationships
- Imported by `normalization_settings.py` for `LLMConfiguration` type
- Used at runtime by the normalization pipeline (via `kokoro_text_normalization.py`) for LLM-assisted contraction resolution
