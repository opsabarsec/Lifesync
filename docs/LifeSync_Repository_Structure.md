# LifeSync Multi-Agent Assistant — Repository Handoff

## Purpose

LifeSync is a personal, single-user, Telegram-controlled multi-agent assistant. It combines financial, health, diet, and Google Workspace information through a central LLM router.

This document is intended as context for **Kimi Code running in a GitHub Codespace**. Use it to scaffold the repository and implement the first test-driven version.

## Non-Negotiable Constraints

- Use **Python 3.11+**.
- Use **`uv` exclusively** for Python dependencies and commands.
- Never use `pip` and do not create requirements files.
- Keep dependency declarations in `pyproject.toml` and commit `uv.lock`.
- Load configuration from the root `.env`; commit only `.env.example`.
- Kimi Code is the only coding assistant. GitHub Copilot and Claude Code are not required.
- Run development inside GitHub Codespaces.
- Start read-only. Require explicit confirmation for every external write.
- Do not enable payments, transfers, trading, or other financial mutations.
- Never expose credentials, tokens, full financial records, health records, or diet entries in logs or LLM prompts unless strictly necessary.

## Final Integration Decisions

| Domain | Provider | Strategy |
|---|---|---|
| Finance | Abby | Official REST API using a Bearer API key |
| Finance | Revolut Pro | Regulated Open Banking/AISP aggregator; statement import as fallback |
| Health | Polar Pacer | Pacer → Polar Flow → Polar AccessLink API v3 using OAuth2 |
| Diet | Manual spreadsheet | Local `.xlsx` workbook imported by `diet_tools.py` |
| Workspace | Google | Calendar, Tasks, and Drive wrappers with minimum scopes |
| Routing | OpenRouter | OpenAI-compatible client with structured routing output |
| Interface | Telegram | Allowlisted single-user bot entrypoint |


## Recommended Repository Structure

Use a proper package under `src/lifesync` so imports, packaging, and tests remain reliable:

```text
lifesync/
├── .devcontainer/
│   └── devcontainer.json       # Python 3.11+, uv, Codespaces setup
├── docs/
│   ├── architecture.md         # Routing and synchronization decisions
│   └── connections.md          # Provider setup and OAuth instructions
├── private_data/               # Local secrets/exports; entire directory gitignored
│   └── diet_log.xlsx           # User-maintained diet workbook; never commit
├── src/
│   └── lifesync/
│       ├── __init__.py
│       ├── __main__.py         # Supports: uv run python -m lifesync
│       ├── config.py           # Validated pydantic-settings configuration
│       ├── models.py           # Shared typed contracts
│       ├── orchestrator.py     # Telegram entrypoint, routing, confirmations
│       ├── agents/
│       │   ├── __init__.py
│       │   ├── base.py         # Common agent protocol
│       │   ├── financial_agent.py
│       │   └── health_agent.py
│       └── tools/
│           ├── __init__.py
│           ├── base.py         # ToolResult and ConnectionStatus contracts
│           ├── google_tools.py
│           ├── financial_tools.py  # Abby + Revolut read-only wrappers
│           ├── health_tools.py     # Polar AccessLink only
│           └── diet_tools.py       # Spreadsheet reader and validation
├── tests/
│   ├── unit/
│   │   ├── test_config.py
│   │   ├── test_orchestrator.py
│   │   ├── test_abby_connection.py
│   │   ├── test_revolut_connection.py
│   │   ├── test_polar_connection.py
│   │   └── test_diet_connection.py
│   ├── integration/
│   │   ├── test_financial_agent.py
│   │   ├── test_health_agent.py
│   │   └── test_telegram_flow.py
│   ├── live/
│   │   ├── test_abby_live.py
│   │   ├── test_revolut_live.py
│   │   ├── test_polar_live.py
│   │   └── test_diet_live.py
│   └── fixtures/
│       ├── abby/
│       ├── revolut/
│       ├── polar/
│       ├── google/
│       └── diet/
├── .env.example
├── .gitignore
├── .python-version            # 3.11
├── pyproject.toml
├── uv.lock
└── README.md
```

If `financial_tools.py` becomes too large, split provider implementations into `tools/abby_tools.py` and `tools/revolut_tools.py` while retaining a small financial facade for the agent.

