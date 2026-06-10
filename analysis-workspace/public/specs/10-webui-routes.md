# Behavioral Specification: WebUI Routes

## Module Identity

**Paths**: `abogen/webui/routes/main.py`, `books.py`, `jobs.py`, `voices.py`, `settings.py`, `entities.py`, `api.py`, `abogen/webui/app.py`  
**Role**: Flask HTTP endpoints providing the Web UI interface via htmx-driven HTML responses and JSON API  
**Boundaries**: Route layer only. Delegates to `ConversionService`, voice profiles, pronunciation store, integrations. Does NOT perform TTS or text processing directly.  
**Framework**: Flask blueprints + htmx (partial HTML responses) + Jinja2 templates

---

## Blueprint Organization

| Blueprint | Prefix | Responsibility |
|-----------|--------|---------------|
| `main_bp` | `/` | Index, wizard flow, templates |
| `books_bp` | `/books` | Calibre OPDS browsing |
| `jobs_bp` | `/jobs` | Job lifecycle management |
| `voices_bp` | `/voices` | Voice profiles + speaker presets |
| `settings_bp` | `/settings` | Configuration management |
| `entities_bp` | `/entities` | NER analysis + pronunciation overrides |
| `api_bp` | `/api` | JSON API for AJAX operations |

---

## Wizard Flow (main_bp)

### GET /
**Given** user visits root  
**When** rendered  
**Then** shows index page with job queue status  
**Source**: `main.py:53-54`

### GET /wizard
**Given** user starts new conversion  
**When** visited  
**Then** renders wizard start page (file selection)  
**Source**: `main.py:83-84`

### GET /wizard/<step>
**Given** wizard step name  
**When** navigated to  
**Then** renders appropriate step template; validates step exists  
**Source**: `main.py:91-92`

### POST /wizard/upload
**Given** file upload form submission  
**When** file received  
**Then** saves to uploads dir; parses book (EPUB/PDF/MD/TXT); extracts chapters + metadata; stores in session; returns chapter selection step  
**Source**: `main.py:113-114`

### POST /wizard/text
**Given** direct text input  
**When** submitted  
**Then** saves text as temp file; creates single-chapter structure; stores in session  
**Source**: `main.py:204-205`

### POST /wizard/update
**Given** wizard parameter changes (chapters, voice, format, etc.)  
**When** submitted  
**Then** updates session state; returns partial HTML for updated wizard section  
**Source**: `main.py:265-266`

### POST /wizard/finish
**Given** wizard completion  
**When** submitted  
**Then** enqueues conversion job via ConversionService; redirects to job detail page  
**Source**: `main.py:360-361`

### POST /wizard/cancel
**Given** wizard cancellation  
**When** submitted  
**Then** cleans up session state + temp files; redirects to index  
**Source**: `main.py:349-350`

---

## Job Management (jobs_bp)

### GET /jobs/<job_id>
**Given** job ID  
**When** requested  
**Then** returns job detail page with status, progress, logs, download links  
**Source**: `jobs.py:34-35`

### POST /jobs/<job_id>/pause
**Given** RUNNING or PENDING job  
**When** pause requested  
**Then** calls `service.pause_job()`; returns updated job status partial  
**Source**: `jobs.py:48-49`

### POST /jobs/<job_id>/resume
**Given** PAUSED job  
**When** resume requested  
**Then** calls `service.resume_job()`; returns updated status  
**Source**: `jobs.py:55-56`

### POST /jobs/<job_id>/cancel
**Given** RUNNING or PENDING job  
**When** cancel requested  
**Then** calls `service.cancel_job()`; returns updated status  
**Source**: `jobs.py:62-63`

### POST /jobs/<job_id>/delete
**Given** any job  
**When** delete requested  
**Then** calls `service.delete_job()`; redirects to index  
**Source**: `jobs.py:69-70`

### POST /jobs/<job_id>/retry
**Given** COMPLETED, FAILED, or CANCELLED job  
**When** retry requested  
**Then** re-enqueues with same parameters; redirects to new job if created, otherwise redirects back to original job  
**Source**: `jobs.py:76-82`

