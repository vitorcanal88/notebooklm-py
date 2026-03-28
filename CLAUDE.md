# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**IMPORTANT:** Follow documentation rules in [CONTRIBUTING.md](CONTRIBUTING.md) - especially the file creation and naming conventions.

## Project Overview

`notebooklm-py` is an unofficial Python client for Google NotebookLM that uses undocumented RPC APIs. The library enables programmatic automation of NotebookLM features including notebook management, source integration, AI querying, and studio artifact generation (podcasts, videos, quizzes, etc.).

**Critical constraint**: This uses Google's internal `batchexecute` RPC protocol with obfuscated method IDs that Google can change at any time. All RPC method IDs in `src/notebooklm/rpc/types.py` are undocumented and subject to breakage.

## Development Commands

```bash
# Create/recreate venv with uv (recommended - relocatable venvs)
uv venv .venv
uv pip install -e ".[all]"
playwright install chromium

# Activate virtual environment
source .venv/bin/activate

# Run all tests (excluding e2e by default)
pytest

# Run with coverage
pytest --cov

# Run e2e tests (requires authentication)
pytest tests/e2e -m e2e

# CLI testing
notebooklm --help
```

## Pre-Commit Checks (REQUIRED before committing)

**IMPORTANT:** Always run these checks before committing to avoid CI failures:

```bash
# Format code with ruff
ruff format src/ tests/

# Check for linting issues
ruff check src/ tests/

# Type checking with mypy
mypy src/notebooklm --ignore-missing-imports

# Run tests
pytest
```

Or use this one-liner:
```bash
ruff format src/ tests/ && ruff check src/ tests/ && mypy src/notebooklm --ignore-missing-imports && pytest
```

## Architecture

### Layered Design

```
CLI Layer (cli/)
    ↓
Client Layer (client.py, _*.py APIs)
    ↓
Core Layer (_core.py)
    ↓
RPC Layer (rpc/)
```

1. **RPC Layer** (`src/notebooklm/rpc/`):
   - `types.py`: All RPC method IDs and enums (source of truth)
   - `encoder.py`: Request encoding (triple-nested array structure)
   - `decoder.py`: Response parsing (XSSI stripping, chunked responses)

2. **Core Layer** (`src/notebooklm/_core.py`):
   - HTTP client management with granular timeouts (connect/read/write/pool)
   - RPC call abstraction with auto-retry on auth errors
   - Request counter (`_reqid_counter` starting at 100000)
   - Conversation cache (FIFO, max 100 entries)
   - Async lock for coordinated token refresh (concurrent callers share one task)

3. **Client Layer** (`src/notebooklm/client.py`, `_*.py`):
   - `NotebookLMClient`: Main async client with 8 namespaced APIs
   - Domain files: `_notebooks.py`, `_sources.py`, `_artifacts.py`, `_chat.py`, `_research.py`, `_notes.py`, `_sharing.py`, `_settings.py`

4. **CLI Layer** (`src/notebooklm/cli/`):
   - Modular Click commands with Rich terminal formatting
   - Context file tracks the "current" notebook for stateful sessions

### Key Files

| File | Purpose |
|------|---------|
| `client.py` | `NotebookLMClient` — async context manager, 8 sub-APIs |
| `_core.py` | HTTP + RPC infrastructure, auth refresh, conversation cache |
| `auth.py` | Cookie loading, CSRF/session extraction, regional domain handling |
| `types.py` | Dataclasses and enums (Notebook, Source, Artifact, etc.) |
| `exceptions.py` | 18 exception types organized by domain |
| `_notebooks.py` | `client.notebooks` API |
| `_sources.py` | `client.sources` API (add URL/text/file/YouTube/Drive) |
| `_artifacts.py` | `client.artifacts` API (generate + download all artifact types) |
| `_chat.py` | `client.chat` API (ask, conversation history) |
| `_research.py` | `client.research` API (fast/deep research, import) |
| `_notes.py` | `client.notes` API (create, update, delete, mind maps) |
| `_sharing.py` | `client.sharing` API (public link, per-user permissions) |
| `_settings.py` | `client.settings` API (output language) |
| `rpc/types.py` | RPC method IDs — **source of truth, never auto-generate** |
| `rpc/encoder.py` | `encode_rpc_request`, `build_request_body`, `build_url_params` |
| `rpc/decoder.py` | `decode_response`, `strip_anti_xssi`, `extract_rpc_result` |
| `cli/` | 20 Click command modules |

### Repository Structure

