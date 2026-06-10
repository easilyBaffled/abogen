# Behavioral Specification: Text Normalization

## Module Identity

**Paths**: `abogen/kokoro_text_normalization.py`, `abogen/normalization_settings.py`, `abogen/spacy_contraction_resolver.py`  
**Role**: Multi-phase text normalization pipeline preparing text for TTS synthesis. Handles contractions, possessives, numbers, dates, currencies, Roman numerals, acronyms, titles, terminal punctuation, and phoneme hints.  
**Boundaries**: Pure text transformation. Does NOT perform TTS, audio encoding, or job management.  
**Dependencies**: num2words (optional), spaCy (for contextual mode), LLM client (for LLM mode)

---

## Public Interface

### Primary Entry Point

#### normalize_for_pipeline(text, *, config, settings) -> str
**Given** raw text from book parsing  
**When** called during conversion  
**Then** applies full normalization pipeline in order:
1. Dates → Times → Dotted acronyms (if numbers enabled)
2. Address abbreviations (if titles enabled)
3. Internet slang (if enabled)
4. Mode-specific apostrophe handling (off/llm/spacy)
5. Number normalization (grouped numbers, ranges, fractions, currencies)
6. Title/suffix expansion (if titles enabled)
7. Terminal punctuation (if enabled)
8. All-caps quote normalization (if enabled)
9. Phoneme hints (if add_phoneme_hints enabled)

**Source**: `kokoro_text_normalization.py:2298-2365`

### Configuration

#### ApostropheConfig (frozen dataclass)
**Given** normalization settings  
**When** constructing config  
**Then** controls all normalization behaviors:

| Field | Default | Options |
|-------|---------|---------|
| `contraction_mode` | `"expand"` | expand/collapse/keep |
| `possessive_mode` | `"keep"` | keep/collapse |
| `plural_possessive_mode` | `"collapse"` | keep/collapse |
| `sibilant_possessive_mode` | `"mark"` | keep/mark/approx |
| `fantasy_mode` | `"keep"` | keep/mark/collapse_internal |
| `acronym_possessive_mode` | `"keep"` | keep/collapse_add_s |
| `decades_mode` | `"expand"` | keep/expand |
| `leading_elision_mode` | `"expand"` | keep/expand |
| `ambiguous_past_modal_mode` | `"contextual"` | keep/expand_prefer_would/expand_prefer_had/contextual |
| `convert_numbers` | `True` | bool |
| `convert_currency` | `True` | bool |
| `remove_footnotes` | `True` | bool |
| `number_lang` | `"en"` | num2words language code |
| `year_pronunciation_mode` | `"american"` | off/american |
| `add_phoneme_hints` | `True` | bool |
| `protect_cultural_names` | `True` | bool |

**Source**: `kokoro_text_normalization.py:58-93`

---

## Apostrophe Processing Modes

### Mode: "spacy" (default)
**Given** `normalization_apostrophe_mode = "spacy"`  
**When** processing text  
**Then** uses `resolve_ambiguous_contractions()` for context-dependent disambiguation of 'd/'s, then rule-based `normalize_apostrophes()` for all other contractions  
**Source**: `kokoro_text_normalization.py:2352-2353`

### Mode: "llm"
**Given** `normalization_apostrophe_mode = "llm"` with configured LLM endpoint  
**When** processing text  
**Then** sends sentences to LLM with tool-calling prompt; LLM returns regex replacements; applies replacements to text  
**Source**: `kokoro_text_normalization.py:2338-2351`

### Mode: "off"
**Given** `normalization_apostrophe_mode = "off"`  
**When** processing text  
**Then** only normalizes Unicode apostrophe variants to ASCII `'`; skips contraction expansion  
**Source**: `kokoro_text_normalization.py:2329-2337`

---

## Contraction Expansion

### Lexicon-Based Contractions
**Given** known contraction in `CONTRACTION_LEXICON`  
**When** token matches (case-insensitive)  
**Then** expands per category policy: can't→"can not", won't→"will not", don't→"do not", etc. (18 entries)  
**Source**: `kokoro_text_normalization.py:98-118`

### Suffix Contractions
**Given** token ending with 'll, 're, 've  
**When** matched  
**Then** expands: 'll→"will", 're→"are", 've→"have"; preserves base + space + expansion  
**Source**: `kokoro_text_normalization.py:120-124`

