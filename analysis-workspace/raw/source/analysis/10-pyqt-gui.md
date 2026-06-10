# Chunk 10: PyQt6 Desktop GUI -- Exhaustive Analysis

## 1. Module Overview

`abogen/pyqt/` implements desktop GUI using PyQt6. QWidget-based main window with drag-and-drop file input, TTS conversion via Kokoro, voice formula mixing, batch queue processing, subtitle generation, and chapter-aware book handling.

**Package entry point**: `abogen/pyqt/__init__.py`
**Application entry point**: `abogen/pyqt/main.py` -> `main()` -> `QApplication` + `abogen()` widget

---

## 2. File Inventory

| File | Lines | Role |
|------|-------|------|
| `abogen/pyqt/__init__.py` | 8 | Package marker |
| `abogen/pyqt/main.py` | 188 | App bootstrap, platform fixes, env vars |
| `abogen/pyqt/gui.py` | 4284 | Main window widget + helpers |
| `abogen/pyqt/conversion.py` | 2574 | ConversionThread, VoicePreviewThread, PlayAudioThread |
| `abogen/pyqt/book_handler.py` | 1446 | HandlerDialog for EPUB/PDF/Markdown |
| `abogen/pyqt/predownload_gui.py` | 591 | PreDownloadDialog + worker |
| `abogen/pyqt/queue_manager_gui.py` | 882 | QueueManager dialog |
| `abogen/pyqt/queued_item.py` | 29 | QueuedItem dataclass |
| `abogen/pyqt/voice_formula_gui.py` | 1599 | VoiceFormulaDialog + VoiceMixer widgets |
| **Re-exports (top-level):** | | |
| `abogen/gui.py` | 11 | `from abogen.pyqt.gui import *` |
| `abogen/predownload_gui.py` | 591 | Direct implementation (not re-export) |
| `abogen/queue_manager_gui.py` | 12 | Re-export |
| `abogen/queued_item.py` | 22 | Identical QueuedItem (no word-sub fields) |
| `abogen/voice_formula_gui.py` | 12 | Re-export |
| `abogen/debug_tts_samples.py` | 391 | Debug EPUB generator with TTS test cases |

---

## 3. All Public Classes

### 3.1 `abogen/pyqt/gui.py`

#### `DarkTitleBarEventFilter(QObject)`
Intercepts `QEvent.Type.Show` on Windows to apply dark title bar via DWM API.

#### `ShowWarningSignalEmitter(QObject)`
- **Signal**: `show_warning_signal = pyqtSignal(str, str)` -- (title, message)
- Used for thread-safe HF Hub warnings via `abogen.hf_tracker`.

#### `ThreadSafeLogSignal(QObject)`
- **Signal**: `log_signal = pyqtSignal(object)` -- carries log message.
- Connected to main window's log update slot for worker thread logging.

#### `IconProvider(QFileIconProvider)`
Trivial subclass for potential customization.

#### `InputBox(QLabel)`
Drag-and-drop file input area with visual states (default/active/error).
- Child widgets: `clear_btn`, `chapters_btn`, `textbox_btn`, `edit_btn`, `goto_folder_btn`
- Handles: `dragEnterEvent`, `dropEvent`, `mousePressEvent`, `set_active`, `set_error`, `clear_input`

#### `TextboxDialog(QDialog)`
Direct text input/editing with character count, chapter marker insertion (`<<CHAPTER_MARKER:Title>>`), voice marker insertion (`<<VOICE:name>>`).

#### `WordSubstitutionsDialog(QDialog)`
Configure word substitution rules with options: case_sensitive, replace ALL CAPS, replace numerals, fix nonstandard punctuation.

#### `abogen(QWidget)` -- MAIN APPLICATION WIDGET
Top-level window (~3700 lines). All UI controls, conversion orchestration, settings management.

### 3.2 `abogen/pyqt/conversion.py`

#### `CountdownDialog(QDialog)`
Base dialog with auto-accept countdown timer.

#### `ChapterOptionsDialog(CountdownDialog)`
Asks user how to handle detected chapters (save separately, merge at end).

#### `TimestampDetectionDialog(QDialog)`
Asks whether to use detected timestamps for subtitle timing.

#### `ConversionThread(QThread)`
- **Signals**: `progress_updated(int, str)`, `conversion_finished(object, object)`, `log_updated(object)`, `chapters_detected(int)`
- Full TTS conversion pipeline in background thread.

