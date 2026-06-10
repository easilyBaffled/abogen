# Text Normalization Chunk Analysis

## File Inventory

| File | Lines | Primary Role |
|------|-------|--------------|
| `kokoro_text_normalization.py` | 2379 | Core normalization engine: apostrophes, numbers, dates, times, addresses, Roman numerals, caps, LLM integration |
| `normalization_settings.py` | 247 | Configuration layer: settings extraction, ApostropheConfig construction, LLM configuration builder |
| `spacy_contraction_resolver.py` | 266 | spaCy-based disambiguation of ambiguous `'s` and `'d` contractions |
| `word_substitution.py` | 255 | Marker-preserving word/phrase replacement, caps lowering, numeral conversion, punctuation normalization |
| `heteronym_overrides.py` | 305 | Heteronym detection and replacement tokens for TTS disambiguation (wind, read, tear, close, lead) |

---

## 1. Pipeline Order (normalize_for_pipeline)

Source: `kokoro_text_normalization.py`

The main entry point is `normalize_for_pipeline(text, config, settings)`. The execution order is:

### Phase A: Settings Resolution
1. `get_runtime_settings()` loads from config file + environment variables
2. `build_apostrophe_config(settings, base)` maps flat settings dict into `ApostropheConfig` dataclass

### Phase B: Pre-normalization (before apostrophe/number parsing)
3. If `normalization_numbers` is True:
   - `_normalize_dates(text, language)` -- ISO and MDY date expansion
   - `_normalize_times(text)` -- time format normalization
   - `_normalize_dotted_acronyms(text)` -- collapse "U.S.A." to "USA"
4. If `normalization_titles` is True:
   - `_normalize_address_abbreviations(text)` -- St./Rd./Ave./etc. expansion
5. If `normalization_internet_slang` is True:
   - `_normalize_internet_slang(text)` -- pls/plz to please

### Phase C: Core Normalization (mode-dependent)
Mode is read from `normalization_apostrophe_mode` setting (default: "spacy").

- **Mode "off"**: Unicode apostrophe normalization + grouped numbers only
- **Mode "llm"**: Calls `_normalize_with_llm()` which sends each sentence to an LLM for regex-based replacements
- **Mode "spacy" (default)**: Calls `normalize_apostrophes(text, cfg)` which uses rule-based classification + spaCy contextual disambiguation

### Phase D: Post-normalization
6. If `normalization_titles` is True:
   - `expand_titles_and_suffixes(text)` -- Mr./Mrs./Dr. etc. expansion
7. If `normalization_terminal` is True:
   - `ensure_terminal_punctuation(text)` -- appends "." to segments lacking terminal punctuation
8. If `normalization_caps_quotes` is True:
   - `_normalize_all_caps_quotes(text)` -- lowercases ALL-CAPS text within quotation marks
9. If `add_phoneme_hints` is True:
   - `apply_phoneme_hints(text)` -- replaces marker like `IZ` with " iz"

---

## 2. Normalization Rules and Transformations

### 2.1 Apostrophe/Contraction Processing (classify_token)

Token classification priority (returns on first match):