### Ambiguous 'd Resolution
**Given** pronoun + 'd (I'd, you'd, he'd, she'd, we'd, they'd)  
**When** `ambiguous_past_modal_mode` is:
- `expand_prefer_would`: expands to "would" (falls back to "had" if category disabled)
- `expand_prefer_had`: expands to "had" (falls back to "would")
- `contextual`: prefers "would" (spaCy disambiguates in spacy mode)

**Source**: `kokoro_text_normalization.py:1360-1393`

### Ambiguous 's Resolution
**Given** pronoun/demonstrative + 's (it's, that's, what's, etc.)  
**When** classifying  
**Then** prefers "is" (falls back to "has" if category disabled)  
**Source**: `kokoro_text_normalization.py:1396-1415`

### Category-Based Gating
**Given** 6 contraction categories  
**When** checking if expansion enabled  
**Then** each category independently toggleable: aux_be, aux_have, modal_will, modal_would, negation_not, let_us  
**Source**: `kokoro_text_normalization.py:46-53`

### Cultural Name Protection
**Given** tokens matching O'Brien, D'Angelo, L'anything patterns  
**When** `protect_cultural_names=True`  
**Then** apostrophe preserved regardless of mode  
**Source**: `kokoro_text_normalization.py:1302-1308, 599-604`

---

## Number Normalization

### Grouped Numbers (e.g., 12,500)
**Given** comma-separated number groups  
**When** `convert_numbers=True`  
**Then** regex matches `\d{1,3}(,\d{3})+`, strips commas, converts via num2words  
**Source**: `kokoro_text_normalization.py:131, 1630`

### Number Ranges (e.g., 10-20)
**Given** two numbers separated by dash/en-dash/em-dash  
**When** matched by `_NUMBER_RANGE_RE`  
**Then** converts to "[left] to [right]" in words  
**Source**: `kokoro_text_normalization.py:489-502`

### Fractions (e.g., 3/4)
**Given** numerator/denominator with slash  
**When** `_FRACTION_RE` matches  
**Then** reduces via `Fraction()`, converts: 1/2→"one half", 3/4→"three quarters"; denominators > 100 left unchanged  
**Source**: `kokoro_text_normalization.py:457-486`

### Decimal Numbers
**Given** number with decimal point  
**When** `_DECIMAL_NUMBER_RE` matches  
**Then** integer part spelled out, fractional digits spelled individually  
**Source**: `kokoro_text_normalization.py:340-341`

### Currency (e.g., $57,890)
**Given** `$`, `£`, `€`, or `¥` followed by amount  
**When** `convert_currency=True`  
**Then** converts to "[amount] [currency_name]" with optional magnitude suffix  
**Source**: `kokoro_text_normalization.py:148-151`

### num2words Unavailable
**Given** num2words import failed  
**When** any number conversion attempted  
**Then** returns None, number left unchanged in text  
**Source**: `kokoro_text_normalization.py:26-37, 361-373`

---

## Date Normalization

### ISO Dates (YYYY-MM-DD)
**Given** ISO format date  
**When** `_normalize_dates()` processes  
**Then** converts to spoken form; US locale: "Month ordinal, year-words"; non-US: "ordinal Month year-words"  
**Source**: `kokoro_text_normalization.py:241-270`

### Month-Day-Year Dates
**Given** "Jan. 4th, 2024" style dates  
**When** `_MDY_DATE_RE` matches  
**Then** normalizes month name, converts day to ordinal words, year to American pronunciation  
**Source**: `kokoro_text_normalization.py:272-289`

### Year Pronunciation (American)
**Given** year 2000-2099  
**When** `year_pronunciation_mode = "american"`  
**Then** special cases: 2000→"two thousand", 2001-2009→"two thousand [digit]", 2010+→split as "twenty [last-two]" with "oh" for single digits  
**Source**: `kokoro_text_normalization.py:219-238`

### Time Normalization
**Given** time like "3:30 pm" or "5 am"  
**When** `_TIME_RE` matches  
**Then** normalizes meridian to lowercase without dots; preserves hour:minute format  
**Source**: `kokoro_text_normalization.py:292-301`

---

## Roman Numeral Normalization

### Detection Rules
**Given** uppercase Roman numeral token (I, V, X, L, C, D, M)  
**When** deciding whether to convert  
**Then** converts if: (a) all-uppercase AND length ≥ 2, OR (b) has cardinal leading context (Chapter, Book, Part, etc.)  
**Source**: `kokoro_text_normalization.py:955-1020`

