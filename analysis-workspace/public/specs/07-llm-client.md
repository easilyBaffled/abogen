# Behavioral Specification: LLM Client

## Module Identity

**Path**: `abogen/llm_client.py`  
**Role**: Stdlib-only OpenAI-compatible HTTP client for LLM API communication  
**Boundaries**: HTTP request/response handling only. No caching, no retry logic, no streaming, no model loading. Zero external dependencies beyond Python stdlib (`urllib`, `json`, `dataclasses`).  
**Does NOT**: Load models, manage conversation history, handle streaming responses, implement retry/backoff, validate prompt content.

---

## Public Interface

### LLMConfiguration (frozen dataclass)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `base_url` | str | required | API base URL |
| `api_key` | str | required | API key; "ollama" = skip auth header |
| `model` | str | required | Model identifier |
| `timeout` | float | 30.0 | Request timeout in seconds |

#### is_configured() -> bool
**Given** an LLMConfiguration instance  
**When** `is_configured()` is called  
**Then** returns True only if both `base_url.strip()` and `model.strip()` are non-empty  
**Source**: `llm_client.py:21`

### LLMToolCall (frozen dataclass)

| Field | Type | Description |
|-------|------|-------------|
| `name` | str | Tool function name |
| `arguments` | str | JSON-encoded arguments string |

### LLMCompletion (frozen dataclass)

| Field | Type | Description |
|-------|------|-------------|
| `content` | Optional[str] | Text response (stripped, None if empty) |
| `tool_calls` | Tuple[LLMToolCall, ...] | Extracted tool calls |

### LLMClientError (RuntimeError)

Single exception type for all failures.

---

## URL Construction

### Base URL Normalization
**Given** a base_url string  
**When** `_normalized_base_url(base_url)` is called  
**Then** strips whitespace, appends trailing "/" if missing  
**Source**: `llm_client.py:43-48`

### Base URL Empty
**Given** a base_url that is empty or whitespace-only  
**When** `_normalized_base_url(base_url)` is called  
**Then** raises `LLMClientError("LLM base URL is required")`  
**Source**: `llm_client.py:45-46`

### Duplicate v1 Path Prevention
**Given** a base_url ending in `/v1` or `/v1/`  
**When** building a URL with path starting with `v1/`  
**Then** the `v1/` prefix is stripped from path to avoid `/v1/v1/` doubling  
**Source**: `llm_client.py:55-58`

### URL Join
**Given** a normalized base_url and trimmed path  
**When** `_build_url()` is called  
**Then** uses `urllib.parse.urljoin()` for proper URL construction  
**Source**: `llm_client.py:59`

---

## Authentication

### Standard API Key
**Given** an api_key that is non-empty and not "ollama" (case-insensitive)  
**When** headers are built  
**Then** includes `Authorization: Bearer {api_key}`  
**Source**: `llm_client.py:64-66`

### Ollama Special Case
**Given** an api_key equal to "ollama" (case-insensitive after strip)  
**When** headers are built  
**Then** NO Authorization header is included  
**Source**: `llm_client.py:65`

### Empty API Key
**Given** an api_key that is empty or whitespace-only  
**When** headers are built  
**Then** NO Authorization header is included  
**Source**: `llm_client.py:65`

### Default Headers
**Given** any request  
**When** headers are built  
**Then** always includes `Content-Type: application/json` and `Accept: application/json`  
**Source**: `llm_client.py:36-39`

---

## HTTP Communication

### Request Construction
**Given** method, url, optional payload  
**When** `_perform_request()` is called  
**Then** creates `urllib.request.Request` with method uppercased, payload JSON-encoded as UTF-8 bytes  
**Source**: `llm_client.py:79-84`

### Timeout
**Given** a timeout value  
**When** `urlopen()` is called  
**Then** passes timeout directly to `urlopen(req, timeout=timeout)`  
**Source**: `llm_client.py:86`

### Empty Response Body
**Given** a successful response with empty body  
**When** response is processed  
**Then** returns None  
**Source**: `llm_client.py:96-97`

### JSON Response Parsing
**Given** a successful response with body  
**When** response is processed  
**Then** decodes as UTF-8 and parses as JSON  
**Source**: `llm_client.py:99`

---

## Error Behaviors

### HTTP Error (4xx/5xx)
**Given** server returns an HTTP error status  
**When** `_perform_request()` catches `HTTPError`  
**Then** raises `LLMClientError(f"LLM request failed ({exc.code}): {message}")` where message is response body decoded as UTF-8 or `exc.reason`  
**Source**: `llm_client.py:88-90`

### URL/Connection Error
**Given** a network connection failure  
**When** `_perform_request()` catches `URLError`  
**Then** raises `LLMClientError(f"LLM request failed: {exc.reason}")`  
**Source**: `llm_client.py:91-92`

### Generic Exception
**Given** any other exception during request  
**When** `_perform_request()` catches `Exception`  
**Then** raises `LLMClientError("LLM request failed")`  
**Source**: `llm_client.py:93-94`

### Invalid JSON Response
**Given** a response body that is not valid JSON  
**When** `json.loads()` raises `JSONDecodeError`  
**Then** raises `LLMClientError("LLM response was not valid JSON")`  
**Source**: `llm_client.py:100-101`

