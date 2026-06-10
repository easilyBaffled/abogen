# LLM Client Analysis

Chunk: llm-client
File analyzed: `abogen/llm_client.py` (211 lines)

---

## Overview

Lightweight, dependency-free LLM client using only Python stdlib (urllib, json, dataclasses). Targets OpenAI-compatible chat completion APIs including Ollama and local providers. No streaming, no retry, no async.

---

## Public Classes

### LLMClientError(RuntimeError)
Custom exception for all LLM-related failures. Inherits RuntimeError.

### LLMConfiguration (frozen dataclass)
Fields:
- `base_url: str` -- API endpoint base URL
- `api_key: str` -- authentication token
- `model: str` -- model identifier
- `timeout: float = 30.0` -- request timeout in seconds

Method `is_configured() -> bool`: True when both `base_url` and `model` are non-empty after stripping. Note: `api_key` NOT required (supports Ollama/unauthenticated endpoints).

### LLMToolCall (frozen dataclass)
- `name: str` -- tool/function name
- `arguments: str` -- JSON string of arguments

### LLMCompletion (frozen dataclass)
- `content: Optional[str]` -- text response
- `tool_calls: Tuple[LLMToolCall, ...]` -- tool invocations

---

## Public Functions

### list_models(configuration) -> List[Dict[str, str]]
- GET request to `v1/models`
- Returns list of `{"id": ..., "label": ...}`
- Label priority: entry["name"] > entry["description"] > entry["id"]
- Skips entries without valid id

### generate_completion(configuration, *, system_message, user_message, temperature=0.2, max_tokens=None, tools=None, tool_choice=None, response_format=None) -> LLMCompletion
- POST to `v1/chat/completions`
- Two-message payload (system + user)
- Parses first choice; extracts content and/or tool_calls
- Fallback to `choices[0]["text"]` for non-standard APIs
- Raises LLMClientError if neither content nor tool_calls found

---

## URL Normalization Logic

`_normalized_base_url(base_url)`:
- Strips whitespace
- Ensures trailing slash
- Raises LLMClientError if empty

`_build_url(base_url, path)`:
- **v1 deduplication**: If base URL path ends with `/v1` AND path starts with `v1/`, strips `v1/` from path
- Uses `urllib.parse.urljoin` for assembly

Examples:
- `"http://host/v1"` + `"v1/chat/completions"` -> `http://host/v1/chat/completions` (no double v1)
- `"http://host/api"` + `"v1/chat/completions"` -> `http://host/api/v1/chat/completions`

---

## Authentication Handling

`_build_headers(api_key)`:
- Always: `Content-Type: application/json`, `Accept: application/json`
- **Ollama special case**: If key (case-insensitive) equals "ollama", NO Authorization header sent
- **Empty key**: NO Authorization header
- Otherwise: `Authorization: Bearer {token}`

---

## Tool-Call Support

**Input**: Standard OpenAI function-calling tool definitions. `tool_choice` for forced tool use.

**Response parsing**:
- Reads `message["tool_calls"]` as list
- Each entry needs `function.name` and `function.arguments`
- Arguments: if dict/list, serialized via json.dumps; else cast to str
- Entries without valid function name silently skipped

---

## Error Handling

`_perform_request(method, url, *, headers, payload, timeout)`:
- **HTTPError**: reads body, raises LLMClientError with status + body
- **URLError**: raises LLMClientError with reason (DNS, connection refused)
- **Generic Exception**: raises LLMClientError("LLM request failed")
- **Empty body**: returns None (not error)
- **Invalid JSON**: raises LLMClientError("LLM response was not valid JSON")

---

## Design Characteristics

1. Zero external dependencies (only stdlib)
2. Frozen dataclasses throughout (immutable)
3. Ollama-aware (skips auth for "ollama" key)
4. Defensive parsing (isinstance checks at every nesting level)
5. No streaming support (synchronous urlopen, reads entire body)
6. No retry logic (single attempt)
7. Single-turn only (always system + user, no conversation history)
8. No async support
