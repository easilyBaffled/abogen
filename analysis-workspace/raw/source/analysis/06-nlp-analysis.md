# NLP Analysis Chunk: Complete Source Analysis

## File: `abogen/speaker_analysis.py`

### Overview
Pure-Python (no ML dependencies) module for rule-based speaker detection and dialogue attribution in narrative text. Uses regex patterns and heuristics to identify who is speaking in text chunks.

### Public Classes

#### `SpeakerGuess` (dataclass, slots=True)
Fields:
- `speaker_id: str` - unique slug identifier
- `label: str` - display name
- `count: int` (default 0) - number of chunks attributed
- `confidence: str` (default "low") - one of "low", "medium", "high"
- `sample_quotes: List[Dict[str, str]]` - up to 3 samples, each with `excerpt` and `gender_hint` keys
- `suppressed: bool` (default False) - whether speaker was below threshold
- `gender: str` (default "unknown") - resolved gender: "male", "female", or "unknown"
- `detected_gender: str` (default "unknown") - same enum, independent tracking
- `male_votes: int` (default 0) - accumulated weighted male pronoun score
- `female_votes: int` (default 0) - accumulated weighted female pronoun score

Methods:
- `register_occurrence(confidence, text, quote, male_votes, female_votes, sample_excerpt=None)` - Increments count, upgrades confidence if higher rank, appends sample quotes (max 3), accumulates gender votes, calls `_derive_gender` to update both `detected_gender` and `gender`.
- `as_dict() -> Dict[str, Any]` - Serializes to JSON-ready dict with keys: `id`, `label`, `count`, `confidence`, `sample_quotes`, `suppressed`, `gender`, `detected_gender`.

#### `SpeakerAnalysis` (dataclass, slots=True)
Fields:
- `assignments: Dict[str, str]` - chunk_id -> speaker_id mapping
- `speakers: Dict[str, SpeakerGuess]` - all speaker records
- `suppressed: List[str]` - list of suppressed speaker IDs
- `narrator: str` (default "narrator")
- `version: str` (default "1.0")
- `stats: Dict[str, Any]` - summary statistics

Methods:
- `to_dict() -> Dict[str, Any]` - Full serialization including nested speaker dicts and stats.

### Public Functions

#### `analyze_speakers(chapters, chunks, *, threshold=3, max_speakers=8) -> SpeakerAnalysis`
Main entry point. Processes sorted chunks to assign speakers.

**Algorithm:**
1. Sorts chunks by `(chapter_index, chunk_index)`.
2. Iterates chunks sequentially, calling `_infer_chunk_speaker` for each.
3. If inference returns `None`, falls back to `last_explicit` speaker (confidence "medium") or narrator (confidence "low").
4. Maintains a `last_explicit` tracker that updates whenever a non-narrator speaker is identified.
5. Normalizes labels, creates slugified IDs, deduplicates against existing speakers.
6. Counts gender votes per chunk using `_count_gender_votes`.
7. Selects sample excerpts (with surrounding context) for high-confidence attributions.
8. **Post-processing:** Suppresses speakers with count below `threshold` (minimum 1), reassigns their chunks to narrator. Caps active speakers at `max_speakers`, suppressing least-frequent beyond that cap.
9. Recounts narrator assignments. Returns `SpeakerAnalysis`.

### Speaker Detection Algorithm (Private Functions)

#### `_infer_chunk_speaker(text, last_explicit) -> Tuple[Optional[str], str, Optional[str]]`
Returns `(speaker_name_or_None, confidence, quote_or_None)`.

**Detection priority:**
1. **Colon pattern** (`Name: dialogue`) - matches `^Name:` at line start. Confidence: "high".
2. **Quote extraction** - finds quoted text using smart/straight quote regex.
3. **Name near quote** - searches 120 chars before/after quote for Name+DialogueVerb or DialogueVerb+Name patterns. Confidence: "high".
4. **Pronoun fallback** - if pronoun (he/she/they) found near quote, assigns to `last_explicit`. Confidence: "medium".
5. Falls through returning `(None, "low", quote)`.

