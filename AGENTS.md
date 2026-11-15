# AGENTS.md — agent_tracker (LifeSync)

Guidance for AI coding agents working in this repository. Read this together with `docs/LifeSync_Repository_Structure.md`, which is the authoritative design/handoff document.

## Current State of the Repository

**This repository is a greenfield scaffold.** It currently contains only:

- `README.md` — a one-line title (`# agent_tracker`).
- `docs/LifeSync_Repository_Structure.md` — the full design specification and implementation plan for the project described below.

There is no source code, no `pyproject.toml`, no tests, and no configuration yet. The first agent task is to scaffold the repository per the design doc (see "Implementation Order" in `docs/LifeSync_Repository_Structure.md`). Everything in this file describes the **intended** architecture; verify against actual code once it exists, and update this file when reality diverges.

## Project Overview

**LifeSync** is a personal, single-user, Telegram-controlled multi-agent assistant. It combines financial, health, diet, and Google Workspace information through a central LLM router (OpenRouter). The repository directory is named `agent_tracker` for historical reasons; the Python package will be `lifesync`.

Integration decisions (do not change without explicit user approval):

| Domain | Provider | Strategy |
|---|---|---|
| Finance | Abby | Official REST API with Bearer API key |
| Finance | Revolut Pro | Regulated Open Banking/AISP aggregator; statement import as fallback |
| Health | Polar Pacer | Polar Flow → Polar AccessLink API v3 via OAuth2 |
| Diet | Manual spreadsheet | Local `.xlsx` workbook read by `diet_tools.py` |
| Workspace | Google | Calendar, Tasks, Drive wrappers with minimum scopes |
| Routing | OpenRouter | OpenAI-compatible client, structured routing output |
| Interface | Telegram | Allowlisted single-user bot |

## Technology Stack

- **Python 3.11+** (`.python-version` pinned to 3.11).
- **`uv` exclusively** for all Python dependency and command management. Never use `pip`; never create requirements files. Dependencies live in `pyproject.toml`; `uv.lock` is committed.
- Planned libraries (add only as needed): `openai`, `python-telegram-bot`, `httpx`, `pydantic`, `pydantic-settings`, `tenacity`, `openpyxl`, `google-api-python-client`, `google-auth`, `google-auth-oauthlib`.
- Dev tools: `pytest`, `pytest-asyncio`, `respx`, `ruff`, `mypy`.
- Development runs inside a **GitHub Codespace** configured via `.devcontainer/devcontainer.json`.

## Planned Repository Layout

A `src/` layout package under `src/lifesync/`:

```text
src/lifesync/
├── __main__.py         # Entrypoint: uv run python -m lifesync
├── config.py           # Validated pydantic-settings configuration (loads root .env)
├── models.py           # Shared Pydantic contracts (UserRequest, RouteDecision, ToolResult, etc.)
├── orchestrator.py     # Telegram entrypoint: allowlist, routing, confirmations, lifecycle
├── agents/             # base.py protocol + financial_agent.py + health_agent.py
└── tools/              # base.py, google_tools.py, financial_tools.py, health_tools.py, diet_tools.py
tests/
├── unit/               # Mocked connection tests per provider
├── integration/        # Agent and Telegram flow tests
├── live/               # Opt-in real-provider smoke tests (marked `live`)
└── fixtures/           # Sanitized provider response fixtures (abby/, revolut/, polar/, google/, diet/)
```

If `financial_tools.py` grows too large, split it into `tools/abby_tools.py` and `tools/revolut_tools.py` behind a small financial facade.

Module contracts that matter:

- `config.py` — load `.env` via pydantic-settings; validate at startup; provider settings optional until enabled; never print secrets in errors.
- `orchestrator.py` — owns the full request lifecycle: Telegram update → allowlist check → normalize + correlation ID → OpenRouter structured routing → clarification on ambiguity → permitted tool invocation → confirmation for writes → merged result with partial-failure reporting → sanitized logging. The LLM routes and summarizes; it must not control authentication or bypass tool permissions.
- `diet_tools.py` — reads `DIET_DATA_PATH` (default `./private_data/diet_log.xlsx`); requires a `diet` worksheet with headers `date`, `meal`, `item` (optional: `time`, `amount`, `calories`, `protein_g`, `carbs_g`, `fat_g`, `notes`); missing nutrients stay `None`, never zero; errors report spreadsheet row numbers; exposes workbook mtime as the freshness timestamp.

## Build, Test, and Run Commands

```bash
uv sync                          # Install locked dependencies
uv run pytest -m "not live"      # Default test suite (no real credentials needed)
uv run pytest -m live            # Opt-in live provider smoke tests (read-only)
uv run ruff check .              # Lint
uv run mypy src                  # Type check
uv run python -m lifesync        # Run the application
```

Test-first rule: write mocked connection tests and sanitized fixtures **before** implementing each provider. Tests marked `live` must skip when credentials/files are absent and are excluded from default CI.

## Non-Negotiable Constraints

- Python 3.11+; `uv` exclusively (no pip, no requirements files).
- Configuration from root `.env`; commit only `.env.example`. The entire `private_data/` directory is gitignored (contains `diet_log.xlsx` and local secrets/exports).
- Kimi Code is the only coding assistant used on this project.
- Start read-only; require explicit confirmation for every external write.
- **No financial mutations**: no payments, transfers, or trading. Financial tools are read-only (Abby reads; Revolut via read-only Open Banking consent with no payment scopes).
- Do not use the Revolut **Business** API for Pro accounts; use a regulated AISP aggregator or statement import.
- Never expose credentials, tokens, full financial records, health records, or diet entries in logs or LLM prompts unless strictly necessary.
- No Yazio code or configuration anywhere.
- No secrets or personal data in committed fixtures.

## Environment Variables

Core (see `.env.example` once scaffolded): `OPENROUTER_API_KEY`, `OPENAI_API_BASE` (default `https://openrouter.ai/api/v1`), `OPENROUTER_MODEL`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USER_IDS`, `APP_TIMEZONE`, `LOG_LEVEL`, `DIET_DATA_PATH`. Provider keys (`ABBY_API_KEY`, Polar OAuth settings, Revolut aggregator settings) are added only with their connector. Production OAuth tokens should move from `.env` to protected token storage.

## Provider Connection Notes

- **Abby**: Python `httpx` (not the Node.js SDK); `Authorization: Bearer <key>`; first smoke `GET /organizations?page=1&limit=1`; keep base URL configurable (`api.app-abby.com` vs `api.abby.fr` both appear in docs); handle 401/403/404/429, timeouts, malformed JSON; no aggressive retry on 429.
- **Polar**: OAuth authorization-code flow with `accesslink.read_all` and random `state`; Basic client auth + form-encoded token exchange; register user via `POST /v3/users` (handle `409`), verify via `GET /v3/users/{user-id}`; handle consent `403` distinctly; empty exercise results usually mean the watch hasn't synced, not auth failure.
- **Diet**: `openpyxl` for `.xlsx`; reread on demand for MVP; never present calories/macros as measured when the user didn't enter them.

## Definition of Done (MVP Acceptance)

From the design doc — the implementation is not complete until: default tests pass with no real credentials; live tests are opt-in and read-only; Telegram rejects non-allowlisted users; ambiguous requests trigger clarification; a provider outage yields a partial response rather than total failure; diet responses report workbook last-modified time; Polar responses distinguish missing sync from auth failure; no Yazio code; no financial write capability; no secrets committed.