### POST /jobs/<job_id>/audiobookshelf
**Given** COMPLETED job  
**When** send to Audiobookshelf requested  
**Then** uploads output files to configured Audiobookshelf server  
**Source**: `jobs.py:85-86`

### GET /jobs/<job_id>/epub
**Given** COMPLETED job with EPUB3 output  
**When** requested  
**Then** serves EPUB3 file for download  
**Source**: `jobs.py:214-215`

### GET /jobs/<job_id>/download/<file_type>
**Given** COMPLETED job and file type (audio/subtitles/metadata)  
**When** requested  
**Then** serves appropriate output file  
**Source**: `jobs.py:229-230`

### POST /jobs/clear-finished
**Given** terminal jobs exist  
**When** requested  
**Then** removes all COMPLETED/FAILED/CANCELLED jobs from list  
**Source**: `jobs.py:207-208`

---

## Voice Management (voices_bp)

### GET /voices/
**Given** user visits voice profiles  
**When** rendered  
**Then** shows all voice profiles with formula display  
**Source**: `voices.py:24-25`

### POST /voices/test
**Given** voice formula and sample text  
**When** test requested  
**Then** synthesizes short audio clip; returns audio player partial  
**Source**: `voices.py:28-29`

### GET /voices/configs
**Given** speaker configurations exist  
**When** requested  
**Then** returns JSON list of speaker configs  
**Source**: `voices.py:49-50`

### POST /voices/configs/save
**Given** speaker config data  
**When** submitted  
**Then** saves/updates named speaker configuration  
**Source**: `voices.py:53-54`

### POST /voices/configs/delete
**Given** speaker config name  
**When** submitted  
**Then** deletes named configuration  
**Source**: `voices.py:69-70`

### GET /voices/presets + POST /voices/presets
**Given** preset management  
**When** accessed  
**Then** GET: renders preset page; POST: saves preset with voice formula + parameters  
**Source**: `voices.py:80-81`

---

## Settings (settings_bp)

### GET /settings/
**Given** user visits settings  
**When** rendered  
**Then** shows configuration form with current values from config.json  
**Source**: `settings.py:189-190`

### POST /settings/update
**Given** settings form submission  
**When** submitted  
**Then** validates + saves updated config values; applies normalization settings to ApostropheConfig; returns confirmation partial  
**Source**: `settings.py:37-38`

### POST /settings/debug/run
**Given** debug TTS request  
**When** submitted  
**Then** generates debug WAV files for normalization comparison; returns run ID  
**Source**: `settings.py:224-225`

### GET /settings/debug/<run_id>
**Given** debug run completed  
**When** visited  
**Then** shows comparison page with original vs normalized audio  
**Source**: `settings.py:238-239`

---

## Entity Analysis (entities_bp)

### POST /entities/analyze
**Given** book text and language  
**When** analysis requested  
**Then** spawns entity extraction (NER); returns pending ID for polling  
**Source**: `entities.py:24-25`

### GET /entities/pending/<pending_id>
**Given** pending analysis  
**When** polled  
**Then** returns analysis results if complete, or 202 (still processing)  
**Source**: `entities.py:38-39`

### POST /entities/pending/<pending_id>/refresh
**Given** completed analysis  
**When** refresh requested  
**Then** re-runs entity extraction  
**Source**: `entities.py:54-55`

### GET /entities/pending/<pending_id>/overrides
**Given** pending analysis context  
**When** requested  
**Then** returns list of pronunciation overrides for detected entities  
**Source**: `entities.py:61-62`

### POST /entities/pending/<pending_id>/overrides
**Given** override data (token, pronunciation, voice)  
**When** submitted  
**Then** creates/updates pronunciation override in store  
**Source**: `entities.py:71-72`

### DELETE /entities/pending/<pending_id>/overrides/<override_id>
**Given** existing override  
**When** deleted  
**Then** removes from pronunciation store  
**Source**: `entities.py:86-87`

### GET /entities/pending/<pending_id>/overrides/search
**Given** search query  
**When** submitted  
**Then** searches pronunciation store by partial match  
**Source**: `entities.py:96-97`

---

## JSON API (api_bp)

### Voice Profile CRUD