| Priority | Category | Pattern | Default Action |
|----------|----------|---------|----------------|
| 1 | `decade` | `'\d0s` (e.g., '90s) | expand -> "nineties" |
| 2 | `leading_elision` | 'tis, 'twas, 'cause, 'em, 'round, 'til | expand to full word |
| 3 | `contraction_modal_would` / `contraction_aux_have` | ambiguous 'd (I'd, you'd, he'd, she'd, we'd, they'd) | contextual (spaCy-resolved or prefer "would") |
| 4 | `contraction_aux_be` / `contraction_aux_have` | ambiguous 's (it's, that's, what's, where's, who's, when's, how's, there's, here's, he's, she's, we's, they's, you's) | contextual (spaCy-resolved or prefer "is") |
| 5 | lexicon contraction | 18 entries: let's, can't, won't, don't, doesn't, didn't, isn't, aren't, wasn't, weren't, haven't, hasn't, hadn't, couldn't, shouldn't, wouldn't, mustn't, mightn't, shan't | expand to full form |
| 6 | 'm suffix | I'm only | expand -> "I am" |
| 7 | suffix contractions | 'll -> will, 're -> are, 've -> have | expand per category enablement |
| 8 | `irregular_possessive` | children's, men's, women's, people's, geese's, mouse's | keep unchanged |
| 9 | `plural_possessive` | `\w+s'` pattern | collapse (remove trailing apostrophe) |
| 10 | `acronym_possessive` | `[A-Z]{2,}'s` | keep |
| 11 | `sibilant_possessive` | word ending in s/x/z/ch/sh + 's | mark with IZ marker |
| 12 | `singular_possessive` | generic `\w+'s` | keep |
| 13 | `cultural_name` | O'Brien, D'Angelo, L'anything patterns | keep (protected) |
| 14 | `fantasy_internal` | apostrophe between letters (Ta'veren) | keep/mark/collapse per config |
| 15 | `other` | anything else with apostrophe | keep or collapse |

### 2.2 Number Normalization

Order of number replacement:
1. URL replacement
2. Footnote removal
3. Currency conversion
4. Number ranges with separators -- "100-200" -> "one hundred to two hundred"
5. Space-separated ranges
6. Fractions -- "3/4" -> "three quarters"
7. Decimal numbers -- "3.14" -> "three point one four"
8. Grouped numbers with commas -- "12,500" -> "twelve thousand five hundred"
9. Plain integers
10. Roman numerals

### 2.3 Year Pronunciation (American Mode)

Rules for 4-digit numbers detected as year-like:
- 2000 -> "two thousand"
- 2001-2009 -> "two thousand [digit]"
- 1100-1999 -> "[first-two] hundred [last-two]" with "oh" for single digits (e.g., 1905 -> "nineteen oh five")
- 2010-2099 -> "[first-two] [last-two]" (e.g., 2024 -> "twenty twenty-four")
- X000 -> "[X] thousand"

Context check: If "address" appears within 60 chars and no year marker (bc/ad/bce/ce) is present, year pronunciation is suppressed.

### 2.4 Date Normalization

- ISO format YYYY-MM-DD or YYYY/MM/DD -> "Month ordinal, year-words" (US) or "ordinal Month year-words" (non-US)
- MDY format "Jan 4th, 2024" -> same spoken form
- US locale detection via LC_ALL, LC_TIME, LANG environment variables

### 2.5 Roman Numeral Normalization

- Validates that the numeral round-trips (parse -> compose matches original)
- Cardinal context words (80+ entries): act, chapter, volume, part, episode, season, etc.
- Ordinal rendering for name-like sequences: "King Henry VIII" -> "King Henry the eighth" (values <= 50 with title, <= 20 with 2+ titlecase words)
- Compound patterns: "Chapter-IV" detected via regex

### 2.6 Currency Conversion

- Pattern: `[$GBP EUR JPY]\s*amount(\s+magnitude)?`
- Sub-unit handling: $0.99 -> "ninety-nine cents"
- Magnitude support: "$2.5 million" -> "two point five million dollars"
- Uses num2words with `to="currency"` for standard amounts

---

## 3. Configuration Options and Effects

### 3.1 ApostropheConfig Dataclass

| Field | Default | Effect |
|-------|---------|--------|
| `contraction_mode` | "expand" | expand/collapse/keep contractions |
| `possessive_mode` | "keep" | keep/collapse singular possessives |
| `plural_possessive_mode` | "collapse" | collapse removes trailing apostrophe |
| `sibilant_possessive_mode` | "mark" | mark inserts IZ phoneme hint |
| `fantasy_mode` | "keep" | keep/mark/collapse_internal for fantasy apostrophes |
| `acronym_possessive_mode` | "keep" | keep/collapse_add_s |
| `decades_mode` | "expand" | expand '90s -> nineties |
| `leading_elision_mode` | "expand" | expand 'tis -> it is |
| `ambiguous_past_modal_mode` | "contextual" | keep/expand_prefer_would/expand_prefer_had/contextual |
| `add_phoneme_hints` | True | emit markers like IZ |
| `protect_cultural_names` | True | always keep O'Brien, D'Angelo, etc. |
| `convert_numbers` | True | grouped numbers to words |
| `convert_currency` | True | currency symbols to words |
| `remove_footnotes` | True | strip footnote indicators |
| `number_lang` | "en" | num2words language code |
| `year_pronunciation_mode` | "american" | off/american |
| `contraction_categories` | all True | per-category enable/disable |

### 3.2 Environment Variable Overrides

| Setting Key | Environment Variable |
|-------------|---------------------|
| `llm_base_url` | `ABOGEN_LLM_BASE_URL` |
| `llm_api_key` | `ABOGEN_LLM_API_KEY` |
| `llm_model` | `ABOGEN_LLM_MODEL` |
| `llm_timeout` | `ABOGEN_LLM_TIMEOUT` |
| `llm_prompt` | `ABOGEN_LLM_PROMPT` |
| `llm_context_mode` | `ABOGEN_LLM_CONTEXT_MODE` |
| spaCy model | `ABOGEN_SPACY_MODEL` (default: en_core_web_sm) |

---

## 4. LLM Integration Points

Activated when `normalization_apostrophe_mode == "llm"`.

Flow:
1. Text split into lines, then sentences via regex `[^.!?]+[.!?]+|[^.!?]+$`
2. For each sentence, mustache template rendered with `{{ sentence }}` and `{{ paragraph }}`
3. LLM completion requested with system prompt + forced tool use
4. Tool: `apply_regex_replacements` returns ordered regex substitutions
5. Each substitution applied sequentially; allowed flags: IGNORECASE, MULTILINE, DOTALL
6. Legacy prompt auto-migrated to new format

---

## 5. Edge Cases and Special Handling

1. **spaCy graceful degradation**: If unavailable, returns empty dict; pipeline falls back to rule-based defaults
2. **num2words graceful degradation**: If unavailable, numbers left as-is
3. **Contiguous token requirement**: spaCy resolution only for tokens immediately adjacent (no whitespace)
4. **Year vs address ambiguity**: 60-char context window check suppresses year pronunciation near "address"
5. **Decimal number protection**: Numbers adjacent to "." followed by digit are not converted
6. **URL vs number ambiguity**: Numeric-only "domains" checked via float() to avoid matching decimals
7. **Fraction simplification**: Reduced via `Fraction()` before spelling; denominators > 100 not converted
8. **Roman numeral validation**: Must round-trip (parse -> compose = original)
9. **Heteronym replacement tokens**: Phonetic spellings ("wynd", "red", "tier", "cloze", "led") that TTS pronounces correctly
10. **Word substitution marker preservation**: Chapter/voice/metadata/timestamp markers are never modified

---

## 6. Dependencies Between Modules

```
normalize_for_pipeline (kokoro_text_normalization.py)
  +-- normalization_settings.py (build_apostrophe_config, get_runtime_settings, build_llm_configuration)
  +-- spacy_contraction_resolver.py (resolve_ambiguous_contractions, called from normalize_apostrophes)
  +-- llm_client.py (external, called from _normalize_with_llm)
  +-- num2words (external optional)
  +-- spacy (external optional)

word_substitution.py [separate pipeline stage, not called from normalize_for_pipeline]
  +-- subtitle_utils.py (marker patterns)
  +-- num2words (optional)

heteronym_overrides.py [separate pipeline stage, not called from normalize_for_pipeline]
  +-- spacy (optional)
```

---

## 7. Contraction Category System

Six independently toggleable categories:
1. `contraction_aux_be` -- I'm, 're, 's (is)
2. `contraction_aux_have` -- 've, 's (has), 'd (had)
3. `contraction_modal_will` -- 'll
4. `contraction_modal_would` -- 'd (would)
5. `contraction_negation_not` -- all n't contractions
6. `contraction_let_us` -- let's

Each can be individually disabled. When disabled, the contraction is left as-is even if contraction_mode is "expand".