---

## Model Listing

### list_models(configuration) -> List[Dict[str, str]]

### Incomplete Configuration
**Given** configuration where `is_configured()` is False AND `base_url.strip()` is empty  
**When** `list_models()` is called  
**Then** raises `LLMClientError("LLM configuration is incomplete")`  
**Source**: `llm_client.py:105-106`

### Successful Listing
**Given** a valid configuration and server returns `{"data": [...]}`  
**When** `list_models()` is called  
**Then** sends GET to `{base_url}/v1/models`, extracts `data` array, returns list of `{"id": ..., "label": ...}` dicts  
**Source**: `llm_client.py:107-126`

### Model Entry Parsing
**Given** a model entry in the response data array  
**When** processing entries  
**Then** extracts `id` field (required, skips if empty); uses `name` or `description` or `id` for label  
**Source**: `llm_client.py:119-125`

### Non-Mapping Response
**Given** server returns a non-mapping response (not dict-like)  
**When** `list_models()` processes response  
**Then** raises `LLMClientError("Unexpected response when listing models")`  
**Source**: `llm_client.py:112-113`

### Missing/Invalid Data Field
**Given** response where `data` is not a list  
**When** `list_models()` processes response  
**Then** returns empty list `[]`  
**Source**: `llm_client.py:115-116`

---

## Completion Generation

### generate_completion(configuration, *, system_message, user_message, temperature=0.2, max_tokens=None, tools=None, tool_choice=None, response_format=None) -> LLMCompletion

### Incomplete Configuration
**Given** configuration where `is_configured()` is False  
**When** `generate_completion()` is called  
**Then** raises `LLMClientError("LLM configuration is incomplete")`  
**Source**: `llm_client.py:140-141`

### Request Payload Construction
**Given** valid configuration and messages  
**When** `generate_completion()` constructs payload  
**Then** includes: `model`, `messages` (system + user), `temperature`; conditionally includes `max_tokens`, `tools`, `tool_choice`, `response_format`  
**Source**: `llm_client.py:145-160`

### Message Format
**Given** system_message and user_message  
**When** payload is constructed  
**Then** messages array contains exactly `[{"role": "system", "content": system_message}, {"role": "user", "content": user_message}]`  
**Source**: `llm_client.py:148-151`

### Content Extraction
**Given** response with `choices[0].message.content`  
**When** processing completion  
**Then** strips whitespace; if empty after strip, sets content to None  
**Source**: `llm_client.py:178-183`

### Tool Call Extraction
**Given** response with `choices[0].message.tool_calls` array  
**When** processing completion  
**Then** extracts each tool call's `function.name` and `function.arguments`; skips entries with empty name; arguments dict/list are JSON-serialized to string  
**Source**: `llm_client.py:184-200`

### Fallback Text Field
**Given** response where `message.content` is None/empty but `choices[0].text` exists  
**When** processing completion  
**Then** uses `text` field as content (stripped)  
**Source**: `llm_client.py:203-207`

### No Content and No Tool Calls
**Given** response with no extractable content or tool calls  
**When** processing completion  
**Then** raises `LLMClientError("LLM response did not include text content")`  
**Source**: `llm_client.py:210`

### Non-Mapping Response
**Given** response is not a mapping type  
**When** processing completion  
**Then** raises `LLMClientError("Unexpected response from LLM")`  
**Source**: `llm_client.py:166`

### Empty Choices
**Given** response where `choices` is not a list or is empty  
**When** processing completion  
**Then** raises `LLMClientError("LLM response did not include choices")`  
**Source**: `llm_client.py:168-169`

### Invalid Choice Entry
**Given** `choices[0]` is not a mapping  
**When** processing completion  
**Then** raises `LLMClientError("LLM response choice was invalid")`  
**Source**: `llm_client.py:171-172`

---

## Configuration Contracts

| Source | Priority | Fields |
|--------|----------|--------|
| Environment variables | Higher | `ABOGEN_LLM_BASE_URL`, `ABOGEN_LLM_API_KEY`, `ABOGEN_LLM_MODEL`, `ABOGEN_LLM_TIMEOUT` |
| config.json | Lower | `llm_base_url`, `llm_api_key`, `llm_model`, `llm_timeout` |

Note: Configuration resolution is NOT in this module — handled by `normalization_settings.py`. This module only consumes `LLMConfiguration` instances.

---

## Thread Safety

All functions are stateless — no module-level mutable state. `_DEFAULT_HEADERS` is read-only dict. All dataclasses are frozen. Thread-safe by design.

---

## Invariants

1. **Zero external dependencies** — only stdlib `json`, `urllib`, `dataclasses`
2. **No streaming** — always reads full response body
3. **No retry logic** — single attempt per call
4. **Frozen dataclasses** — all data objects are immutable
5. **Content always stripped** — whitespace-only content becomes None
6. **Tool call arguments always string** — dict/list arguments are JSON-serialized
7. **Single exception type** — all errors raise `LLMClientError`
8. **URL normalization idempotent** — trailing slash always present after normalization
