# LifeSync

LifeSync is a personal, single-user, Telegram-controlled multi-agent assistant. It combines financial (Abby, Revolut Pro), health (Polar Pacer), diet (local `.xlsx` workbook), and Google Workspace information through a central OpenRouter LLM router.

- **Interface**: allowlisted Telegram bot
- **Finance**: read-only — Abby REST API + Revolut Pro via a regulated Open Banking aggregator (no payments, transfers, or trading)
- **Health**: Polar AccessLink API v3 (activities, sleep, recovery) + manual diet spreadsheet
- **Stack**: Python 3.11+, `uv` exclusively, pydantic, httpx; test-first with mocked provider tests and opt-in `live` smoke tests

See `docs/LifeSync_Repository_Structure.md` for the full design and implementation plan, and `AGENTS.md` for agent working guidance.

## Quick start

```bash
cp .env.example .env   # fill in your keys
uv sync
uv run pytest -m "not live"   # default test suite (no real credentials)
uv run python -m lifesync    # run the application
```