| Method | Path | Behavior |
|--------|------|----------|
| GET | `/api/voice-profiles` | List all profiles |
| POST | `/api/voice-profiles` | Create/update profile |
| DELETE | `/api/voice-profiles/<name>` | Delete by name |
| POST | `/api/voice-profiles/<name>/duplicate` | Copy profile |
| POST | `/api/voice-profiles/import` | Import from JSON |
| GET | `/api/voice-profiles/export` | Export all as JSON |
| POST | `/api/voice-profiles/preview` | Preview voice (synthesize sample) |

**Source**: `api.py:51-217`

### Speaker Preview
**Given** voice formula + text  
**When** POST `/api/speaker-preview`  
**Then** synthesizes audio clip; returns base64-encoded audio  
**Source**: `api.py:217-218`

### Calibre OPDS Feed
**Given** OPDS URL configured  
**When** GET `/api/integrations/calibre-opds/feed`  
**Then** fetches and returns OPDS feed entries as JSON  
**Source**: `api.py:361-362`

### Audiobookshelf Folders
**Given** ABS configured  
**When** POST `/api/integrations/audiobookshelf/folders`  
**Then** fetches available library folders from Audiobookshelf  
**Source**: `api.py:408`

---

## Book Browsing (books_bp)

### GET /books/
**Given** Calibre OPDS integration enabled  
**When** visited  
**Then** renders book search/browse page  
**Source**: `books.py:18-19`

### GET /books/search
**Given** search query  
**When** submitted  
**Then** searches Calibre OPDS catalog; returns results partial  
**Source**: `books.py:30-31`

### Integration Gate
**Given** integration settings  
**When** checking enabled  
**Then** requires `calibre_opds.enabled=True` in config  
**Source**: `books.py:14-15`

---

## Template Filters

### datetimeformat
**Given** Unix timestamp float  
**When** rendered in template  
**Then** formats as `%Y-%m-%d %H:%M:%S`  
**Source**: `main.py:31-32`

### durationformat
**Given** seconds as float  
**When** rendered  
**Then** formats as `Xh Ym Zs` or `Ym Zs` or `Zs`  
**Source**: `main.py:38-39`

---

## App Configuration (app.py)

### Flask App Factory
**Given** app initialization  
**When** `create_app()` or app module loaded  
**Then** creates Flask app; registers all blueprints; configures: session secret (from file or generated), static folder, template folder, upload folder  
**Source**: `app.py`

### Session Management
**Given** Flask sessions  
**When** wizard state stored  
**Then** uses server-side session with secret_key from `{settings_dir}/secret_key` file  
**Source**: `app.py`

---

## Error Handling

### Job Not Found
**Given** invalid job_id  
**When** any job route accessed  
**Then** returns 200 with friendly `job_not_found.html` page (not 404 — avoids confusion from stale browser tabs)  
**Source**: `jobs.py:37-38`

### File Upload Validation
**Given** uploaded file  
**When** checking  
**Then** validates extension against allowed types (epub, pdf, md, txt, markdown); rejects with 400 on invalid  
**Source**: `main.py:113+`

### Integration Disabled
**Given** integration not configured  
**When** integration route accessed  
**Then** returns appropriate error (400/503) with message  
**Source**: `books.py:14-15`, `jobs.py:85+`

---

## htmx Patterns

### Partial Responses
Routes return full page on direct GET, partial HTML on htmx requests (detected via `HX-Request` header or `request.headers.get("HX-Request")`)

### Polling for Progress
Job detail page polls `/jobs/<id>` with htmx trigger for progress updates until terminal state

### OOB Swaps
Some responses include out-of-band swap elements to update multiple page regions simultaneously

---

## Invariants

1. **Blueprint isolation**: Each blueprint handles its own URL prefix; no cross-blueprint route conflicts
2. **Service singleton**: All routes share single `ConversionService` instance
3. **Config read-through**: Settings routes always read current config.json (not cached stale values)
4. **Job operations idempotent**: Pause/resume/cancel on already-terminal jobs are no-ops
5. **File serving safe**: Download routes validate job ownership and file existence before serving
6. **Session state ephemeral**: Wizard state cleared on cancel/finish; not persisted across restarts
7. **JSON API returns JSON**: api_bp always returns `application/json`; other blueprints return HTML
8. **Upload cleanup**: Temp files removed on wizard cancel or after conversion completes