### Cardinal vs Ordinal Decision
**Given** Roman numeral with preceding context  
**When** `_should_render_ordinal()` evaluates  
**Then** ordinal if: preceded by Title-cased name(s) possibly with name titles (King, Pope, etc.); value ≤ 50 with title, ≤ 20 without; outputs "the [ordinal]"  
**Source**: `kokoro_text_normalization.py:909-952`

### Cardinal Context Words (37 entries)
act, appendix, article, battle, book, campaign, chapter, episode, film, final, fantasy, game, installment, lesson, level, mission, movement, opus, operation, page, part, phase, psalm, round, scene, season, section, series, song, super, bowl, stage, step, track, volume, war, world  
**Source**: `kokoro_text_normalization.py:732-771`

### Compound Patterns
**Given** token like "Chapter:IV" or "Part-III"  
**When** `_ROMAN_CONTEXT_COMPOUND_RE` matches  
**Then** splits context and roman parts, converts roman to cardinal words  
**Source**: `kokoro_text_normalization.py:970-986`

### Validation
**Given** candidate Roman numeral string  
**When** `_roman_to_int()` converts  
**Then** validates by round-tripping: `_int_to_roman(_roman_to_int(token))` must equal `token.upper()`  
**Source**: `kokoro_text_normalization.py:838-857`

### Chapter Title Batch Normalization
**Given** list of chapter titles  
**When** `normalize_roman_numeral_titles(titles, threshold=0.5)` called  
**Then** converts Roman prefixes to Arabic only if > 50% of non-empty titles have Roman prefix  
**Source**: `kokoro_text_normalization.py:1175-1233`

---

## All-Caps Quote Normalization

### Detection
**Given** text inside matched quote pairs  
**When** body is all-uppercase with > 1 letter and (multi-word or > 4 letters)  
**Then** converts to sentence case while preserving acronyms from allowlist  
**Source**: `kokoro_text_normalization.py:1082-1172`

### Acronym Allowlist (25 entries)
AI, API, CPU, DIY, GPU, HTML, HTTP, HTTPS, ID, JSON, MP3, MP4, M4B, NASA, OCR, PDF, SQL, TV, TTS, UK, UN, UFO, OK, URL, USA, US, VR  
**Source**: `kokoro_text_normalization.py:1023-1051`

### Roman Numeral Preservation
**Given** all-caps word consisting only of I, V, X, L, C, D, M  
**When** normalizing caps segment  
**Then** word preserved in uppercase (not sentence-cased)  
**Source**: `kokoro_text_normalization.py:1074-1079`

---

## Title and Suffix Expansion

### Title Abbreviations
Mr→mister, Mrs→missus, Ms→miz, Dr→doctor, Prof→professor, Rev→reverend, Gen→general, Sgt→sergeant  
**Source**: `kokoro_text_normalization.py:624-633`

### Suffix Abbreviations
Jr→junior, Sr→senior  
**Source**: `kokoro_text_normalization.py:635-638`

### Address Abbreviations
St→street, Rd→road, Ave→avenue, Blvd→boulevard, Ln→lane, Dr→drive  
Only at end of clause (followed by `,`, `.`, `!`, `?`, or EOL)  
**Source**: `kokoro_text_normalization.py:304-321`

### Case Preservation
**Given** abbreviation in original casing  
**When** expanding  
**Then** `_match_casing()`: ALL CAPS→upper, Title Case→capitalize, else→lowercase  
**Source**: `kokoro_text_normalization.py:1236-1244`

---

## Terminal Punctuation

### Missing Punctuation Insertion
**Given** text line not ending with terminal punctuation  
**When** `ensure_terminal_punctuation()` called  
**Then** appends `.` if last char not in {`.`, `?`, `!`, `…`, `;`, `:`}  
**Source**: `kokoro_text_normalization.py:1260-1299`

