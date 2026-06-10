# Intelligence Source Inventory -- abogen

Generated: 2026-06-09

## Summary

| # | Source Type | Availability | Location / Rationale |
|---|-------------|-------------|----------------------|
| 1 | Source code (Python) | available | 66 files, ~36,293 LOC in `abogen/` package directory |
| 2 | Documentation | available | README.md (730 lines), CHANGELOG.md, docs/ (2 planning docs), .env.example |
| 3 | SDK / package metadata | available | pyproject.toml with full dependency list, PyPI-published (`abogen` v1.3.1) |
| 4 | Community content | available | GitHub repo (denizsafak/abogen), 632 commits, 10+ contributors, active issues tracker |
| 5 | Runtime executability (CLI) | available | Entry points: `abogen-cli`, `abogen-web`, `abogen-pyqt` defined in pyproject.toml |
| 6 | Runtime executability (GUI) | available | PyQt6 desktop GUI (`abogen` command), Flask Web UI on port 8808 |
| 7 | Binary artifacts | unavailable | No compiled .so/.dylib/.pyd/.exe/.pyc files found in the repo |
| 8 | Git history | available | 632 commits, 2025-04-24 to 2026-04-30, 10 contributors, active development |
| 9 | Test suites | available | 39 test files, ~5,016 LOC in `tests/`, pytest with conftest.py fixtures |
| 10 | Visual UI (screenshots/templates) | available | demo/ has 6 screenshots + 1 GIF; webui/templates has 26 HTML files, 12 JS files, 1 CSS |
| 11 | Machine-readable contracts | unavailable | No OpenAPI specs, JSON schemas, or config schemas found |
| 12 | Docker / containerization | available | Dockerfile, docker-compose.yaml, docker-compose.webui.yml, GHCR image workflow |
| 13 | CI/CD pipelines | available | 2 GitHub Actions workflows (pip install matrix, Docker multi-arch build) |

## Detailed Notes

### 1. Source Code

- Package: `abogen/` with 66 Python files across subpackages:
  - `abogen/pyqt/` -- PyQt6 desktop GUI (7 files)
  - `abogen/webui/` -- Flask web application (15 files including routes/)
  - `abogen/integrations/` -- Audiobookshelf + Calibre OPDS (3 files)
  - `abogen/epub3/` -- EPUB 3 media-overlay exporter (2 files)
  - Core modules: book_parser, chunking, text_extractor, tts_supertonic, utils, llm_client, voice_formulas, subtitle_utils, etc.
- Total: ~36,293 lines of Python source
- Build system: Hatchling (hatch)
- Version: 1.3.1 (from `abogen/VERSION`)

### 2. Documentation

- `README.md` -- Comprehensive (730 lines): installation for Win/Mac/Linux, usage guides, troubleshooting, configuration tables, Docker usage, LLM normalization, Audiobookshelf integration
- `CHANGELOG.md` -- Detailed release notes back to v1.2.1+
- `docs/entities_step_overhaul_plan.md` -- Planning document
- `docs/epub3_upgrade_plan.md` -- Planning document
- `.env.example` -- Documented environment variable reference
- `demo/README.md` -- Demo guide

### 3. SDK / Package Metadata

- Published on PyPI as `abogen`
- Python 3.10-3.12 required
- 23 direct dependencies including: kokoro, Flask, PyQt6, spacy, PyMuPDF, ebooklib, beautifulsoup4, soundfile, mutagen, httpx, numpy
- Optional dependency groups: cuda126, cuda, cuda130, rocm, dev
- Custom UV index configuration for torch/ROCm

### 4. Community Content

- GitHub: github.com/denizsafak/abogen
- TrendShift badge present (trending project)
- 10+ contributors (Deniz Safak: 331 commits, JB: 249, Juraj Borza: 41, plus community)
- Active issues page referenced throughout README
- Multiple PRs cited (120, 75, 10, 94, 65, 35, 5, etc.)

### 5-6. Runtime Executability

- CLI entry points: `abogen-cli` and `abogen-web` -> `abogen.webui.app:main`
- GUI entry points: `abogen` and `abogen-pyqt` -> `abogen.pyqt.main:main`
- Web UI: Flask app serving on port 8808 with htmx-driven live updates
- Docker: single-command deployment with GPU support

### 7. Binary Artifacts

- None found. Pure Python project.
- Dependencies like PyTorch bring compiled binaries at install time but nothing committed to the repo.

### 8. Git History

- 632 total commits
- First commit: 2025-04-24
- Latest commit: 2026-04-30
- Roughly 1 year of development
- Active merge of community PRs

### 9. Test Suites

- 39 test files in `tests/`
- ~5,016 LOC of tests
- Uses pytest with session-scoped fixtures (conftest.py isolates settings dir)
- Covers: book parsing, EPUB handling (NCX, nav, heuristic), text normalization, voice formulas, conversion pipeline, integrations (Audiobookshelf, Calibre OPDS), speaker analysis, service layer
- CI runs tests on ubuntu/macos/windows matrix

### 10. Visual UI

- Screenshots in `demo/`: abogen.png, abogen2.png, abogen-webui.png, voice_mixer.png, queue.png, chapter_marker.png, abogen.gif
- Assets in `abogen/assets/`: icon.ico, icon.png, flags/, loading.gif, settings.svg
- Web UI templates: 26 HTML files (Jinja2), 12 JavaScript files, 1 CSS file
- HTMX-powered partial templates for live job status updates

### 11. Machine-Readable Contracts

- No OpenAPI/Swagger specification found despite the Flask API having JSON endpoints (`/api/jobs/<id>`, `/partials/jobs`, etc.)
- No JSON Schema files
- No formal config schema (configuration is documented via README tables and .env.example)

### 12-13. Docker and CI/CD

- `abogen/webui/Dockerfile` -- Main container image
- `docker-compose.yaml` -- GPU-enabled deployment (NVIDIA Container Toolkit)
- `docker-compose.webui.yml` -- Alternative compose file
- GHCR publishing workflow (manual trigger, linux/amd64)
- `test_pip.yml` -- Cross-platform install verification (ubuntu, macos, windows x Python 3.12)