#### Key Patterns
- `_DIALOGUE_VERBS`: 20 verbs ("said", "asked", "replied", "whispered", "shouted", etc.)
- `_NAME_FRAGMENT`: Unicode-aware capitalized word regex `[A-ZÀ-ÖØ-Þ][\w''\-]*`
- `_NAME_PATTERN`: One or more name fragments separated by whitespace
- `_COLON_PATTERN`: `^\s*(Name)\s*:\s*(.+)$`
- `_NAME_BEFORE_VERB`: `(Name)\s+VERB\b`
- `_VERB_BEFORE_NAME`: `VERB\s+(Name)`
- `_QUOTE_PATTERN`: Matches text within smart or straight double quotes, handling escaped characters

#### Gender Detection (`_count_gender_votes`)
Sophisticated weighted voting system:
1. Finds all occurrences of speaker's label (name) in text. Falls back to diacritics-stripped matching.
2. If no label matches found, uses entire text but applies 0.25 degradation factor.
3. For each pronoun match, computes weight as `window_weight * quote_weight`:
   - `_window_weight`: 1.0 if pronoun is after name within 60-char radius, 0.2 if before, 0.0 if outside radius.
   - `_quote_weight`: Reduces weight for pronouns inside quoted speech (0.05-0.6 depending on context), full 1.0 outside quotes.
4. Token weights: "he"=1.0, "him"=0.6, "his"=0.75, "himself"=1.0; "she"=1.0, "her"=0.4, "hers"=0.75, "herself"=1.0.
5. Title hints ("Mrs", "Monsieur", etc.) add 2.5 to respective gender score within 40 chars of name.

#### Gender Derivation (`_derive_gender`)
Requires votes to exceed `max(2, opposing_votes + 1)` to assign gender. Retains existing assignment if threshold not met.

#### Name Normalization (`_normalize_candidate_name`)
- Strips quotes, punctuation, collapses whitespace.
- Removes leading/trailing stop words ("and", "but", "then", etc.).
- Keeps only leading contiguous capitalized words.
- Rejects if result is a pronoun or stop word.

### Edge Cases / Error Handling
- Empty text returns `(None, "low", None)`.
- Non-dict chunks produce empty text via `_get_chunk_text`.
- Diacritics in names handled via NFKD normalization + proportional index mapping back.
- Quote detection tolerates escaped characters inside quotes.
- `_safe_int` handles non-numeric chapter/chunk indices gracefully (returns 0).

---

## File: `abogen/entity_analysis.py`

### Overview
spaCy-powered Named Entity Recognition (NER) module that extracts people and other entities from chapter text, producing structured summary with frequency counts, sample sentences, and searchable token index.

### Public Classes

#### `EntityRecord` (dataclass, slots=True)
Fields:
- `key: Tuple[str, str]` - (category, normalized_key)
- `label: str` - display label
- `kind: str` - NER label (e.g., "PERSON", "ORG", "GPE", "PROPN")
- `category: str` - "people" or "entities"
- `count: int` - occurrence count
- `samples: List[Dict[str, Any]]` - up to 5 samples with `excerpt` and `chapter_index`
- `chapter_indices: set[int]` - which chapters contain this entity
- `forms: Counter` - tracks variant surface forms
- `first_position: Optional[Tuple[int, int]]` - (chapter_index, token_position)

Methods:
- `register(*, chapter_index, position, text, sentence)` - Increments count, adds chapter, tallies form, sets first_position, appends up to 5 unique sample excerpts.
- `as_dict(ordinal) -> Dict[str, Any]` - Serializes with computed `id` field (`"{category}_{ordinal}"`), sorted chapter_indices, forms as top-6 list.

#### `EntityExtractionResult` (dataclass, slots=True)
Fields: `summary: Dict[str, Any]`, `cache_key: str`, `elapsed: float`, `errors: List[str]`

#### `EntityModelError(RuntimeError)`
Raised when spaCy or its model unavailable.

### Public Functions

#### `extract_entities(chapters, *, language="en") -> EntityExtractionResult`
Main extraction pipeline.

