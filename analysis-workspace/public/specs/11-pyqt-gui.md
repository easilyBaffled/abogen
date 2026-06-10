# Behavioral Specification: PyQt6 Desktop GUI

## Module Identity

**Paths**: `abogen/pyqt/gui.py`, `abogen/pyqt/conversion.py`, `abogen/pyqt/book_handler.py`, `abogen/pyqt/queue_manager_gui.py`, `abogen/pyqt/queued_item.py`  
**Role**: PyQt6 desktop application providing drag-and-drop book conversion with queue management, voice selection, and real-time progress  
**Boundaries**: GUI layer only. Delegates to: book_parser (parsing), voice_formulas (blending), KPipeline (TTS), ffmpeg (encoding). Shares no state with WebUI.  
**Framework**: PyQt6 (QWidget-based, not QMainWindow)

---

## Public Interface

### Main Widget: `abogen` (QWidget)
**Given** application launched in desktop mode  
**When** `abogen` widget instantiated  
**Then** creates full UI: input box (drag-drop), voice selector, format options, speed slider, progress bar, control buttons, queue panel  
**Source**: `gui.py:910, 1046`

### Entry Point
**Given** CLI invocation without `--web`  
**When** `main.py` runs  
**Then** creates QApplication, instantiates `abogen` widget, shows window  
**Source**: `pyqt/main.py`

---

## Input Handling

### Drag-and-Drop File Input
**Given** `InputBox` widget  
**When** file dragged over  
**Then** accepts: .epub, .pdf, .txt, .md, .markdown, .srt, .vtt, .ass (subtitle resume); visual highlight on dragEnter; parses on drop  
**Source**: `gui.py:423-518`

### File Dialog Input
**Given** user clicks file open button  
**When** `open_file_dialog()` called  
**Then** shows native file picker filtered to supported formats  
**Source**: `gui.py:1489`

### Text Editor Input
**Given** user clicks text edit button  
**When** `TextboxDialog` opened  
**Then** provides multi-line editor with character count, chapter marker insertion, voice marker insertion, save-as-text, word substitutions  
**Source**: `gui.py:624-806`

### File Info Display
**Given** file loaded  
**When** `InputBox.set_file_info(path)` called  
**Then** shows: filename, file type icon, file size (human readable), chapter count, total character count, estimated duration  
**Source**: `gui.py:227-360`

---

## Book Processing

### HandlerDialog
**Given** a book file to parse  
**When** `HandlerDialog` invoked  
**Then** runs parser in background; shows progress dialog; extracts chapters + metadata; populates chapter selection  
**Source**: `book_handler.py`

### Chapter Selection
**Given** parsed book with chapters  
**When** user interacts with chapter list  
**Then** can select/deselect individual chapters; select all/none; reorder; see per-chapter character counts  
**Source**: `gui.py:519`

### Chapter Options Dialog
**Given** chapters selected  
**When** `ChapterOptionsDialog` opened  
**Then** allows: rename chapters, set per-chapter voice overrides, configure chapter merge behavior  
**Source**: `conversion.py:ChapterOptionsDialog`

---

## Voice System (GUI)

### Voice Profile Combo
**Given** voice profiles loaded  
**When** combo populated  
**Then** lists all saved profiles from `voice_profiles.json`; includes "Custom Formula" option  
**Source**: `gui.py:1866`

### Voice Formula GUI
**Given** user selects "Custom Formula"  
**When** formula editor opened  
**Then** provides: voice picker (52 voices), weight sliders, formula text field, preview button  
**Source**: `voice_formula_gui.py`

### Voice Preview
**Given** voice formula set  
**When** preview button clicked  
**Then** `VoicePreviewThread` synthesizes short sample; plays via system audio  
**Source**: `conversion.py:VoicePreviewThread`

### Profile Selection Change
**Given** profile selected  
**When** `on_voice_combo_changed()` fires  
**Then** updates subtitle format options based on profile's provider (Kokoro vs SuperTonic have different format support)  
**Source**: `gui.py:1826-1857`

---

## Conversion Pipeline

### ConversionThread (QThread)
**Given** conversion parameters assembled  
**When** thread started  
**Then** runs: load pipeline → resolve voice → iterate chapters → TTS synthesis → audio encoding → subtitle writing; emits progress/log/finished signals  
**Source**: `conversion.py:ConversionThread`

### Progress Reporting
**Given** RUNNING conversion  
**When** segments processed  
**Then** emits `progress(int, str)` signal with percentage + ETR string; updates progress bar + ETR label  
**Source**: `gui.py:1968-1995`

### Log Display
**Given** conversion events  
**When** `ThreadSafeLogSignal` emits  
**Then** appends to log text area (replaces input box during conversion); auto-scrolls; signal marshaled to main thread  
**Source**: `gui.py:122-128, 1912-1950`

### Cancellation
**Given** RUNNING conversion  
**When** cancel button clicked  
**Then** sets cancel flag; checked at segment boundaries; thread terminates; UI restored  
**Source**: `gui.py`, `conversion.py`

### Sleep Prevention
**Given** conversion starts  
**When** `prevent_sleep_start()` called  
**Then** prevents system sleep; `prevent_sleep_end()` called on completion/cancel  
**Source**: `gui.py` (uses `utils.prevent_sleep_start/end`)

---

## Queue Management

### QueueManager
**Given** multiple files to convert  
**When** items added to queue  
**Then** `QueueManager` processes items sequentially (single-threaded); tracks: pending, current, completed, failed  
**Source**: `queue_manager_gui.py`

### QueuedItem (dataclass)
**Given** conversion parameters  
**When** item created  
**Then** stores: file_path, chapters, voice settings, output format, speed, subtitle options  
**Source**: `queued_item.py`

