# Behavioral Specification: WebUI Service Layer

**Module**: `abogen/webui/service.py`, `abogen/webui/conversion_runner.py`  
**Version**: STATE_VERSION = 8  
**Scope**: Job orchestration, conversion pipeline execution, state persistence

---

## 1. Module Identity

### Single-Worker Sequential Processor
**Given** ConversionService is initialized  
**When** jobs are enqueued  
**Then** exactly one daemon thread named `abogen-conversion-worker` processes jobs sequentially; only one job executes at any time  
**Source**: `service.py:955-960`

### Service Construction Order
**Given** `build_service(runner, *, output_root=None, uploads_root=None)` is called  
**When** instantiated  
**Then** performs: (1) initialize empty `_jobs` dict and `_queue` list, (2) create RLock + stop/wake Events, (3) determine state file path, (4) create output/upload/state directories, (5) bootstrap voice cache, (6) load persisted state  
**Source**: `service.py:1606, 581-604`

---

## 2. Public Interface

### JobStatus Enum
Values: `PENDING`, `RUNNING`, `PAUSED`, `COMPLETED`, `FAILED`, `CANCELLED` (string-based via `str, Enum`)  
**Source**: `service.py:73-79`

### Job Dataclass
Defaults: `status=PENDING`, `progress=0.0`, `cancel_requested=False`, `pause_requested=False`, `paused=False`, `pause_event` initially set  
**Source**: `service.py:98-160`

### Job.estimated_time_remaining
**Given** RUNNING job with `started_at` set and `progress > 0`  
**When** accessed  
**Then** returns `max(0.0, (elapsed / progress) - elapsed)`; None if not RUNNING or progress zero  
**Source**: `service.py:162-178`

### ConversionService.list_jobs
**Given** jobs exist  
**When** called  
**Then** returns all jobs sorted by `created_at` descending, under RLock  
**Source**: `service.py:607-609`

### ConversionService.get_job
**Given** job_id  
**When** called  
**Then** returns Job if found, else None; under RLock  
**Source**: `service.py:611-613`

---

## 3. Job Lifecycle

### Enqueue Creates PENDING Job
**Given** valid parameters  
**When** `enqueue(...)` called  
**Then** new Job with `id=uuid4().hex`, `status=PENDING`, `created_at=time.time()`; added to `_jobs`, appended to `_queue`, wake_event set, worker started if needed  
**Source**: `service.py:615-724`

### Worker Dequeues FIFO
**Given** jobs in queue  
**When** worker iterates  
**Then** pops first item from `_queue` (index 0), skipping terminal states  
**Source**: `service.py:962-979`

### Job Transitions to RUNNING
**Given** dequeued job without cancel_requested  
**When** `_run_job` begins  
**Then** status→RUNNING, `started_at` set, state persisted  
**Source**: `service.py:987-994`

### Successful Completion
**Given** RUNNING job  
**When** runner returns without exception and not cancelled and status not FAILED  
**Then** status→COMPLETED, `_post_completion_hooks` invoked, `finished_at` set  
**Source**: `service.py:1009-1017`

### Runner Exception
**Given** RUNNING job  
**When** runner raises exception  
**Then** `job.error` set, status→FAILED, `finished_at` set, traceback logged (up to 20 lines)  
**Source**: `service.py:997-1008`

### Mid-Run Cancellation
**Given** RUNNING job  
**When** runner returns but `cancel_requested` is True  
**Then** status→CANCELLED  
**Source**: `service.py:1010-1012`

---

## 4. Queue Management

### FIFO Ordering
Jobs processed in insertion order (first appended = first popped from index 0)  
**Source**: `service.py:719, 974`

### No Capacity Limit
`_queue` list grows unbounded  
**Source**: `service.py:615-724`

### Resumed Jobs at Front
**Given** PAUSED job paused before starting  
**When** resumed  
**Then** inserted at index 0 (front of queue)  
**Source**: `service.py:796`

---

## 5. State Persistence

### JSON Format
`{version: 8, jobs: [...], queue: [...]}`  
**Source**: `service.py:1208-1222`

### Atomic Write
Writes to `.tmp` then `os.replace()` for atomicity  
**Source**: `service.py:1216-1219`

### Log Truncation
Only last 500 log entries persisted per job  
**Source**: `service.py:1174`

### Persistence After Every Mutation
Called after enqueue, cancel, pause, resume, delete, clear_finished, status changes  
**Source**: `service.py:755, 778, 807, 894, 917, 994, 1019, 1029`

### Persistence Failures Silent
Exceptions swallowed; in-memory state remains authoritative  
**Source**: `service.py:1220-1222`

### State Path Resolution
Priority: (1) `ABOGEN_QUEUE_STATE_PATH`, (2) `ABOGEN_QUEUE_STATE_DIR`, (3) `{settings_dir}/queue/`, (4) `get_internal_cache_path("jobs")`; filename always `queue_state.json`  
**Source**: `service.py:1224-1256`