**Algorithm:**
1. Computes SHA-1 cache key from all chapter texts and indices.
2. Loads spaCy model via `_load_model(language)`.
3. For each chapter: processes with spaCy NLP pipeline, then:
   - Iterates named entities via `_iter_named_entities` (skips excluded NER labels: CARDINAL, DATE, ORDINAL, PERCENT, TIME, LAW, MONEY, QUANTITY).
   - Extracts standalone PROPN tokens not already part of NER spans via `_extract_propn_tokens`.
4. Each span normalized, categorized ("people" for PERSON, "entities" otherwise), registered.
5. Builds token index for search functionality.
6. Separates people from entities (entities excludes anything with PERSON kind or matching people key).
7. Sorts both lists by count descending, then label alphabetically.

**Output JSON structure:**
```json
{
  "people": [{"id", "label", "normalized", "category", "kind", "count", "samples", "chapter_indices", "first_chapter", "forms"}],
  "entities": [...same structure...],
  "index": {"tokens": [{"token", "normalized", "category", "count", "samples"}]},
  "stats": {"tokens", "chapters", "processed", "people", "entities"},
  "model": {"name", "version", "lang"}
}
```

#### `search_tokens(index, query, *, limit=15) -> List[Dict[str, Any]]`
Searches token index by substring matching against token label or normalized form. Returns up to `limit` results.

#### `merge_override(summary, overrides) -> Dict[str, Any]`
Merges pronunciation/voice overrides into entity summary. For each person/entity whose normalized key appears in `overrides`, adds `"override"` sub-dict to entry.

#### `normalize_token(token: str) -> str`
Public normalization: applies `_normalize_label` then `_token_key` (lowercase + collapse spaces).

#### `normalize_manual_override_token(token: str) -> str`
Simpler normalization for manual user input: strips quotes, lowercases, collapses spaces. Does NOT apply title/suffix stripping.

