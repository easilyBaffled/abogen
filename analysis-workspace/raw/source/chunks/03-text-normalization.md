# Chunk 3: text-normalization

## Files
- `abogen/kokoro_text_normalization.py`
- `abogen/normalization_settings.py`
- `abogen/spacy_contraction_resolver.py`
- `abogen/word_substitution.py`
- `abogen/heteronym_overrides.py`

## Description
Text preprocessing pipeline for TTS readiness. `kokoro_text_normalization.py` is the main normalizer (numbers, currency, contractions, phoneme hints, abbreviations). `normalization_settings.py` provides runtime settings extraction from config with LLM configuration building. `spacy_contraction_resolver.py` uses spaCy POS tagging for context-aware contraction expansion. `word_substitution.py` handles user-defined word replacements, ALL CAPS conversion, numeral conversion. `heteronym_overrides.py` handles words with multiple pronunciations (e.g., "read" past vs. present).

## Internal Relationships
- `kokoro_text_normalization.py` imports `spacy_contraction_resolver`
- `normalization_settings.py` imports from `kokoro_text_normalization` (ApostropheConfig, CONTRACTION_CATEGORY_DEFAULTS) and `llm_client` (LLMConfiguration)
- `chunking.py` imports `normalize_for_pipeline` from `kokoro_text_normalization` and settings builders from `normalization_settings`
