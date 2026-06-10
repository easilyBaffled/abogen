# Behavioral Claims Extracted from Documentation

## Source: README.md

### Application Purpose and Core Capabilities
- Abogen is a text-to-speech conversion tool that turns ePub, PDF, text, markdown, or subtitle files into high-quality audio with matching subtitles
- Uses Kokoro-82M for TTS synthesis
- Generates approximately 1 minute of audio in 5 seconds with synced subtitles
- Processes approximately 3,000 characters in 11 seconds producing 3:28 of audio on RTX 2060 Mobile
- Licensed under MIT; Kokoro licensed under Apache-2.0

### Supported Input Formats
- ePub files
- PDF files
- Plain text (.TXT) files
- Markdown (.MD) files
- Subtitle files: .SRT, .ASS, .VTT

### Supported Output Formats
- Audio: WAV, FLAC, MP3, OPUS ("best compression"), M4B ("with chapters")
- Subtitle: SRT ("standard"), ASS (wide), ASS (narrow), ASS (centered wide), ASS (centered narrow)

### User-Facing Interfaces
- `abogen` command: launches PyQt6 Desktop GUI with "stable core features"
- `abogen-web` command: launches Flask Web UI with "Core features + Supertonic TTS, LLM Normalization, Audiobookshelf Integration and more"
- Web UI is described as "under active development" and "provides the most feature-rich experience"
- Web UI runs at http://localhost:8808
- Jobs run in background worker; browser updates automatically
- Multiple jobs can run sequentially; worker processes them in order

### Desktop GUI Features
- Drag and drop file input or built-in text editor
- Speech speed adjustment from 0.1x to 2.0x
- Voice selection by language code prefix and gender suffix
- Voice mixer for creating custom voices by mixing voice models with profile system
- Voice preview before processing
- Subtitle generation modes: Disabled, Line, Sentence, Sentence + Comma, Sentence + Highlighting, 1-N words
- Chapter control for ePUBs (specific chapters), markdown, PDFs (chapters + pages)
- Save each chapter separately option
- Create merged version option
- Save in project folder with metadata
- Queue mode: add multiple files, individual settings per file, batch processing
- Theme options: System, Light, Dark
- Configurable max words per subtitle
- Configurable silence between chapters (seconds)
- Configurable max lines in log window
- Separate chapters audio format selection (wav, flac, mp3, opus)
- Desktop shortcut creation
- Config directory access
- Cache directory access
- Clear cache files
- Silent gaps between subtitles option
- Subtitle speed adjustment methods: TTS Regeneration (better quality) or FFmpeg Time-stretch (better speed)
- spaCy sentence segmentation option for improved subtitle timing
- Pre-download models/voices for offline use
- Disable Kokoro internet access option
- Check for updates at startup option
- Reset to default settings
- Replace single newlines with spaces option

### Web UI Features
- Upload via drag-and-drop or upload button
- Choose voice, language, speed, subtitle style, output format
- Create job; job immediately appears in queue
- Live progress and log updates
- Download audio/subtitle assets when complete
- Cancel or delete jobs any time
- Download logs for troubleshooting

### Container/Docker Support
- Docker build from repository root
- Uploaded files stored in /data/uploads; outputs in /data/outputs
- Docker Compose with GPU support by default (NVIDIA Container Toolkit)
- CPU-only deployment by commenting out GPU resource reservations

### Container Environment Variables
- ABOGEN_HOST: default 0.0.0.0, bind address
- ABOGEN_PORT: default 8808, HTTP port
- ABOGEN_DEBUG: default false, Flask debug mode
- ABOGEN_UPLOAD_ROOT: default /data/uploads
- ABOGEN_OUTPUT_ROOT: default /data/outputs (legacy alias of ABOGEN_OUTPUT_DIR)
- ABOGEN_OUTPUT_DIR: default /data/outputs
- ABOGEN_SETTINGS_DIR: default /config
- ABOGEN_TEMP_DIR: default /data/cache (Docker) or platform cache dir
- ABOGEN_UID: default 1000
- ABOGEN_GID: default 1000
- ABOGEN_LLM_BASE_URL: OpenAI-compatible endpoint
- ABOGEN_LLM_API_KEY: API key for LLM endpoint
- ABOGEN_LLM_MODEL: default model for LLM
- ABOGEN_LLM_TIMEOUT: default 30 seconds
- ABOGEN_LLM_CONTEXT_MODE: default "sentence", options: sentence, paragraph, document
- ABOGEN_LLM_PROMPT: custom normalization prompt template
- TORCH_VERSION, TORCH_INDEX_URL, ABOGEN_DATA: Docker Compose build/runtime knobs