### Closing Punctuation Awareness
**Given** line ending with closing quotes/brackets  
**When** checking terminal punctuation  
**Then** looks BEFORE closers (``"'"')]}»›``) for actual terminal punctuation  
**Source**: `kokoro_text_normalization.py:1268-1271`

### Ellipsis Preservation
**Given** line ending with `...` or `…`  
**When** checking  
**Then** treated as terminal; no period added  
**Source**: `kokoro_text_normalization.py:1279-1280`

---

## Phoneme Hints

### apply_phoneme_hints(text, iz_marker="‹IZ›") -> str
**Given** normalized text with possessives  
**When** called  
**Then** inserts phoneme markers for sibilant possessive pronunciation (/ɪz/ insertion)  
**Source**: `kokoro_text_normalization.py:1997`

---

## LLM Integration

### Prompt Template
**Given** LLM mode enabled  
**When** building prompt  
**Then** uses Mustache-style `{{ sentence }}` and `{{ paragraph }}` interpolation; default prompt asks for regex replacements via tool calling  
**Source**: `normalization_settings.py:15-27`

### Sentence Splitting for LLM
**Given** text to normalize  
**When** splitting for LLM calls  
**Then** splits on sentence boundaries (`.`, `!`, `?` followed by space/newline)  
**Source**: `kokoro_text_normalization.py:2089-2093`

### LLM Response Parsing
**Given** LLM completion  
**When** extracting replacements  
**Then** parses tool_calls for regex patterns + replacements; applies each via `re.sub()`  
**Source**: `kokoro_text_normalization.py:2169-2298`

### LLM Error Propagation
**Given** LLM call fails  
**When** `LLMClientError` raised  
**Then** re-raised to caller (not swallowed)  
**Source**: `kokoro_text_normalization.py:2343-2344`

---

## Normalization Settings

### Settings Resolution Priority
1. Environment variables (ABOGEN_LLM_BASE_URL, etc.)
2. config.json values
3. `_SETTINGS_DEFAULTS` hardcoded defaults

**Source**: `normalization_settings.py:68-75, 86-100`

### Sample Texts for Preview
4 samples: apostrophes, numbers, titles, punctuation — used in UI settings preview  
**Source**: `normalization_settings.py:77-82`

---

## Spacing Cleanup

### _cleanup_spacing(text) -> str
**Given** normalized text with potential spacing artifacts  
**When** called as final cleanup  
**Then** removes: BOM/zero-width chars, spaces before closing punctuation, spaces after opening punctuation; ensures spaces after sentence punctuation; tightens hyphen/dash spacing; collapses multiple spaces  
**Source**: `kokoro_text_normalization.py:675-698`

---

## Utility Functions

### normalize_unicode_apostrophes(text) -> str
Normalizes NFKC, replaces all apostrophe variants (`, ´, ꞌ, ʼ, ') with ASCII `'`  
**Source**: `kokoro_text_normalization.py:656-660`

### tokenize(text) -> List[str]
Splits into word tokens (letters/digits/apostrophes/hyphens) and single-char punctuation tokens  
**Source**: `kokoro_text_normalization.py:663-665`

### tokenize_with_spans(text) -> List[(str, int, int)]
Same as tokenize but returns (token, start, end) tuples for position-preserving replacement  
**Source**: `kokoro_text_normalization.py:668-672`

---

## Error Behaviors

| Condition | Behavior |
|-----------|----------|
| num2words unavailable | Number conversion disabled; numbers left as digits |
| spaCy unavailable | Falls back to rule-based disambiguation |
| LLM endpoint unreachable | `LLMClientError` raised |
| LLM returns invalid JSON | Graceful parse failure; sentence unchanged |
| Invalid regex from LLM | `_apply_single_regex_replacement` catches `re.error`, skips |
| Empty text input | Returns empty string |

---

## Thread Safety

Module-level state is all immutable (compiled regex, frozen dicts, constants). No mutable module state. Thread-safe for concurrent use.

---

## Invariants

1. **Pipeline order fixed**: Dates/times before numbers; titles after apostrophes; phoneme hints last
2. **num2words optional**: All number features degrade gracefully without it
3. **Case always preserved**: `_match_casing()` and `_case_preserving_words()` maintain original casing pattern
4. **Cultural names never modified**: O'Brien, D'Angelo always protected when `protect_cultural_names=True`
5. **Contractions category-gated**: Each expansion requires its category enabled
6. **Unicode apostrophes normalized early**: Before any matching occurs
7. **Spacing cleaned once**: `_cleanup_spacing()` applied after all transformations
8. **Roman numeral round-trip validation**: Only valid numerals converted
9. **Threshold gating for batch titles**: Roman prefix conversion only if > 50% prevalence
10. **LLM errors propagate**: Not silently swallowed; caller decides recovery