### Entity Label Normalization (`_normalize_label`)
1. Strips quotes.
2. Removes title prefixes (Mr, Mrs, Dr, Sir, Captain, etc. - 17 titles).
3. Removes suffixes (Jr, Sr, II-VI, MD, PhD, Esq, etc.).
4. Removes possessives ('s).
5. Removes non-word characters except hyphens and apostrophes.
6. Collapses whitespace.
7. Rejects single-character results and stop words.
8. Title-cases each part (preserves all-uppercase abbreviations).

### spaCy Model Loading (`_load_model`)
- Thread-safe via `_MODEL_LOCK` (RLock).
- Model name resolved from `ABOGEN_SPACY_MODEL` env var, or defaults to `en_core_web_sm`.
- Caches loaded models in `_MODEL_CACHE` dict.
- Sets `nlp.max_length` to at least 2,000,000 tokens.
- Raises `EntityModelError` if spaCy not installed or model not downloadable.

### Edge Cases
- spaCy import wrapped in try/except; module sets `spacy = None` if unavailable.
- Empty chapter list returns `_empty_result` with `processed: False`.
- Individual chapters with text exceeding `nlp.max_length` trigger dynamic increase.
- `_extract_propn_tokens` avoids double-counting tokens already in NER spans.

---

## File: `abogen/speaker_configs.py`

### Overview
CRUD module for persistent speaker voice configurations stored as JSON on disk. Manages named configuration presets that map speaker IDs to voice/language settings.

### Storage Location
File path: `{user_config_dir}/speaker_configs.json` (derived from `get_user_config_path()`).

### JSON Structure
```json
{
  "abogen_speaker_configs": {
    "config_name": {
      "language": "a",
      "languages": ["a", "b"],
      "default_voice": "voice_id",
      "speakers": {
        "speaker_slug": {
          "id": "speaker_slug",
          "label": "Display Name",
          "gender": "male|female|unknown",
          "voice": "voice_id",
          "voice_profile": "profile_id",
          "voice_formula": "formula_string",
          "resolved_voice": "final_voice",
          "languages": ["a"]
        }
      },
      "version": 1,
      "notes": "optional text"
    }
  }
}
```

### Public Functions

#### `load_configs() -> Dict[str, Dict[str, Any]]`
Loads all configurations from disk. Returns empty dict if file missing or corrupt. Handles both wrapped (`abogen_speaker_configs` key) and unwrapped formats.

#### `save_configs(configs: Dict[str, Dict[str, Any]]) -> None`
Saves all configurations. Sanitizes each entry. Writes with wrapper key, indented, sorted keys.

#### `get_config(name: str) -> Optional[Dict[str, Any]]`
Retrieves single config by name. Returns None if not found or name empty.

#### `upsert_config(name: str, payload: Dict[str, Any]) -> Dict[str, Any]`
Creates or updates named config. Raises `ValueError` if name empty. Sanitizes payload before saving.

#### `delete_config(name: str) -> None`
Removes config by name. No-op if not found.

#### `list_configs() -> List[Dict[str, Any]]`
Returns all configs as list sorted by name, with `name` field injected into each entry.

#### `slugify_label(label: str) -> str`
Converts label to alphanumeric slug with underscores. Returns "speaker" for empty input.

#### `describe_language(code: str) -> str`
Looks up language code in `LANGUAGE_DESCRIPTIONS` constant. Falls back to uppercased code.

### Sanitization Logic (`_sanitize_config`, `_sanitize_speaker`)
- Config: normalizes language to lowercase, validates `languages` as list of lowercase strings, validates `default_voice` as string, sanitizes each speaker entry, ensures integer version.
- Speaker: validates gender enum (male/female/unknown), resolves voice from formula/profile/voice fallback chain, generates slug ID from label if none provided.

### Edge Cases
- Corrupt JSON file returns empty dict (silently).
- Non-string/non-dict entries in payload skipped.
- Empty config names rejected (`ValueError` on upsert, no-op on delete/get).

---

## File: `abogen/pronunciation_store.py`

### Overview
Persistent pronunciation override store. Originally SQLite-backed, now migrated to JSON file. Provides CRUD operations for per-language pronunciation/voice overrides keyed by normalized entity tokens.

### Storage Location
File path: `{user_settings_dir}/overrides.json` (via `get_user_settings_dir()` or fallback `get_internal_cache_path("pronunciations")`).

### JSON Structure
```json
{
  "version": 1,
  "overrides": {
    "language_code": {
      "normalized_token": {
        "id": "uuid",
        "normalized": "normalized_token",
        "token": "Original Token",
        "language": "language_code",
        "pronunciation": "IPA or phonetic",
        "voice": "voice_id",
        "notes": "user notes",
        "context": "usage context",
        "usage_count": 0,
        "created_at": 1234567890.0,
        "updated_at": 1234567890.0
      }
    }
  }
}
```

### Public Functions

#### `load_overrides(language: str, tokens: Iterable[str]) -> Dict[str, Dict[str, Any]]`
Loads overrides for specific tokens in language. Normalizes each token via `normalize_token` from `entity_analysis`. Thread-safe via `_DB_LOCK`.

#### `search_overrides(language: str, query: str, *, limit=15) -> List[Dict[str, Any]]`
Substring search against normalized key and token label. Sorted by usage_count descending, then updated_at descending.

#### `save_override(*, language, token, pronunciation=None, voice=None, notes=None, context=None) -> Dict[str, Any]`
Creates or updates override entry. Generates UUID for new entries. Updates timestamp on modification. Raises `ValueError` if token normalizes to empty.

#### `delete_override(*, language: str, token: str) -> None`
Removes override by normalized token. No-op if not found.

#### `all_overrides(language: str) -> List[Dict[str, Any]]`
Returns all overrides for language, sorted by updated_at descending.

#### `increment_usage(*, language: str, token: str, amount: int = 1) -> None`
Increments `usage_count` and updates `updated_at` timestamp.

#### `get_override_stats(language: str) -> Dict[str, int]`
Returns counts: `total`, `filtered`, `with_pronunciation`, `with_voice`.

### Legacy Migration (`_migrate_legacy_sqlite`)
- Checks for `pronunciations.db` SQLite file.
- If found, reads all rows from `overrides` table, converts to JSON, writes to `overrides.json`, renames SQLite to `.db.bak`.
- Silently catches all exceptions during migration.

### Concurrency
- All public functions use `_DB_LOCK` (threading.RLock) for thread safety.
- Writes use atomic file operations: write to `.tmp` then `shutil.move`.

---

## File: `abogen/spacy_utils.py`

### Overview
Lazy-loaded spaCy utilities specifically for sentence segmentation. Separate from `entity_analysis.py`'s model loading -- serves text-chunking pipeline with different model configurations.

### Key Difference from entity_analysis Model Loading
- `entity_analysis._load_model`: loads full pipeline (all components), sets max_length to 2M, uses only `en_core_web_sm`.
- `spacy_utils.get_spacy_model`: **disables** NER, tagger, lemmatizer, attribute_ruler (only keeps parser for sentence boundaries), supports multiple languages.

### Language Model Mapping (`SPACY_MODELS` dict)
| Code | Model | Language |
|------|-------|----------|
| `a` | `en_core_web_sm` | American English |
| `b` | `en_core_web_sm` | British English |
| `e` | `es_core_news_sm` | Spanish |
| `f` | `fr_core_news_sm` | French |
| `i` | `it_core_news_sm` | Italian |
| `p` | `pt_core_news_sm` | Brazilian Portuguese |
| `z` | `zh_core_web_sm` | Mandarin Chinese |
| `j` | `ja_core_news_sm` | Japanese |
| `h` | `xx_sent_ud_sm` | Hindi (multi-language) |

### Public Functions

#### `get_spacy_model(lang_code, log_callback=None) -> nlp | None`
Loads or retrieves cached spaCy model for sentence segmentation.

**Behavior:**
1. Returns cached model if available in `_nlp_cache`.
2. Looks up model name from `SPACY_MODELS`. Returns None if language not mapped.
3. Lazy-loads spaCy module. Returns None if not installed.
4. Loads model with disabled components: `["ner", "tagger", "lemmatizer", "attribute_ruler"]`.
5. If parser not in pipeline, adds sentencizer as fallback.
6. On `OSError` (model not found): auto-downloads via `spacy.cli.download`, then retries.
7. Caches successful load in `_nlp_cache`.

#### `segment_sentences(text, lang_code, log_callback=None) -> List[str] | None`
Segments text into sentences. Returns None if spaCy unavailable. Dynamically increases `max_length` if text exceeds current limit.

#### `is_spacy_available() -> bool`
Returns whether spaCy can be imported.

#### `clear_cache() -> None`
Clears `_nlp_cache` dict to free memory.

### Edge Cases
- spaCy not installed: all functions return None gracefully.
- Model not found: auto-download attempted. If download fails, returns None with error log.
- Large texts: `max_length` dynamically increased to `len(text) + 1000`.
- Missing parser component: sentencizer pipe added as fallback.
- No thread lock (unlike entity_analysis) - module-level `_nlp_cache` dict not thread-protected.

---

## Cross-Module Relationships

1. **`pronunciation_store` depends on `entity_analysis`**: imports `normalize_token` for consistent key normalization.
2. **`speaker_configs` depends on `abogen.constants`** (for `LANGUAGE_DESCRIPTIONS`) and **`abogen.utils`** (for `get_user_config_path`).
3. **`pronunciation_store` depends on `abogen.utils`**: uses `get_internal_cache_path` and `get_user_settings_dir`.
4. **`entity_analysis` and `spacy_utils` are independent spaCy consumers** with different loading strategies and component configurations.
5. **`speaker_analysis` is fully standalone**: no dependencies on spaCy or other abogen modules; pure regex-based.

## Storage Format Summary

| Module | Format | Location | Thread-safe |
|--------|--------|----------|-------------|
| `speaker_configs` | JSON (wrapper key) | user config dir | No |
| `pronunciation_store` | JSON (flat) | user settings dir | Yes (RLock) |
| `entity_analysis` | In-memory only (returns dict) | N/A | Model cache: Yes (RLock) |
| `spacy_utils` | In-memory cache only | N/A | No |
