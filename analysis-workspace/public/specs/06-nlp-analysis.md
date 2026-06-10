# Behavioral Specification: NLP and Entity Analysis

## Module Identity

**Paths**: `abogen/entity_analysis.py`, `abogen/speaker_analysis.py`, `abogen/spacy_utils.py`, `abogen/heteronym_overrides.py`, `abogen/pronunciation_store.py`  
**Role**: spaCy-based NLP subsystem for entity extraction, speaker detection, heteronym disambiguation, sentence segmentation, and pronunciation override persistence  
**Boundaries**: Produces analysis results consumed by WebUI routes and conversion runner. Does NOT perform TTS synthesis or audio generation.  
**Dependencies**: spaCy (optional — graceful degradation when unavailable), `entity_analysis.normalize_token` used by pronunciation_store

---

## Public Interface

### entity_analysis.py

| Export | Type | Description |
|--------|------|-------------|
| `EntityRecord` | dataclass | Entity with key, label, frequency, samples |
| `EntityExtractionResult` | dataclass | Full extraction results |
| `extract_entities(chapters, *, language)` | function | NER pipeline |
| `normalize_token(token)` | function | Token normalization for deduplication |

### speaker_analysis.py

| Export | Type | Description |
|--------|------|-------------|
| `SpeakerEntry` | dataclass | Detected speaker with gender, confidence |
| `SpeakerAnalysis` | dataclass | Full analysis results |
| `analyze_speakers(chapters, *, language)` | function | Speaker detection pipeline |

### heteronym_overrides.py

| Export | Type | Description |
|--------|------|-------------|
| `HeteronymSpec` | frozen dataclass | Token with 2 variant definitions |
| `HeteronymVariant` | frozen dataclass | Single variant (key, label, replacement, example) |
| `extract_heteronym_overrides(chapters, *, language, existing)` | function | Finds heteronyms in text |

### pronunciation_store.py

| Export | Type | Description |
|--------|------|-------------|
| `load_overrides(language, tokens)` | function | Load matching overrides |
| `search_overrides(language, query, *, limit=15)` | function | Search by partial match |
| `save_override(*, language, token, pronunciation, voice, notes, context)` | function | Create/update override |
| `delete_override(language, token)` | function | Remove override |
| `increment_usage(language, token)` | function | Bump usage counter |
| `list_all_overrides(language)` | function | All overrides for language |

### spacy_utils.py

| Export | Type | Description |
|--------|------|-------------|
| `segment_sentences(text, *, language)` | function | spaCy sentence segmentation |
| `SPACY_MODELS` | dict | Language code → model name mapping |

---

## Entity Extraction

### NER Pipeline
**Given** chapters with text content and a language code  
**When** `extract_entities(chapters, language=lang)` is called  
**Then** loads spaCy model, processes text, extracts named entities, deduplicates by normalized key, counts frequency, collects sample sentences  
**Source**: `entity_analysis.py:extract_entities`

### Excluded NER Labels
**Given** spaCy produces entity spans  
**When** filtering entities  
**Then** excludes labels: CARDINAL, DATE, ORDINAL, PERCENT, TIME, LAW, MONEY, QUANTITY  
**Source**: `entity_analysis.py:74-83` (`_EXCLUDED_NER_LABELS`)

### Stop Word Filtering
**Given** an entity text  
**When** normalizing for inclusion  
**Then** entities whose normalized form matches stop words (the, that, this, and, but, dr, mr, etc.) are excluded  
**Source**: `entity_analysis.py:48-72` (`_STOP_LABELS`)