### Version Compatibility
Accepts version 8 (current) and 7 (previous); any other version ignored  
**Source**: `service.py:1349-1351`

### Restart Recovery
RUNNING/PAUSED jobs reset to PENDING on restart; progress zeroed; re-queued  
**Source**: `service.py:1364-1372`

---

## 6. Worker Thread

### Daemon Thread
Created with `daemon=True`; named `abogen-conversion-worker`  
**Source**: `service.py:955-960`

### Wake/Poll Mechanism
Waits on `_wake_event` with 0.5s timeout when idle; woken by `enqueue()`  
**Source**: `service.py:978, 721`

### Shutdown Protocol
Sets `_stop_event`, sets `_wake_event`, joins thread with 5s timeout  
**Source**: `service.py:920-925`

### Single Worker Guarantee
`_ensure_worker()` no-ops if thread alive  
**Source**: `service.py:951-953`

---

## 7. Conversion Pipeline

### Pipeline Stages (run_conversion_job)
1. Settings/normalization config loading
2. Text extraction (`extract_from_path`)
3. Pronunciation override compilation
4. Chapter selection/override
5. Chunking with normalization
6. Voice resolution with per-job cache
7. TTS synthesis (Kokoro or SuperTonic)
8. Audio encoding (WAV/FLAC direct, MP3/Opus/M4B via ffmpeg)
9. Subtitle writing (concurrent with audio)
10. M4B metadata embedding (if applicable)
11. EPUB3 generation (if enabled)
12. Metadata JSON artifact
13. Resource cleanup

**Source**: `conversion_runner.py`

### Cancellation Check Points
Called: (a) once per chapter start, (b) once per TTS segment  
**Source**: `conversion_runner.py:1932, 1872`

### Progress Reporting
Incremented by `len(graphemes)` per segment; capped at 0.999 during processing; set to 1.0 on completion  
**Source**: `conversion_runner.py:1887-1892, 2351-2352`

### M4B Forces Merge
If format is "m4b", `merge_chapters_at_end` forced True  
**Source**: `conversion_runner.py:1749-1754`

### CUDA Fallback
If KPipeline init raises RuntimeError containing "CUDA", re-init with CPU  
**Source**: `conversion_runner.py:1601-1606`

### Resource Cleanup (finally)
Always: close sinks, close subtitle writer, clear pipelines, gc.collect(), torch.cuda.empty_cache()  
**Source**: `conversion_runner.py:2408-2423`

---

## 8. Post-Completion Hooks

### Audiobookshelf Auto-Upload
**Given** job COMPLETED  
**When** `_post_completion_hooks` runs  
**Then** checks config for `audiobookshelf.enabled=True` AND `auto_send=True`; if both, uploads  
**Source**: `service.py:1037-1053`

### Existing Item Deletion
**Given** same-title item exists  
**When** pre-upload check  
**Then** deleted before uploading new version  
**Source**: `service.py:1111-1125`

### Hook Failure Isolation
**Given** hook raises exception  
**When** caught  
**Then** error logged to job; status remains COMPLETED  
**Source**: `service.py:1040-1043`

---

## 9. Pause/Resume/Cancel

### Cancel PENDING
Immediate: status→CANCELLED, removed from queue  
**Source**: `service.py:738-756`

### Cancel RUNNING
Sets `cancel_requested=True`; actual cancellation at next segment boundary via `canceller()`  
**Source**: `service.py:738-756, conversion_runner.py:2712-2717`

### Pause PENDING
Removes from queue, status→PAUSED  
**Source**: `service.py:758-778`

### Pause RUNNING (Limitation)
Sets `pause_requested=True` and logs, but runner has NO pause_event.wait() calls — pause not enforced during active synthesis  
**Source**: `service.py:768-778`

### Resume
Clears paused state; if was PENDING: inserts at front of queue; if was RUNNING: returns to RUNNING  
**Source**: `service.py:791-805`

---

## 10. Thread Safety

| Mechanism | Protects |
|-----------|----------|
| `self._lock` (RLock) | All `_jobs` and `_queue` mutations |
| `_wake_event` (Event) | Worker wake signaling |
| `_stop_event` (Event) | Worker shutdown |
| Per-job `pause_event` (Event) | Pause/resume signaling |

Worker holds lock only briefly (dequeue, status updates); NOT during long-running conversion.  
**Source**: `service.py:592, 608, 612, 717, 964`

---

## 11. Invariants

1. **Single active job**: At most one RUNNING at any time
2. **FIFO ordering**: Queue processed in insertion order
3. **Atomic state writes**: Disk file never in partial state
4. **Progress bounded [0, 1.0]**: Capped at 0.999 during processing, 1.0 only on completion
5. **Terminal jobs have finished_at**: COMPLETED/FAILED/CANCELLED always timestamped
6. **Resource cleanup guaranteed**: finally block always executes after any job outcome
7. **Job ID unique**: uuid4().hex per job
8. **Queue position accurate**: Updated after every mutation
9. **Persistence after every mutation**: On-disk state reflects in-memory (when writes succeed)