### LLM-Assisted Text Normalization
- Handles "tricky apostrophes and contractions" via OpenAI-compatible LLM
- Configured from Settings -> LLM
- Accepts base URL for endpoint (Ollama, OpenAI proxy); appends /v1/... automatically
- Also accepts inputs already ending in /v1
- Refresh models to load catalog, pick default model, adjust timeout or prompt template
- Preview box to test prompt; Normalization panel can synthesize short audio preview
- Docker/CI seeding via ABOGEN_LLM_* variables

### Audiobookshelf Integration
- Push finished audiobooks directly into Audiobookshelf
- Configured under Settings -> Integrations -> Audiobookshelf
- Requires: Base URL, Library ID, Folder (name or ID), API token
- Folder name resolved to correct ID automatically
- Browse folders feature to fetch available folders
- Enable automatic uploads for future jobs or trigger individual uploads from queue

### JSON API Endpoints (Web UI)
- GET /api/jobs/<id>: returns job metadata, progress, log lines in JSON
- GET /partials/jobs: renders live job list as HTML (htmx polling)
- GET /partials/jobs/<id>/logs: renders log window

### Core Features (Both Interfaces)
- Chapter markers: <<CHAPTER_MARKER:Chapter Title>> tags in text files
- Auto-added for ePUB/PDF/markdown; manual addition supported
- Metadata tags for M4B: TITLE, ARTIST, ALBUM, YEAR, ALBUM_ARTIST, COMPOSER, GENRE, COVER_PATH
- Auto-extracted from EPUB/PDF; manual addition supported
- Timestamp-based text files: HH:MM:SS, HH:MM:SS,ms, HH:MM:SS.ms format detection
- Text before first timestamp auto-starts at 00:00:00
- Timestamp mode ignores subtitle generation mode setting

### Supported Languages
- American English (a), British English (b), Spanish (e), French (f), Hindi (h), Italian (i), Japanese (j), Brazilian Portuguese (p), Mandarin Chinese (z)
- Japanese requires additional pip install misaki[ja]
- Mandarin Chinese requires misaki[zh] (included by default since v1.1.6)
- Word-level subtitle modes only available for English; non-English uses duration-based fallback supporting Line, Sentence, Sentence + Comma

### Platform Support
- Windows, Linux, macOS
- Python 3.10 to 3.12 required
- espeak-ng required as system dependency on all platforms
- NVIDIA GPU support via CUDA (Windows/Linux)
- AMD GPU support via ROCm (Linux only)
- Apple Silicon (M1, M2) MPS GPU acceleration
- CPU fallback when no compatible GPU available

### Error Conditions / Troubleshooting
- "CUDA GPU is not available. Using CPU": PyTorch cannot use GPU, falls back to CPU
- PATH warning on Linux when script installed to non-PATH directory
- "No matching distribution found": Python version incompatible (need 3.10-3.12)
- "[WinError 1114] DLL initialization routine failed": resolved by reinstalling torch with CUDA
- Japanese audio not working: related to additional Kokoro dependencies
- abogen-cli command provides detailed error messages for troubleshooting

---

## Source: CHANGELOG.md

### Version 1.3.0 Features (Latest)
- Web UI added (abogen-web) for Docker and headless server deployments
- EPUB 3 packaging pipeline building media-overlay EPUBs from generated audio and chunk metadata
- Persisted chunk timing metadata in job artifacts
- Reorganized codebase to support both PyQt6 desktop GUI and Web UI from shared core
- Supertonic TTS engine support with GPU acceleration
- Entity analysis and pronunciation override system for proper nouns
- Speaker/role assignment for multi-voice "theatrical" audiobooks
- Calibre OPDS and Audiobookshelf integration

