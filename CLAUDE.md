# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`gemini-docs-mcp` is a local **STDIO MCP server** (built with [FastMCP](https://github.com/jlowin/fastmcp)) that gives MCP clients (IDEs, agents, etc.) tools to search and retrieve Google Gemini API documentation. On startup it scrapes `https://ai.google.dev/gemini-api/docs/llms.txt`, fetches every linked doc page, strips it to plain text, and stores it in a local SQLite database with an FTS5 full-text index. Clients then query that index via MCP tools instead of fetching docs live.

## Architecture

```
gemini_docs_mcp/
├── __init__.py     # empty package marker
├── config.py       # resolves DB_PATH (env var or ~/.mcp/gemini-api-docs/database.db)
├── ingest.py        # scrapes llms.txt + doc pages, upserts into SQLite FTS table
└── server.py        # FastMCP server: lifespan ingestion + the 3 MCP tools
```

- **`config.py`** – `get_db_path()` reads `GEMINI_DOCS_DB_PATH` env var, falling back to `~/.mcp/gemini-api-docs/database.db`. Creates the parent directory if missing. `DB_PATH` is computed once at import time.

- **`ingest.py`** – `ingest_docs()` is the entry point (also runnable standalone via `uv run python -m gemini_docs_mcp.ingest`):
  1. Fetches `LLMS_TXT_URL` (`https://ai.google.dev/gemini-api/docs/llms.txt`) and parses `- [Title](url)` lines via `parse_llms_txt`.
  2. For each link, `process_link` fetches the page (HTML stripped to text via BeautifulSoup, scripts/styles/header/footer/nav removed), hashes the content (SHA256), and **upserts only if the hash changed** — keeps re-ingestion cheap on subsequent server starts.
  3. Concurrency is bounded by `MAX_CONCURRENT_REQUESTS = 20` via an `asyncio.Semaphore`.
  4. The `docs` table (`url` PK, `title`, `content`, `content_hash`, `last_updated`) is created on first run with FTS5 enabled on `title`/`content` using **`tokenize="trigram"`** and `create_triggers=True` (so the FTS index stays in sync with upserts automatically).

- **`server.py`** – Defines the `mcp` FastMCP instance with `server_lifespan` (an `@asynccontextmanager`) that calls `ingest_docs()` before the server starts serving. Exposes three tools:
  - `search_documentation(queries: list[str])` – FTS5 keyword search, OR-combines up to several short queries, returns top `DB_TOP_K = 3` matching pages with full content. Prepends a hardcoded `[!WARNING]` block reminding the model that legacy SDKs/models exist and should be avoided.
  - `get_capability_page(capability: str = "")` – returns the full content of a doc page by exact title, or (if called with no argument) a list of all available titles.
  - `get_current_model()` – shortcut that looks up the page whose title contains "Gemini Models" (or whose URL contains `/models`).
  - `main()` calls `mcp.run()` and is the entry point registered as the `gemini-docs-mcp` console script.

## Key Conventions & Gotchas

- **FTS5 query sanitization**: `sanitize_term()` in `server.py` wraps any query term containing a `.` in double quotes (escaping existing quotes) because raw dots (e.g. `2.5`) cause FTS5 syntax errors. Any change to query handling must preserve this.
- **Trigram tokenizer**: the `docs` table FTS index uses `tokenize="trigram"`. Keep this in mind when changing search behavior — it affects how short/partial terms match.
- **SDK/version messaging is intentional**: the hardcoded warning in `search_documentation` and the tool descriptions explicitly steer model output away from the legacy `google-generativeai` / `@google/generative-ai` SDKs and Gemini ≤2.0 models toward `google-genai` / `@google/genai` and current models. This messaging exists because the eval harness (see below) checks generated code for exactly this. Don't remove it without understanding the eval implications.
- **Tool descriptions are prompt-engineered**: the docstrings/`description=` fields on the MCP tools (especially `search_documentation`) are tuned to get LLM clients to issue short, FTS-friendly keyword queries. Treat changes to these strings as prompt-engineering changes, not just docs — re-run the eval harness after editing them.
- **Upsert-by-hash**: ingestion is idempotent and incremental — `process_link` skips writes when `content_hash` is unchanged, so re-running ingestion on an existing DB is cheap.
- **Database is local/untracked**: `database.db` and `tests/generated/` are gitignored. Don't commit DB files or generated test scripts.

## Development Setup

This project uses **`uv`** for dependency management (Python ≥3.10, `uv.lock` committed).

```bash
# Install dependencies (including dev group)
uv sync

# Run the MCP server directly
uv run gemini-docs-mcp
# or
uv run python -m gemini_docs_mcp.server
```

Environment variables:
- `GEMINI_DOCS_DB_PATH` – overrides the SQLite DB location (default `~/.mcp/gemini-api-docs/database.db`).
- `GEMINI_API_KEY` (or other `google-genai` auth env vars) – required by `verify_gemini.py` and `tests/eval_harness.py`, which call the live Gemini API via `google-genai`.

## Verification Scripts (repo root)

These are manual/dev scripts, not a pytest suite:

- `verify_db.py` – directly runs FTS searches against the local DB (`db["docs"].search(...)`). Useful for checking ingestion/indexing without spinning up the MCP server. Run with optional query args: `uv run python verify_db.py "function calling" "embeddings"`.
- `verify_server.py` – spins up the MCP server as a subprocess via `mcp.client.stdio`, lists tools, and calls `search_documentation`. Good smoke test that the server starts and responds over stdio.
- `verify_gemini.py` – end-to-end check: connects a real `google-genai` client to the MCP server (as a tool) and asks it to generate code, exercising the full client → MCP → docs DB path.

## Evaluation Harness (`tests/`)

`tests/eval_harness.py` is the project's quality gate for generated code correctness:

```bash
uv run python -m tests.eval_harness --mode static    # default: static SDK-keyword analysis
uv run python -m tests.eval_harness --mode execute    # also executes generated Python scripts
```

- Reads prompts from `tests/test_prompts.json` (Python + TypeScript prompts asking for Gemini API code).
- For each prompt, calls `gemini-flash-latest` with the MCP server attached as a tool, extracts the generated code block, and:
  - **`static` mode**: checks whether the code uses the **new** SDK (`from google import genai` / `@google/genai`) vs. the legacy SDK (`google.generativeai` / `@google/generative-ai`, `GenerativeModel`, etc.) via `check_sdk_version_py` / `check_sdk_version_ts`. Pass = `new_sdk`.
  - **`execute` mode**: additionally runs generated Python scripts as subprocesses (60s timeout) and checks for a zero exit code.
- Generated code is written to `tests/generated/` (gitignored, recreated each run).
- Results are written to `tests/result.json` (`summary` + `failures`), and the README's "Test Results" table is manually updated from this file. **If you change tool descriptions, search behavior, or the SDK-warning content in `server.py`, re-run this harness and update both `tests/result.json` and the README table.**
- Exits non-zero if any test fails (suitable for CI use, though no CI workflow is currently configured).

## Docker

`Dockerfile` builds a `python:3.11-slim` image, installs deps with `uv sync --frozen --no-dev`, sets `GEMINI_DOCS_DB_PATH=/data/database.db`, and runs `uv run gemini-docs-mcp` as the entrypoint. There is no `.dockerignore` (it was intentionally removed — see git history).

## Commit Style

Recent history mixes short imperative messages (`update readme`, `fix queries`, `add notes`) and Conventional-Commit-style prefixes in Portuguese or English (`fix: corrigir erros no verify_db, server, ingest e README`). Either style is acceptable; prefer a short, descriptive summary of *why* the change was made.