#### `VoicePreviewThread(QThread)`
- **Signals**: `finished()`, `error(str)`
- Generates voice preview audio, caches at `preview_cache/`.

#### `PlayAudioThread(QThread)`
- **Signals**: `finished()`, `error(str)`
- Plays audio via pygame.mixer.

### 3.3 `abogen/pyqt/book_handler.py`

#### `HandlerDialog(QDialog)`
Chapter/page selection for EPUB/PDF/Markdown files.
- **Inner class**: `_LoaderThread(QThread)` with signal `error = pyqtSignal(str)`
- **Class variables** (persist across instances): `_save_chapters_separately`, `_merge_chapters_at_end`, `_save_as_project`, `_content_cache`
- Context menu: Select all, deselect all, invert selection, select by content length

### 3.4 `abogen/pyqt/predownload_gui.py`

#### `PreDownloadWorker(QThread)`
- **Signals**: `progress(str, str, str)`, `category_done(str)`, `finished()`, `error(str)`
- Downloads: kokoro voices -> model -> config -> spaCy models sequentially.

#### `PreDownloadDialog(QDialog)`
- **Inner class**: `StatusCheckWorker(QThread)` with signals for per-category checks.
- Shows download status, start/cancel controls.

### 3.5 `abogen/pyqt/queue_manager_gui.py`

#### `ElidedLabel(QLabel)` -- Label with text elision
#### `QueueListItemWidget(QWidget)` -- Custom list item (filename + char count)
#### `DroppableQueueListWidget(QListWidget)` -- Drag-and-drop enabled list
#### `QueueManager(QDialog)` -- Full queue management with `OVERRIDE_FIELDS`, duplicate detection, context menu

### 3.6 `abogen/pyqt/queued_item.py`

#### `QueuedItem` (dataclass)
Fields: `file_name`, `lang_code`, `speed`, `voice`, `save_option`, `output_folder`, `subtitle_mode`, `output_format`, `total_char_count`, `replace_single_newlines`, `use_silent_gaps`, `subtitle_speed_method`, `save_base_path`, `save_chapters_separately`, `merge_chapters_at_end`, `word_substitutions_enabled`, `word_substitutions_list`, `case_sensitive_substitutions`, `replace_all_caps`, `replace_numerals`, `fix_nonstandard_punctuation`

### 3.7 `abogen/pyqt/voice_formula_gui.py`

#### `SaveButtonWidget(QWidget)` -- Save button for dirty profiles
#### `FlowLayout(QLayout)` -- Custom flow layout wrapping horizontally
#### `VoiceMixer(QWidget)` -- Individual voice: checkbox + slider (0-1) + spinbox + flag/gender icons
#### `HoverLabel(QLabel)` -- Label with hover-reveal delete button
#### `VoiceFormulaDialog(QDialog)` -- Voice mixing with profile CRUD, import/export, 30ms debounced updates

### 3.8 `abogen/debug_tts_samples.py`

#### `DebugTTSSample` (frozen dataclass) -- Fields: `code`, `label`, `text`
#### Functions: `marker_for(code)`, `build_debug_epub(dest_path, title=...)`, `iter_expected_codes()`
#### `DEBUG_TTS_SAMPLES` -- 50 samples across 10 categories

---

## 4. Widget Hierarchy

```
QApplication
 +-- abogen(QWidget)  [main window]
      +-- InputBox(QLabel)  [drag-drop area]
      |    +-- QPushButton "X" (clear_btn)
      |    +-- QPushButton "Chapters" (chapters_btn)
      |    +-- QPushButton "Textbox" (textbox_btn)
      |    +-- QPushButton "Edit" (edit_btn)
      |    +-- QPushButton "Go to folder" (goto_folder_btn)
      +-- QHBoxLayout [controls row 1]
      |    +-- QLabel "Speed:" + QSlider (0.5-2.0) + QLabel
      |    +-- QComboBox (voice_combo) + QPushButton "Preview"
      +-- QHBoxLayout [controls row 2]
      |    +-- QComboBox (subtitle_combo)
      |    +-- QPushButton "Word Substitutions"
      |    +-- QComboBox (format_combo) [wav/flac/mp3/opus/m4b]
      |    +-- QComboBox (subtitle_format_combo) [srt/ass]
      |    +-- QCheckBox "Replace newlines"
      +-- QHBoxLayout [save location]
      |    +-- QLabel "Save to:" + QLabel (path) + QPushButton "Browse"
      +-- QHBoxLayout [action row]
      |    +-- QCheckBox "GPU" + QPushButton (settings gear)
      |    +-- QPushButton "Start" / QPushButton "Cancel" (hidden)
      +-- QProgressBar (hidden)
      +-- QLabel (ETR)
      +-- QWidget (finish_widget, hidden)
      |    +-- QPushButton "Open" / "Go to folder" / "New" / "Back"
      +-- QHBoxLayout [queue row]
      |    +-- QPushButton "Queue" + QLabel (count)
      +-- QTextEdit (log_output, hidden by default)
```