```
src/notebooklm/
├── __init__.py          # Public exports (30+ types, 18 exceptions)
├── client.py            # NotebookLMClient
├── auth.py              # Authentication (cookies, CSRF, session)
├── types.py             # Dataclasses (Notebook, Source, Artifact, Note…)
├── exceptions.py        # Exception hierarchy
├── paths.py             # Storage path resolution (NOTEBOOKLM_HOME)
├── _core.py             # Core infrastructure
├── _notebooks.py        # NotebooksAPI
├── _sources.py          # SourcesAPI
├── _artifacts.py        # ArtifactsAPI (largest file, 800+ lines)
├── _chat.py             # ChatAPI
├── _research.py         # ResearchAPI
├── _notes.py            # NotesAPI
├── _sharing.py          # SharingAPI
├── _settings.py         # SettingsAPI
├── _url_utils.py        # YouTube URL detection, auth redirect detection
├── _logging.py          # Logging configuration
├── _version_check.py    # Runtime Python ≥3.10 guard
├── notebooklm_cli.py    # CLI entry point (Windows asyncio fix)
├── rpc/
│   ├── types.py         # Method IDs and enums
│   ├── encoder.py       # Request encoding
│   └── decoder.py       # Response parsing
└── cli/
    ├── __init__.py
    ├── helpers.py        # get_client(), get_current_notebook(), run_async()
    ├── options.py        # Reusable Click option decorators
    ├── error_handler.py  # Error formatting
    ├── grouped.py        # Click command grouping
    ├── session.py        # login, use, status, clear
    ├── notebook.py       # list, create, delete, rename
    ├── source.py         # source add, list, delete, rename
    ├── artifact.py       # artifact list, delete, export
    ├── generate.py       # generate audio, video, quiz, flashcards, etc.
    ├── download.py       # download audio, video, infographic, slides, …
    ├── download_helpers.py  # Shared download utilities
    ├── chat.py           # ask, history, configure
    ├── note.py           # note create, update, list, delete
    ├── research.py       # research status, wait, import
    ├── share.py          # share, unshare, view-level
    ├── language.py       # output language config
    ├── agent.py          # Agent integration
    ├── skill.py          # Skills/tooling
    └── agent_templates.py  # Skill template generation
```

## API Patterns

### Client Usage

```python
# Correct pattern - async context manager, namespaced APIs
async with await NotebookLMClient.from_storage() as client:
    notebooks = await client.notebooks.list()
    await client.sources.add_url(nb_id, url)
    result = await client.chat.ask(nb_id, question)
    status = await client.artifacts.generate_audio(nb_id)
    await client.sharing.set_public(nb_id, public=True)
    await client.settings.set_output_language("en")
```

### Auth Precedence (highest to lowest)

1. Explicit `--storage` CLI flag
2. `NOTEBOOKLM_AUTH_JSON` environment variable (inline JSON)
3. `$NOTEBOOKLM_HOME/storage_state.json`
4. `~/.notebooklm/storage_state.json`

### CLI Structure

Commands are organized as:
- **Top-level**: `login`, `use`, `status`, `clear`, `list`, `create`, `ask`
- **Grouped**: `source add/list/delete/rename`, `artifact list/delete/export`, `generate audio/video/quiz/flashcards/infographic/slide-deck/data-table/mind-map`, `download audio/video/quiz/flashcards/infographic/slide-deck/mind-map/data-table`, `note create/update/list/delete`, `research start/status/import`, `share/unshare`

## Exception Hierarchy

```
NotebookLMError (base)
├── ValidationError
├── ConfigurationError
├── NetworkError          # Connection failures, timeouts
├── RPCError              # API-level errors
│   ├── AuthError         # 401/403 — re-login required
│   ├── RPCTimeoutError
│   ├── RateLimitError    # 429 — includes retry-after
│   ├── ServerError       # 5xx
│   ├── ClientError       # 4xx (non-auth)
│   └── DecodingError     # Malformed response
├── NotebookError
│   └── NotebookNotFoundError
├── SourceError
│   ├── SourceAddError
│   ├── SourceProcessingError
│   ├── SourceTimeoutError
│   └── SourceNotFoundError
├── ChatError
└── ArtifactError
    ├── ArtifactNotFoundError
    ├── ArtifactNotReadyError
    ├── ArtifactParseError
    └── ArtifactDownloadError
```

## RPC Protocol Details

### Endpoints

| Endpoint | Usage |
|----------|-------|
| `https://notebooklm.google.com/_/LabsTailwindUi/data/batchexecute` | Most RPCs |
| `https://notebooklm.google.com/_/LabsTailwindUi/data/google.internal.labs...` | Chat/query |
| `https://notebooklm.google.com/upload/_/` | File uploads |

### Request Format

```python
# Triple-nested array (position-sensitive, no field names)
[[["rpc_method_id", '["param1","param2",null]', null, "generic"]]]

# Request body
f.req=<url-encoded>&at=<csrf-token>&
```

### Response Format

- Anti-XSSI prefix: `)]}'\n` (always stripped before parsing)
- Chunked: multiple lines, each size-prefixed
- Error codes: 0 (OK), 400, 401, 403, 404, 429, 500

### RPC Parameter Nesting

