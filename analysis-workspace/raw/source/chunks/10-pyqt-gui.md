# Chunk 10: pyqt-gui

## Files
- `abogen/pyqt/__init__.py`
- `abogen/pyqt/main.py`
- `abogen/pyqt/gui.py`
- `abogen/pyqt/book_handler.py`
- `abogen/pyqt/conversion.py`
- `abogen/pyqt/predownload_gui.py`
- `abogen/pyqt/queue_manager_gui.py`
- `abogen/pyqt/queued_item.py`
- `abogen/pyqt/voice_formula_gui.py`
- `abogen/gui.py`
- `abogen/predownload_gui.py`
- `abogen/queue_manager_gui.py`
- `abogen/queued_item.py`
- `abogen/voice_formula_gui.py`
- `abogen/debug_tts_samples.py`

## Description
PyQt6 desktop GUI (legacy/alternative interface). `pyqt/main.py` bootstraps Qt, loads DLLs on Windows, finds platform plugins. `pyqt/gui.py` is the main window. `pyqt/conversion.py` runs TTS in a QThread using book_parser, voice_formulas, and Kokoro pipeline. `pyqt/book_handler.py` loads/processes books. Other modules provide predownload, queue management, and voice formula editing dialogs. Top-level files (`gui.py`, `predownload_gui.py`, etc.) appear to be legacy versions or shared code.

## Internal Relationships
- Imports from `book_parser` (not text_extractor), `subtitle_utils`, `voice_formulas`, `constants`, `utils`, `hf_tracker`, `spacy_utils`
- Does not use the web service layer
