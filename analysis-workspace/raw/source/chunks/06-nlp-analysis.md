# Chunk 6: nlp-analysis

## Files
- `abogen/speaker_analysis.py`
- `abogen/entity_analysis.py`
- `abogen/speaker_configs.py`
- `abogen/pronunciation_store.py`
- `abogen/spacy_utils.py`

## Description
NLP content analysis. `speaker_analysis.py` detects speakers from dialogue attribution patterns (colon-prefixed, said-verbs) with gender inference from pronoun proximity. `entity_analysis.py` extracts named entities via spaCy NER (people, organizations, locations). `speaker_configs.py` persists per-book speaker voice assignments. `pronunciation_store.py` stores user pronunciation overrides in JSON (migrated from SQLite). `spacy_utils.py` provides lazy spaCy model loading with per-language model selection.

## Internal Relationships
- `speaker_analysis.py` is standalone (no imports from other abogen modules)
- `entity_analysis.py` uses spaCy directly
- `speaker_configs.py` imports from `constants` and `utils`
- `pronunciation_store.py` imports `normalize_token` from `entity_analysis` and paths from `utils`
- `spacy_utils.py` is used by the PyQt conversion module
