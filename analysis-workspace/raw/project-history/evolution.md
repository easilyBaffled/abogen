# Git Archaeology: abogen Evolution

## Timeline Summary

- **2025-04-24**: Initial commit (desktop-only GUI app with Kokoro TTS)
- **2025-05-03 to 2025-07**: Queue manager, voice formulas, voice mixer, drag & drop
- **2025-09 to 2025-10**: Web UI transition (Flask), legacy UI removal
- **2025-11**: Integrations (Audiobookshelf, Calibre OPDS), normalization settings
- **2025-12**: Supertonic TTS, EPUB3 export, spaCy, heteronyms, debug TTS, WebUI merge into main
- **2026-01**: PyQt + WebUI additive merge, book_parser extraction
- **2026-02**: Voice markers, word substitution
- **2026-05**: Supertonic 3 support, GPU acceleration
- **2026-04-30**: Latest commit (CPU fallback)

## Contributors

| Commits | Author |
|---------|--------|
| 331 | Deniz Şafak (project creator) |
| 249 | JB (major contributor, WebUI/integrations) |
| 41 | Juraj Borza (Supertonic TTS, voice features) |
| 9 | copilot-swe-agent[bot] |
| 2 | Mohan Krishnan |
| 2 | olandir (voice tags) |
| 1 | Various community contributors |

## Major Refactors

1. **2025-10-07**: Remove legacy UI, transition to straight webapp
2. **2025-12-13**: Extract subtitle_utils from conversion.py
3. **2025-12-21**: Move web-related components to webui/ module
4. **2025-12-22**: Extract book_parser.py from book_handler.py
5. **2026-01-06**: Additive merge of webui branch with PyQt GUI support (dual-GUI architecture)
6. **2026-01-08**: Further refactoring of book_handler logic into PDFParser
7. **2025-12-02**: Refactor pronunciation storage from SQLite to JSON

## Feature Introductions

| Date | Feature |
|------|---------|
| 2025-04 | Core TTS with Kokoro engine |
| 2025-05 | Voice formulas for mixed voices |
| 2025-06 | Queue manager |
| 2025-07 | Voice mixer GUI, drag & drop |
| 2025-08 | spaCy sentence segmentation |
| 2025-09 | Conversion service with job management |
| 2025-10 | Flask web UI, chapter overrides |
| 2025-11 | Audiobookshelf + Calibre OPDS integration |
| 2025-11 | LLM client for text normalization |
| 2025-11 | Voice fallback logic, voice asset caching |
| 2025-12 | Supertonic TTS provider |
| 2025-12 | Heteronym handling |
| 2025-12 | Debug TTS samples (WAV artifacts) |
| 2025-12 | EPUB 3 media-overlay export |
| 2025-12 | Normalization settings UI |
| 2026-01 | Dual GUI (PyQt + Web) merged |
| 2026-02 | Voice markers, word substitution |
| 2026-05 | Supertonic 3, GPU acceleration |

## Removed Features

1. **Speaker mode handling** (2025-10-12): Removed from various components
2. **Legacy desktop UI** (2025-10-07): Replaced by Flask webapp
3. **SQLite pronunciation store** (2025-12-02): Replaced by JSON storage
4. **TensorRT dependency** (2025-12-21): Removed, CUDA prioritized
5. **Performance tests** (2025-11-22): Deleted along with PERFORMANCE_OPTIMIZATIONS.md
6. **`abogen_` prefix** on output files (2025-05-21): Removed

## Configuration Evolution

- **Early**: Minimal config, GUI-only settings
- **2025-10**: Docker GPU support added
- **2025-11**: Integration settings (Audiobookshelf URL/API key, Calibre OPDS URL)
- **2025-11**: Normalization settings (currency, footnotes, internet slang)
- **2025-12**: Network mode config for Docker, debug TTS settings
- **2025-12**: Supertonic GPU config (CUDA/ONNX providers)
- **2026-01**: Environment variable fallbacks for integration settings

## Architectural Decisions (from git history)

1. **Dual-GUI architecture**: PyQt desktop app AND Flask web UI coexist, sharing core logic
2. **Plugin-style TTS**: Kokoro (default) + Supertonic (alternative) as separate providers
3. **JSON over SQLite**: Pronunciation store migrated for simplicity
4. **Hatchling build system**: Early switch from setuptools
5. **HTMX-driven web UI**: Server-rendered with live partial updates
6. **Job-based conversion**: Async conversion with logging and status tracking

## Commit Message Conventions

- feat: prefix for new features
- fix: prefix for bug fixes
- refactor: prefix for restructuring
- Merge PR format: "Merge pull request #N from user/branch"
- Descriptive messages without conventional commits also common (especially from main author)