---

## 5. QThread Communication Patterns

| Thread | Spawner | Signals | Cancel |
|--------|---------|---------|--------|
| ConversionThread | Start button | progress_updated, conversion_finished, log_updated, chapters_detected | Sets flag + kills ffmpeg |
| VoicePreviewThread | Preview button | finished, error | N/A |
| PlayAudioThread | Preview finished | finished, error | pygame.mixer stop |
| _LoaderThread | HandlerDialog open | error (+ implicit finished) | N/A |
| PreDownloadWorker | PreDownloadDialog | progress, category_done, finished, error | _cancelled flag |
| StatusCheckWorker | PreDownloadDialog | per-category result signals | N/A |
| LoadPipelineThread | First conversion | finished(pipeline) | N/A |

---

## 6. Conversion Pipeline (Desktop vs WebUI Differences)

### Desktop-only features:
- Queue system with per-item settings override
- Voice preview caching (SHA256 hash key)
- Chapter options dialog with countdown auto-accept
- Timestamp detection dialog for subtitle files
- Save chapters separately / merge options
- Progress bar with ETR (estimated time remaining)
- Finish widget (open/folder/new/back buttons)
- Dark/Light/System themes
- OS sleep prevention (`prevent_sleep_start/end`)
- HF tracker thread-safe warnings
- Word substitution system
- Subtitle speed method choice ("tts" re-generation vs "ffmpeg" time-stretch)

### Desktop conversion pipeline (`ConversionThread.run()`):
1. Input text (pre-processed by HandlerDialog/TextboxDialog)
2. Pipeline loading (shared KPipeline, loaded once)
3. Chapter splitting on `<<CHAPTER_MARKER:Title>>`
4. Voice marker splitting on `<<VOICE:voice_name>>`
5. Word substitutions (if enabled)
6. TTS generation (Kokoro `pipeline.generate()`)
7. Speed adjustment ("tts" or "ffmpeg")
8. Subtitle generation (Line/Sentence/Sentence+Comma/Sentence+Highlighting/N-word)
9. Audio encoding (wav direct, flac/mp3/opus/m4b via ffmpeg)
10. Chapter handling (save separately or merge)
11. Metadata embedding via ffmpeg
12. Sleep prevention
13. GPU acceleration (CUDA/MPS/CPU)

### Shared with WebUI:
- Kokoro pipeline usage
- Voice formula resolution
- Chapter marker parsing
- Audio format encoding
- M4B metadata embedding

### Not in Desktop (WebUI-only):
- Job queue with persistence/pause/resume
- Entity analysis and pronunciation overrides
- Heteronym detection
- Speaker analysis/multi-voice
- Audiobookshelf integration
- Calibre OPDS browsing
- EPUB3 export with SMIL
- LLM-assisted normalization
- SuperTonic TTS engine
- Text normalization pipeline (desktop uses word_substitution.py instead)

---

## 7. User-Facing Dialogs

| Dialog | Trigger | Purpose |
|--------|---------|---------|
| TextboxDialog | "Textbox"/"Edit" button | Direct text input/editing |
| WordSubstitutionsDialog | "Word Substitutions" button | Find/replace rules |
| HandlerDialog | Opening .epub/.pdf/.md | Chapter selection + preview |
| VoiceFormulaDialog | Voice combo "Voice formula..." | Mix voices with weights |
| QueueManager | "Queue" button | Batch queue management |
| PreDownloadDialog | Settings > "Pre-download models" | Offline model download |
| ChapterOptionsDialog | Auto (chapters detected) | Save separately/merge |
| TimestampDetectionDialog | Auto (.srt/.ass/.vtt input) | Use timestamps? |
| QMessageBox (About) | Settings > "About" | Version + GitHub link |
| QMessageBox (Cancel) | "Cancel" during conversion | Confirm cancellation |
| QMessageBox (Queue Summary) | After queue completes | HTML results table |

---

## 8. Settings Menu (gear button -> QMenu)

