# Chunk 1: core-infrastructure

## Files
- `abogen/main.py`
- `abogen/constants.py`
- `abogen/utils.py`
- `abogen/hf_tracker.py`
- `abogen/is_nvidia.py`
- `abogen/check_cuda.py`

## Description
Application entry point, constants/configuration, shared utilities (file I/O, config management, path resolution, GPU detection, sleep prevention, process creation), and HuggingFace download tracking.

## Internal Relationships
- `main.py` imports from `utils` and launches the web UI
- `constants.py` defines VOICES_INTERNAL, language mappings, format lists
- `utils.py` provides `load_config`, `clean_text`, `detect_encoding`, `get_user_cache_path`, `prevent_sleep_*`, `create_process`, `get_gpu_acceleration` used by nearly every other module
- `hf_tracker.py` wraps `hf_hub_download` with progress callbacks for the PyQt GUI