## Module Responsibilities

### `config.py`

- Load `.env` through `pydantic-settings`.
- Validate required settings at startup.
- Keep provider settings optional until that provider is enabled.
- Never print secrets in validation errors.

### `models.py`

Define shared Pydantic models before implementing providers:

- `UserRequest`
- `RouteDecision`
- `AgentRequest`
- `AgentResponse`
- `ToolResult`
- `ConnectionStatus`
- `ConfirmationRequest`
- `BankAccount`, `Balance`, `BankTransaction`
- `Exercise`, `SleepRecord`, `ActivitySummary`
- `DietEntry`

### `orchestrator.py`

Own the complete request lifecycle:

1. Receive Telegram update.
2. Reject users not on the allowlist.
3. Normalize the request and assign a correlation ID.
4. Route through OpenRouter using structured output.
5. Ask for clarification when intent is ambiguous.
6. Invoke only the permitted agents/tools.
7. Require confirmation for writes.
8. Merge results and report partial failures or stale data.
9. Log sanitized status, latency, and token usage.

The LLM routes and summarizes; it must not control authentication or bypass tool permissions.

### `financial_agent.py`

- Read Abby organizations, contacts, invoices, and accounting data as needed.
- Read Revolut accounts, balances, and transactions through the selected provider.
- Search permitted financial/work documents through Google Drive.
- Preserve currencies and provider timestamps.
- Expose no payment or transfer functions.

### `health_agent.py`

- Read Polar activities, exercises, sleep, and recovery.
- Read manually recorded diet entries through `diet_tools.py`.
- Clearly distinguish automatic Polar data from manual diet data.
- Avoid medical diagnosis and unsupported health conclusions.

### `diet_tools.py`

- Read `DIET_DATA_PATH`, defaulting to `./private_data/diet_log.xlsx`.
- Require a worksheet named `diet`.
- Require headers: `date`, `meal`, `item`.
- Support optional headers: `time`, `amount`, `calories`, `protein_g`, `carbs_g`, `fat_g`, `notes`.
- Treat one row as one food or drink item.
- Preserve missing nutrient values as `None`, never zero.
- Report invalid rows with spreadsheet row numbers.
- Group entries by day and meal for summaries.
- Expose workbook modification time as the diet-data freshness timestamp.

## Environment Variables

Minimum application settings:

```dotenv
OPENROUTER_API_KEY=
OPENAI_API_BASE=https://openrouter.ai/api/v1
OPENROUTER_MODEL=
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOWED_USER_IDS=
APP_TIMEZONE=
LOG_LEVEL=INFO
DIET_DATA_PATH=./private_data/diet_log.xlsx
```

Provider settings are added only when implementing that connector:

```dotenv
ABBY_API_KEY=
ABBY_BASE_URL=https://api.app-abby.com

POLAR_CLIENT_ID=
POLAR_CLIENT_SECRET=
POLAR_REDIRECT_URI=
POLAR_ACCESS_TOKEN=
POLAR_USER_ID=

REVOLUT_PROVIDER=
REVOLUT_CONNECTION_ID=
```

Google credentials, OAuth token files, and aggregator-specific credentials must also be configurable. Production OAuth tokens should move from `.env` to protected token storage.

## Connection Requirements

### Abby

- Create the API key in Abby Settings → Integrations.
- Use Python `httpx`, not Abby's Node.js SDK.
- First smoke request: `GET /organizations?page=1&limit=1`.
- Send `Authorization: Bearer <ABBY_API_KEY>`.
- Keep the base URL configurable because Abby documentation shows both `api.app-abby.com` and `api.abby.fr` in different sections.
- Handle `401`, `403`, `404`, `429`, timeouts, and malformed JSON.

### Revolut Pro

- Do not use the Revolut Business API.
- Use a regulated Open Banking/AISP aggregator only after confirming support for the user's country, Revolut entity, and Revolut Pro account.
- Hide the aggregator behind a provider-neutral read interface.
- Use read-only account, balance, and transaction permissions.
- Treat expired or revoked consent as `REAUTHORIZATION_REQUIRED`.
- Respect transaction-history and unattended-refresh limits.
- If Pro is unsupported, use statement import rather than password automation.