Parameters are position-sensitive. Source ID nesting varies by method:
- `[id]` — single wrap
- `[[id]]` — double wrap
- `[[[id]]]` — triple wrap
- `[[[[id]]]]` — quad wrap

**Always check an existing working method before writing new params.**

## Artifact Download Formats

| Artifact Type | Available Formats |
|---------------|-------------------|
| Audio | MP3, MP4 |
| Video | MP4 |
| Report | Markdown |
| Quiz | JSON, Markdown, HTML |
| Flashcards | JSON, Markdown, HTML |
| Infographic | PNG |
| Slide Deck | PDF, PPTX |
| Mind Map | JSON |
| Data Table | CSV |

## Testing Strategy

- **Unit tests** (`tests/unit/`): Pure functions — encoding/decoding, no network
- **Integration tests** (`tests/integration/`): Mocked HTTP with VCR cassettes (`tests/cassettes/`)
- **E2E tests** (`tests/e2e/`): Real API, require auth, marked `@pytest.mark.e2e`

### Recording Integration Tests

```bash
# Record new VCR cassettes (requires real auth)
NOTEBOOKLM_VCR_RECORD=1 pytest tests/integration/
```

### E2E Test Status

- ✅ Notebook operations (list, create, rename, delete)
- ✅ Source operations (add URL/text/YouTube, rename)
- ✅ Download operations (audio, video, infographic, slides)
- ⚠️ Artifact generation may fail due to rate limiting

## Common Pitfalls

1. **RPC method IDs change**: All IDs in `rpc/types.py`. Google changes them without notice — capture network traffic to find new values.
2. **Nested list structures**: RPC params are position-sensitive. Check existing working implementations before writing new ones.
3. **Source ID nesting**: Different methods need `[id]`, `[[id]]`, `[[[id]]]`, or `[[[[id]]]]` — no single rule applies.
4. **CSRF tokens expire**: Use `client.refresh_auth()` or re-run `notebooklm login`. The core auto-retries once on auth failure.
5. **Rate limiting**: Add delays between bulk operations. `RateLimitError` includes a `retry_after` attribute.
6. **Media artifact completion**: Audio/video/infographic/slides must have a live download URL before status is truly complete — `wait_for_completion()` handles this.
7. **Conversation cache is local**: FIFO, max 100 entries, lost on client restart. Not synced to server.
8. **Windows asyncio**: The CLI entry point (`notebooklm_cli.py`) uses `SelectorEventLoop` fallback to avoid `ProactorEventLoop` hangs.
9. **Regional Google cookies**: Auth supports 60+ ccTLDs (`.google.co.uk`, `.google.com.sg`, etc.). Priority: `.google.com` > regional. Always preserve this logic.
10. **TYPE_CHECKING imports**: Used to avoid circular imports between `_artifacts.py` ↔ `_notes.py`. Keep this pattern when adding new cross-module types.

## Documentation

All docs use lowercase-kebab naming in `docs/`:

| File | Content |
|------|---------|
| `docs/cli-reference.md` | CLI commands and options |
| `docs/python-api.md` | Python API reference |
| `docs/configuration.md` | Storage and settings |
| `docs/troubleshooting.md` | Known issues |
| `docs/stability.md` | API versioning policy |
| `docs/development.md` | Architecture, testing, releasing |
| `docs/rpc-development.md` | RPC capture and debugging |
| `docs/rpc-reference.md` | RPC payload structures |
| `docs/examples/` | Runnable example scripts |
| `docs/scratch/` | Temporary investigation logs (YYYY-MM-DD-context.md) |

## When to Suggest CLI vs API

- **CLI**: Quick tasks, shell scripts, LLM agent automation
- **Python API**: Application integration, complex workflows, async operations

## Pull Request Workflow (REQUIRED)

After creating a PR, you MUST monitor and address feedback.

**Note:** This environment uses GitHub MCP tools (`mcp__github__*`) — not the `gh` CLI. Use `ToolSearch` to find the right tool name.

### 1. Monitor CI Status

Use `mcp__github__pull_request_read` to check PR status and CI checks. Repeat until all checks pass. If any fail, investigate and fix.

### 2. Check for Review Comments

Use `mcp__github__pull_request_read` or `mcp__github__list_pull_requests` to find open review threads.

### 3. Address Feedback

For each review comment (especially from `gemini-code-assist`):
1. Read and understand the feedback
2. Make the suggested fix if it improves the code
3. Commit with a descriptive message referencing the feedback
4. Push and re-check CI
5. Reply to the review thread using `mcp__github__add_reply_to_pull_request_comment` confirming the fix

### 4. Verify Final State

Use `mcp__github__pull_request_read` to confirm the PR is mergeable with all checks passing.

**Important**: Do NOT consider a PR complete until:
- All CI checks pass
- All review comments are addressed
- PR is in a mergeable state
