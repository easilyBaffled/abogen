# Spec Coverage Notes

## Files Not Covered by Behavioral Specs

These files exist in the codebase but are not primary subjects of any spec. They are either thin wrappers, legacy duplicates, or utility helpers whose behavior is implicitly covered by the specs that consume them.

### Utility/Helper (behavior implicit in consuming specs)
- `abogen/text_extractor.py` — text extraction facade; called by conversion_runner (spec 08)
- `abogen/speaker_configs.py` — speaker config persistence; used by WebUI routes (spec 10)
- `abogen/word_substitution.py` — word substitution preprocessing; used by PyQt GUI (spec 11)
- `abogen/webui/routes/utils/*.py` (8 files) — route helper utilities; used by WebUI routes (spec 10)

### Debug/Development
- `abogen/debug_tts_samples.py` — debug TTS sample builder
- `abogen/webui/debug_tts_runner.py` — debug TTS runner

### Pre-Download UI
- `abogen/predownload_gui.py` — model predownload UI (top-level)
- `abogen/pyqt/predownload_gui.py` — model predownload UI (pyqt package)

### Legacy Duplicates (top-level mirrors of pyqt/ package)
- `abogen/gui.py` — legacy, same as `pyqt/gui.py`
- `abogen/voice_formula_gui.py` — legacy, same as `pyqt/voice_formula_gui.py`
- `abogen/queue_manager_gui.py` — legacy, same as `pyqt/queue_manager_gui.py`
- `abogen/queued_item.py` — legacy, same as `pyqt/queued_item.py`

### Package Init Files (no behavioral content)
- `abogen/integrations/__init__.py`
- `abogen/webui/__init__.py`
- `abogen/webui/routes/__init__.py`
- `abogen/epub3/__init__.py`
- `abogen/pyqt/__init__.py`

## Cross-Spec Interface Notes

1. **text_extractor.py → book_parser.py**: `extract_from_path()` wraps the parser classes from spec 02; the conversion pipeline (spec 08) calls this facade.
2. **Chunk.voice_profile → voice_profiles.json**: A profile name string in the Chunk dataclass (spec 04) resolves to a voice formula dict (spec 05) during synthesis.
3. **clean_text()**: Canonical implementation in `subtitle_utils.py` (spec 04); re-exported via `utils.py` (spec 01) for convenience imports.