### Polar Pacer

- Synchronize the Pacer to the user's Polar Flow account first.
- Register an AccessLink application and obtain client credentials.
- Use OAuth authorization-code flow with `accesslink.read_all` and a random `state` value.
- Exchange the code once, register the user with `POST /v3/users`, and store the Polar user ID.
- Verify with `GET /v3/users/{user-id}` before reading health data.
- Handle `409` registration and `403` mandatory-consent responses explicitly.
- Empty exercise results may mean the watch has not synchronized, not that authentication failed.
- Synchronize regularly; do not assume unlimited historical availability.

### Diet Spreadsheet

- No network connection or Yazio credentials are involved.
- Use `openpyxl` for `.xlsx` files.
- Reread the workbook on demand for the MVP.
- Do not calculate or present calories/macros as measured values when the user did not enter them.

## Test-First Requirements

Install development tools with:

```bash
uv add --dev pytest pytest-asyncio respx
```

Mark real-provider tests as `live`. Skip them when credentials/files are absent and exclude them from default CI.

### Abby

1. Bearer header and pagination are correct.
2. Invalid key produces a sanitized authentication error.
3. `429` does not trigger an aggressive retry loop.
4. Live smoke returns `200` and a list-valued `docs`; an empty list is valid.

### Revolut Pro

1. Consent configuration contains only read permissions and no payment scopes.
2. Account responses normalize account ID, currency, connection ID, and consent status.
3. Expired/revoked consent produces an actionable reauthorization state.
4. Live sandbox reads accounts only.
5. Real Pro acceptance runs only after manual user authorization and never hard-codes a balance.

### Polar

1. Token exchange uses Basic client authentication and form-encoded data.
2. User registration and user verification preserve the correct Polar identity.
3. Registration `409` and consent `403` are handled distinctly.
4. Exercise and sleep fixtures normalize correctly.
5. Live OAuth smoke reads user information, then sleep/exercises without requiring non-empty results.
6. Pacer acceptance records a short workout, synchronizes Flow, and verifies the workout appears through AccessLink.

### Diet Spreadsheet

1. Valid workbook imports correctly.
2. Missing required headers raise `InvalidDietSpreadsheet`.
3. Invalid rows include row numbers in errors.
4. Missing nutrient values remain valid and become `None`.
5. Weekly summaries group items by day and meal.
6. Live local smoke checks file existence, workbook readability, worksheet, and headers; a valid empty worksheet passes.

## Initial Dependencies

Add only as needed:

```bash
uv add openai python-telegram-bot httpx pydantic pydantic-settings tenacity openpyxl
uv add google-api-python-client google-auth google-auth-oauthlib
uv add --dev pytest pytest-asyncio respx ruff mypy
```

Routine commands:

```bash
uv sync
uv run pytest -m "not live"
uv run pytest -m live
uv run ruff check .
uv run mypy src
uv run python -m lifesync
```

## Implementation Order for Kimi Code

1. Scaffold this repository structure, `.gitignore`, `pyproject.toml`, and Codespaces configuration.
2. Implement configuration, shared models, sanitized provider errors, and connector contracts.
3. Create all mocked connection tests and sanitized fixtures before provider implementations.
4. Implement the diet importer and its tests.
5. Implement Abby and Polar read-only connections.
6. Run the Revolut Pro aggregator feasibility check; do not silently substitute the Business API.
7. Implement financial and health agents against the typed tools.
8. Implement OpenRouter routing and the Telegram allowlisted entrypoint.
9. Add Google Workspace read operations, then confirmation-protected writes.
10. Add daily/weekly briefings only after connection smoke tests pass.

## MVP Acceptance Criteria

- Default tests run without real credentials and make no live provider calls.
- Live tests are opt-in and read-only.
- Telegram rejects non-allowlisted users.
- Ambiguous requests produce clarification rather than arbitrary tool calls.
- A provider outage produces a partial response rather than failing the entire request.
- Diet responses identify the workbook's last-modified time.
- Polar responses distinguish missing watch synchronization from authentication failures.
- No Yazio code or configuration exists.
- No financial write capability exists.
- No secrets or personal fixture data are committed.