### Version 1.2.5
- "Override item settings with current selection" in queue manager
- Fixed xcb platform plugin error on Linux
- Fixed "No module named pip" error for uv installs

### Version 1.2.4
- Subtitle generation for all languages (duration-based timing)
- spaCy sentence segmentation option
- Pre-download models/voices for offline use
- Millisecond timestamp support (HH:MM:SS.ms)

### Version 1.2.2
- Subtitle file voicing: .srt, .ass, .vtt to timed audio
- Timestamp-based text files with HH:MM:SS or HH:MM:SS,ms format
- Silent gaps between subtitles option
- Subtitle speed adjustment methods: TTS Regeneration vs FFmpeg Time-stretch
- M4B cover image embedding from EPUB/PDF or manual METADATA_COVER_PATH tag

### Version 1.2.1
- Upgraded from PyQt5 to PyQt6

### Version 1.2.0
- "Line" subtitle generation mode

### Version 1.1.8
- Markdown (.MD) file support
- "Configure silence between chapters" option

### Version 1.1.7
- MPS GPU acceleration for Silicon Mac
- Word-by-word karaoke highlighting

### Version 1.1.6
- Mandarin Chinese (misaki[zh]) included by default

### Version 1.1.0
- Queue system for multiple items
- Dark theme support

### Version 1.0.9
- Chunking/segmenting system fixing memory outage on large files
- Subtitle format options (srt, ass variants)

### Version 1.0.8
- AMD GPU support on Linux
- Voice preview caching
- M4B metadata support

### Version 1.0.7
- .opus output format added
- "Save in project folder with metadata" option

### Version 1.0.5
- M4B output format with chapter metadata

### Version 1.0.3
- Voice mixing (multiple voices combined)
- Profile system for voice mixer

---

## Source: docs/entities_step_overhaul_plan.md

### Entity Analysis System
- POS tagging integration using spaCy for proper noun detection
- Step 3 of wizard renamed from "Speakers" to "Entities"
- Three sub-tabs: People, Entities, Manual Overrides
- People tab: characters with dialogue/speech evidence
- Entities tab: non-person proper nouns (organizations, places, artefacts)
- Manual Overrides: user-added entries with search, pronunciation editing, voice assignment
- Voice selection dropdowns for People and Manual Override rows
- Proper noun filtering: discard stopwords, remove titles ("Mr.", "Dr."), strip possessives ("Bob's" -> "Bob")
- Pronunciation overrides persist across projects in shared store
- Overrides applied to every preview request and final conversion
- POS tagging is English-only for initial release
- spaCy processes manuscript once and caches results
- Uses spaCy NER entity type labels: PERSON, ORG, GPE, etc.

---

## Source: docs/epub3_upgrade_plan.md

### EPUB 3 Output with Narration
- Generate EPUB 3 output preserving source metadata with embedded audio narration via media overlays
- Configurable chunking granularity: paragraph vs. sentence
- Speaker assignments per chunk, defaulting to single narrator
- EPUB3PackageBuilder: build XHTML spine with IDs, generate overlay SMIL files, write OPF manifest/spine, assemble zip
- Audio embedded inside EPUB (not linked externally)

### Configurable Chunking
- Users select chunking level (paragraph or sentence) before audio generation
- Stable unique IDs per chunk (format: chap{index}_para{idx}_sent{idx})
- Chunker uses spaCy for sentence segmentation; regex fallback when model unavailable

### Speaker Assignment Foundations
- Every chunk carries speaker_id (default "narrator")
- Multi-Speaker marked as "Coming Soon" in UI
- PendingJob and Job store speakers metadata
- JobResult optionally includes chunk_speakers.json artifact

---

## Source: .env.example