### Queue Controls
**Given** queue with items  
**When** user interacts  
**Then** can: add current file to queue, clear queue, manage queue (reorder/remove), view queue progress  
**Source**: `gui.py:2024-2120`

### Queue Progress Format
**Given** queue processing  
**When** progress displayed  
**Then** shows `"[current/total] filename - X%"` format  
**Source**: `gui.py:1953-1967`

---

## Output Configuration

### Format Selection
| Format | Extension | Encoder |
|--------|-----------|---------|
| WAV | .wav | Direct |
| FLAC | .flac | Direct |
| MP3 | .mp3 | ffmpeg |
| Opus | .opus | ffmpeg |
| M4B | .m4b | ffmpeg (forces merge) |

### Subtitle Options
| Format | Extension |
|--------|-----------|
| SRT | .srt |
| VTT | .vtt |
| ASS | .ass |
| None | — |

### Speed Control
**Given** speed slider  
**When** adjusted  
**Then** range 0.5x-2.0x; label updates with current value; affects TTS `speed` parameter  
**Source**: `gui.py:1756-1760`

### Merge Chapters Option
**Given** multi-chapter book  
**When** merge enabled  
**Then** produces single output file; M4B format forces this on  
**Source**: GUI checkbox + `conversion.py`

---

## UI Components

### Dark Mode / Title Bar
**Given** Windows platform  
**When** `DarkTitleBarEventFilter` active  
**Then** applies dark title bar styling on `QEvent.Type.Show` events  
**Source**: `gui.py:99-113`

### InputBox States
1. **Empty**: shows drag-drop prompt with dashed border
2. **File loaded**: shows file info, chapters button, edit button, folder button
3. **Converting**: replaced by log output area
4. **Error**: shows error message with red styling

### Word Substitutions Dialog
**Given** text preprocessing needed  
**When** dialog opened  
**Then** provides: substitution list (find/replace pairs), case sensitive toggle, replace all-caps toggle, replace numerals toggle, fix nonstandard punctuation toggle  
**Source**: `gui.py:808-905`

### Timestamp Detection Dialog
**Given** subtitle file loaded for resume  
**When** `TimestampDetectionDialog` shown  
**Then** detects last timestamp in subtitle file; offers to resume from that position  
**Source**: `conversion.py:TimestampDetectionDialog`

---

## Thread Architecture

| Thread | Type | Purpose |
|--------|------|---------|
| Main (GUI) | QApplication event loop | All UI updates |
| `ConversionThread` | QThread | TTS synthesis + encoding |
| `VoicePreviewThread` | QThread | Short voice sample synthesis |
| `PlayAudioThread` | QThread | Audio playback |
| `LoadPipelineThread` | QThread | Background model loading |

### Signal-Slot Communication
All cross-thread communication via Qt signals:
- `progress(int, str)` — progress bar update
- `finished(bool, str)` — conversion complete
- `log_signal(str)` — log message
- `ShowWarningSignalEmitter.show_warning_signal(str, str)` — thread-safe warning dialog

**Source**: `gui.py:115-128`, `conversion.py`

---

## Configuration Integration

### Config Load on Start
**Given** application starts  
**When** `initUI()` called  
**Then** reads `config.json` for: last voice profile, last output format, last speed, last subtitle format, window geometry  
**Source**: `gui.py:1046`

### Config Save on Change
**Given** user changes settings  
**When** voice/format/speed changed  
**Then** persists to `config.json` via `save_config()`  
**Source**: `gui.py`

### Output Directory
**Given** conversion output needed  
**When** determining output path  
**Then** uses: (1) config `output_dir` if set, (2) `get_user_output_root()` default  
**Source**: `gui.py:546+`

### Go To Folder
**Given** output exists  
**When** folder button clicked  
**Then** opens native file manager at output directory via `QDesktopServices.openUrl()`  
**Source**: `gui.py:546-622`

---

## Platform-Specific Behaviors

### Windows
- Dark title bar event filter active
- `CREATE_NO_WINDOW` flag for subprocess (ffmpeg)
- Native file dialog style

### macOS
- System menu bar integration
- `caffeinate` for sleep prevention
- Native drag-drop appearance

### Linux
- `systemd-inhibit` for sleep prevention
- GTK file dialog (if available)

---

## Error Handling

### Conversion Failure
**Given** exception during ConversionThread  
**When** thread catches exception  
**Then** emits `finished(False, error_message)`; UI shows error state; log shows traceback  
**Source**: `conversion.py`

### File Parse Failure
**Given** unsupported or corrupt file  
**When** book parser raises  
**Then** `InputBox.set_error(message)` shows error in red; user can retry  
**Source**: `gui.py:361-370`

### Missing ffmpeg
**Given** MP3/Opus/M4B format selected  
**When** ffmpeg not found  
**Then** conversion fails with clear error message about missing ffmpeg  
**Source**: `conversion.py`

---

## Invariants

1. **Single conversion at a time**: Queue enforces sequential processing
2. **UI updates on main thread only**: All signal handlers execute on GUI thread
3. **Config never lost**: `save_config()` preserves keys not related to current change
4. **Progress 0-100**: Integer percentage, never exceeds 100
5. **Thread cleanup guaranteed**: ConversionThread always emits `finished` signal
6. **Input box state machine**: Empty→Loaded→Converting→(Loaded|Empty) — no invalid transitions
7. **Sleep paired**: `prevent_sleep_start` always matched with `end` via finally/finished handler
8. **File handles closed**: Book parsers closed after chapter extraction (context manager pattern)
9. **Queue persists across files**: Adding new file doesn't clear existing queue
10. **Platform detection at startup**: All platform-specific setup done once in `initUI()`