| Menu Item | Type | Action |
|-----------|------|--------|
| Theme > Dark/Light/System | QAction (radio group) | apply_theme() |
| Output formats | QAction | Configure formats |
| Max words per segment | QAction | Set word limit |
| Silence between segments | QAction | Set gap ms |
| Log lines visible | QAction | Set line count |
| Keyboard shortcuts | QAction | Show help |
| Clear preview cache | QAction | Delete cache |
| spaCy sentence segmentation | QAction (checkable) | Toggle spaCy |
| Pre-download models | QAction | Open PreDownloadDialog |
| Disable Kokoro internet | QAction (checkable) | HF_HUB_OFFLINE |
| Check for updates | QAction | GitHub version check |
| Reset all settings | QAction | Clear config |
| About | QAction | About dialog |

---

## 9. Platform-Specific Handling

| Platform | Handling | Location |
|----------|----------|----------|
| Windows | PyTorch DLL preload (c10.dll) | main.py:9-24 |
| Windows | AppUserModelID for taskbar | main.py:71-79 |
| Windows | Dark title bar via DWM API | gui.py DarkTitleBarEventFilter |
| Linux | libxcb-cursor preload (bundled .so) | main.py:50-67 |
| Linux | Wayland/GNOME QT_QPA_PLATFORM | main.py:153-162 |
| Linux | Suppress Wayland warnings | main.py qt_message_handler |
| macOS | PYTORCH_ENABLE_MPS_FALLBACK=1 | main.py:128-129 |
| macOS | MPS device in GPU checkbox | gui.py |
| All | ROCm env vars | main.py:105-106 |
| All | HF Hub env vars | main.py:93-99 |
| All | Qt plugin path for frozen apps | main.py:28-46 |

---

## 10. Core Module Imports

| Source Module | Used For |
|--------------|----------|
| `abogen.utils` | load_config, save_config, get_gpu_acceleration, prevent_sleep, get_resource_path, get_user_cache_path, LoadPipelineThread |
| `abogen.constants` | PROGRAM_NAME, VERSION, GITHUB_URL, LANGUAGE_DESCRIPTIONS, VOICES_INTERNAL, COLORS, SUBTITLE_FORMATS |
| `abogen.subtitle_utils` | clean_text, calculate_text_length |
| `abogen.book_parser` | get_book_parser (NOT text_extractor) |
| `abogen.spacy_utils` | SPACY_MODELS |
| `abogen.voice_profiles` | load_profiles |
| `abogen.hf_tracker` | show_warning_signal_emitter |
| External | PyQt6, kokoro (KPipeline), pygame, huggingface_hub, spacy, torch, numpy, soundfile, ffmpeg (subprocess) |

---

## 11. Subtitle Modes

| Mode | Behavior |
|------|----------|
| Disabled | No subtitles |
| Line | One entry per input line |
| Sentence | One per sentence (regex/spaCy) |
| Sentence+Comma | Sentence + comma boundaries |
| Sentence+Highlighting | ASS karaoke word highlighting |
| N-word | Fixed word count per entry |

---

## 12. Voice System

- **Single voice**: dropdown from `VOICES_INTERNAL`
- **Voice formula**: `"voice1*0.5+voice2*0.5"` -- tensors loaded from HF cache, blended via weighted addition
- **Profiles**: saved in config, import/export as `{"format": "abogen_voice_profiles", "profiles": {...}}`
- **Preview**: SHA256-keyed WAV cache at `preview_cache/`, played via pygame

---

## 13. Queue System

Flow: QueueManager -> add files -> QueuedItem per file -> "Start Queue" -> sequential `start_conversion(from_queue=True)` -> `queue_item_conversion_finished()` -> next or `show_queue_summary()`.

Override system: global (current settings) vs per-item (captured at add time), controlled by `OVERRIDE_FIELDS`.

---

## 14. Theme System

- **Dark**: Custom QPalette (#2b2b2b window, #1e1e1e base, #e0e0e0 text, #3399ff highlight). Windows: DwmSetWindowAttribute dark title bar.
- **Light**: Fusion style factory with default palette.
- **System**: Native OS styling.

---

## 15. Debug TTS Samples

50 frozen-dataclass samples across 10 categories: apostrophes, possessives, numbers, years, dates, currency, titles, punctuation, ALL CAPS quotes, footnotes. `build_debug_epub()` creates valid EPUB with `[[ABOGEN-DBG:CODE]]` markers for regression testing TTS pronunciation quality.