### Environment Configuration
- ABOGEN_SETTINGS_DIR: host directory for JSON settings; mounted to /config in Docker
- ABOGEN_OUTPUT_DIR: host directory for rendered audio/subtitle files; mounted to /data/outputs
- ABOGEN_TEMP_DIR: temporary working directory for audio conversion scratch; mounted to /data/cache
- Falls back to OS cache directory for non-Docker usage
- Only audio conversion scratch files staged in TEMP_DIR; other library caches remain in container volume
- ABOGEN_UID/ABOGEN_GID: default 1000:1000 matching most Linux hosts
- ABOGEN_NETWORK_MODE: "bridge" (default, isolated) or "host" (required for LAN resources like Calibre OPDS)
- ABOGEN_LLM_BASE_URL: example http://localhost:11434 (Ollama); /v1 added automatically
- ABOGEN_LLM_API_KEY: example "ollama"
- ABOGEN_LLM_MODEL: example "llama3.1:8b"
- ABOGEN_LLM_TIMEOUT: example 45 seconds
- ABOGEN_LLM_CONTEXT_MODE: example "sentence"
- ABOGEN_LLM_PROMPT: optional custom prompt, keep on single line or escape newlines

---

## Source: pyproject.toml

### Entry Points
- `abogen` (gui-scripts): abogen.pyqt.main:main
- `abogen-cli`: abogen.webui.app:main
- `abogen-web`: abogen.webui.app:main
- `abogen-pyqt`: abogen.pyqt.main:main
- NOTE: abogen-cli and abogen-web both point to same webui.app:main entry point

### Core Dependencies
- kokoro>=0.9.4 (TTS engine)
- misaki[zh]>=0.9.4 (Chinese language support)
- supertonic>=0.1.0 (Supertonic TTS engine)
- ebooklib>=0.19 (EPUB reading)
- beautifulsoup4>=4.13.4 (HTML parsing)
- spacy>=3.8.7,<4.0 (NLP/sentence segmentation)
- PyMuPDF>=1.25.5 (PDF reading)
- platformdirs>=4.3.7 (OS-specific directories)
- soundfile>=0.13.1 (audio file I/O)
- mutagen>=1.47.0 (audio metadata)
- pygame>=2.6.1 (audio playback for previews)
- charset_normalizer>=3.4.1, chardet>=5.2.0 (encoding detection)
- python-dotenv>=1.0.1 (env file loading)
- static_ffmpeg>=2.13 (bundled FFmpeg)
- Markdown>=3.9 (markdown processing)
- Flask>=3.0.3 (web framework)
- numpy>=1.24.0 (numerical computing)
- gpustat>=1.1.1 (GPU monitoring)
- num2words>=0.5.13 (number-to-word conversion)
- httpx>=0.27.0 (HTTP client)
- PyQt6>=6.5.0 (desktop GUI framework)

---

## Integration Points Summary

### Audiobookshelf
- Push completed audiobooks directly
- Base URL, Library ID, Folder, API token configuration
- Automatic uploads or manual per-job uploads
- Folder name auto-resolved to ID

### Calibre OPDS
- Browse/download books from Calibre content servers
- Requires host network mode for Docker to access LAN resources

### LLM Providers (OpenAI-compatible)
- Ollama, OpenAI proxy, or any OpenAI-compatible endpoint
- Configured via ABOGEN_LLM_* environment variables or Settings UI
- Used for text normalization (apostrophes, contractions)

### Kokoro-82M (TTS Engine)
- Primary TTS engine
- Models/voices downloadable from HuggingFace Hub
- Offline mode available after pre-download
- Timestamp tokens only for English (word-level subtitles English-only)

### Supertonic TTS
- Additional TTS engine with GPU acceleration
- Web UI only feature

### spaCy
- Sentence segmentation for subtitle generation
- NER for entity/speaker detection
- POS tagging for proper noun filtering
- English-only for initial entity analysis release

### FFmpeg
- Bundled via static_ffmpeg package
- Used for audio format conversion, time-stretch, video creation
- M4B chapter metadata generation

### espeak-ng
- Required system dependency on all platforms
- Phoneme backend for Kokoro TTS