### Token Normalization
**Given** a raw entity or token string  
**When** `normalize_token(token)` is called  
**Then** strips title prefixes (Mr., Dr., etc.), removes possessive suffixes ('s), removes suffixes (Jr, Sr, II, etc.), removes non-word chars except hyphens/apostrophes, collapses whitespace, lowercases  
**Source**: `entity_analysis.py:85-95` (regex patterns), `normalize_token` function

### Thread-Safe Model Loading
**Given** concurrent calls to entity extraction  
**When** spaCy model is loaded  
**Then** protected by `_MODEL_LOCK` (threading.RLock) with model caching  
**Source**: `entity_analysis.py` (threading.RLock)

### spaCy Unavailable
**Given** spaCy import failed (spacy = None)  
**When** `extract_entities()` is called  
**Then** returns empty/default result without raising  
**Source**: `entity_analysis.py:12-16`

---

## Speaker Analysis

### Detection Patterns
**Given** text with dialogue  
**When** `analyze_speakers(chapters, language=lang)` is called  
**Then** detects speakers via:
1. Colon pattern: `Name: "dialogue"` 
2. Name-before-verb: `Name said/asked/whispered...`
3. Verb-before-name: `said Name`
**Source**: `speaker_analysis.py:36-38` (`_COLON_PATTERN`, `_NAME_BEFORE_VERB`, `_VERB_BEFORE_NAME`)

### Dialogue Verbs
**Given** speaker detection patterns  
**When** matching verb patterns  
**Then** recognizes 20 verbs: said, asked, replied, whispered, shouted, cried, muttered, answered, hissed, called, added, continued, insisted, remarked, yelled, breathed, murmured, exclaimed, explained, noted  
**Source**: `speaker_analysis.py:9-30`

### Gender Inference
**Given** a detected speaker name  
**When** inferring gender  
**Then** uses:
1. Title hints (Mrs/Miss/Ms/Lady → female; Mr/Sir/Lord → male)
2. Surrounding pronoun analysis (he/him → male; she/her → female)
3. Defaults to "unknown"
**Source**: `speaker_analysis.py:76-100` (`_FEMALE_TITLE_HINTS`, `_MALE_TITLE_HINTS`)

### Pronoun Exclusion
**Given** detected name candidates  
**When** filtering  
**Then** excludes entries matching pronoun labels (he, she, they, him, etc.)  
**Source**: `speaker_analysis.py:43-72` (`_PRONOUN_LABELS`)

### Confidence Ranking
**Given** multiple detections of same speaker  
**When** consolidating  
**Then** confidence ordered: high > medium > low (rank 3, 2, 1)  
**Source**: `speaker_analysis.py:74` (`_CONFIDENCE_RANK`)

---

## Heteronym Detection

### Supported Heteronyms (English Only)
**Given** English text  
**When** `extract_heteronym_overrides()` processes it  
**Then** detects 5 words: wind, read, tear, close, lead  
**Source**: `heteronym_overrides.py:52-138` (`_HETERONYM_SPECS`)

### POS-Based Disambiguation
**Given** a heteronym token with spaCy POS tag  
**When** `default_choice_for_token()` is called  
**Then** dispatches by token:
- wind: VERB → "verb" (/waɪnd/), else → "noun" (/wɪnd/)
- read: VBD/VBN → "past" (/rɛd/), else → "present" (/riːd/)
- tear: VERB → "verb" (/tɛr/), else → "noun" (/tɪr/)
- close: VERB → "verb" (/kloʊz/), else → "adj" (/kloʊs/)
- lead: NOUN → "metal" (/lɛd/), else → "verb" (/liːd/)
**Source**: `heteronym_overrides.py:27-46`

### Replacement Tokens
**Given** a heteronym variant  
**When** applying replacement  
**Then** uses phonetically-distinct tokens: wind→"wynd", read→"red", tear→"tier", close→"cloze", lead→"led"  
**Source**: `heteronym_overrides.py:53-137`

### Case Preservation
**Given** a replacement to apply  
**When** `_preserve_case()` is called  
**Then** matches original casing: ALL CAPS → upper, Title Case → capitalize first, else lowercase  
**Source**: `heteronym_overrides.py:162-169`

### Non-English Bypass
**Given** language code not starting with "en"  
**When** `extract_heteronym_overrides()` is called  
**Then** returns empty list immediately  
**Source**: `heteronym_overrides.py:221-222`

### Deduplication
**Given** same heteronym in same sentence appears multiple times  
**When** processing chapters  
**Then** uses `seen` set of (token_key, sentence) tuples to skip duplicates; key is the lowercase token, value is the full sentence string  
**Source**: `heteronym_overrides.py:242, 266-269`

### Previous Choice Preservation
**Given** `existing` parameter with prior choices  
**When** regenerating overrides  
**Then** preserves previous `choice` values matched by entry ID  
**Source**: `heteronym_overrides.py:231-239`

---

## Pronunciation Store

### Storage Location
**Given** application settings  
**When** `_store_path()` resolves store location  
**Then** uses `{settings_dir}/overrides.json`; falls back to `get_internal_cache_path("pronunciations")` if settings dir unavailable  
**Source**: `pronunciation_store.py:19-25`

### Atomic Writes
**Given** any save operation  
**When** `_save_db(data)` writes to disk  
**Then** writes to `.tmp` file first, then `shutil.move()` for atomic replacement  
**Source**: `pronunciation_store.py:102-108`

### Thread Safety
**Given** concurrent access to store  
**When** any read/write operation occurs  
**Then** protected by `_DB_LOCK` (threading.RLock)  
**Source**: `pronunciation_store.py:15`

### Legacy SQLite Migration
**Given** `pronunciations.db` exists but `overrides.json` does not  
**When** `_load_db()` is first called  
**Then** migrates all entries from SQLite to JSON format, renames `.db` to `.db.bak`  
**Source**: `pronunciation_store.py:29-86`

### Load with Token Filtering
**Given** a language and token set  
**When** `load_overrides(language, tokens)` is called  
**Then** normalizes each token, filters to only matching entries in that language  
**Source**: `pronunciation_store.py:111-124`

### Search by Partial Match
**Given** a query string  
**When** `search_overrides(language, query, limit=15)` is called  
**Then** lowercases query, matches against normalized key OR original token (case-insensitive); sorts by usage_count desc, updated_at desc; returns top N  
**Source**: `pronunciation_store.py:127-148`

### Save (Upsert)
**Given** a token to override  
**When** `save_override(language=..., token=..., pronunciation=..., voice=...)` is called  
**Then** normalizes token; if exists: updates fields + updated_at; if new: creates entry with UUID, usage_count=0, timestamps; saves atomically  
**Source**: `pronunciation_store.py:151-197`

### Save Empty Token
**Given** a token that normalizes to empty string  
**When** `save_override()` is called  
**Then** raises `ValueError("Provide a token to override")`  
**Source**: `pronunciation_store.py:161-162`

### File Load Error Recovery
**Given** corrupted or unreadable JSON file  
**When** `_load_db()` fails to parse  
**Then** returns fresh `{"version": 1, "overrides": {}}`  
**Source**: `pronunciation_store.py:97-99`

---

## spaCy Utilities

### Model Mapping
**Given** a language code  
**When** loading spaCy model  
**Then** maps: a/b→en_core_web_sm, e→es_core_news_sm, f→fr_core_news_sm, i→it_core_news_sm, p→pt_core_news_sm, z→zh_core_web_sm, j→ja_core_news_sm, h→xx_sent_ud_sm  
**Source**: `spacy_utils.py` (SPACY_MODELS dict)

### Sentence Segmentation
**Given** text and language  
**When** `segment_sentences(text, language=lang)` is called  
**Then** loads appropriate spaCy model, processes text, returns list of sentence strings  
**Source**: `spacy_utils.py:segment_sentences`

### Model Loading Fallback
**Given** spaCy model not installed  
**When** loading fails  
**Then** falls back to `spacy.blank("en")` or `spacy.blank("xx")`  
**Source**: `heteronym_overrides.py:193-198` (same pattern in spacy_utils)

---

## Configuration

### spaCy Model Override
**Given** `ABOGEN_SPACY_MODEL` environment variable set  
**When** model is loaded  
**Then** uses specified model name instead of default mapping  
**Source**: `normalization_settings.py` (env var), `entity_analysis.py` (model loading)

### Language-Dependent Behavior
**Given** language code  
**When** NLP analysis runs  
**Then** heteronyms are English-only; entity extraction works with any supported spaCy model; speaker analysis uses English regex patterns — runs on any input but produces meaningful results only for English text

---

## Thread Safety

| Module | Mechanism | Protects |
|--------|-----------|----------|
| `entity_analysis.py` | `_MODEL_LOCK` (RLock) | spaCy model loading/caching |
| `pronunciation_store.py` | `_DB_LOCK` (RLock) | All file read/write operations |
| `heteronym_overrides.py` | None | Stateless per-call (no module mutable state except `_WORD_BOUNDARY_CACHE` dict — not protected) |
| `speaker_analysis.py` | None | Stateless per-call |
| `spacy_utils.py` | None | `_nlp_cache` dict NOT protected (noted gap) |

---

## Invariants

1. **Token normalization is idempotent**: `normalize_token(normalize_token(x)) == normalize_token(x)`
2. **Override keyed by normalized token**: One override per (language, normalized_token) pair
3. **Atomic persistence**: Store file never in partial-write state
4. **Heteronym set is fixed**: Only 5 words (wind, read, tear, close, lead) — no dynamic discovery
5. **Each heteronym has exactly 2 variants**: Tuple[HeteronymVariant, HeteronymVariant]
6. **Entity deduplication by key**: (normalized_text, ner_label) tuple is unique key
7. **Usage count monotonic**: Only incremented, never decremented
8. **SQLite migration is one-time**: After migration, `.db` renamed to `.db.bak`
9. **All overrides have UUID**: Generated on creation, never changes
10. **Gender inference never crashes**: Falls back to "unknown" on any ambiguity
