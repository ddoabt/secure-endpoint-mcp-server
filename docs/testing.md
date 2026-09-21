# Testing Guide

This guide covers how tests in this repo are structured, how to run them, and how to
generate a coverage/JUnit report snapshot. For env vars, Docker usage, and feature
flags, see [`README.md`](../README.md) and [`docs/usage-guide.md`](usage-guide.md) —
this doc only covers testing.

## 1. Test Method

Tests live under `tests/`, one module per `secure_endpoint_mcp/` source module:

- `test_config.py` — `Settings` and env-var-derived feature flag parsing.
- `test_feature_flags.py` — `FeatureFlagManager` (enable/disable/group logic).
- `test_auth_client.py` — `AbsoluteAuthClient` request signing (GET/POST/custom
  headers/endpoints).
- `test_mcp_server.py` — `MCPServer` startup orchestration (spec fetch, tool
  generation, transport start/stop).

All tests are **unit tests with no live network or filesystem I/O**:

- External calls (`httpx`, the Absolute API client, `authlib` JWS signing, FastMCP's
  `FastMCPOpenAPI`) are replaced with `unittest.mock.patch` / `MagicMock`, either
  inline or via `pytest` fixtures (see the fixtures at the top of
  `tests/test_mcp_server.py` for the pattern).
- Environment variables are isolated per test with `mock.patch.dict(os.environ, ...,
  clear=True)` so tests don't leak state or depend on the developer's shell.
- Async code (`MCPServer.initialize`, `start`, `AbsoluteAuthClient.get/post`) is tested
  with `pytest-asyncio` in `auto` mode (configured in `pyproject.toml`) — `async def
  test_*` functions run without needing `@pytest.mark.asyncio` on each one.

There is no dedicated `test_schema_fix.py`; `schema_fix.py` is small (6 statements)
and is exercised indirectly through `test_mcp_server.py`'s spec-fetch tests. If it
grows more branches, give it its own test module.

## 2. Running Tests

```bash
# Install dev dependencies (pytest, pytest-asyncio, pytest-cov, mypy, black, isort)
uv pip install -e ".[dev]"

# Run the full suite
uv run pytest

# Run one file / one test
uv run pytest tests/test_mcp_server.py
uv run pytest tests/test_mcp_server.py::test_initialize

# Verbose output
uv run pytest -v
```

## 3. Coverage & Report Generation

```bash
# Coverage summary in the terminal, with missing line numbers
uv run pytest --cov=secure_endpoint_mcp --cov-report=term-missing

# Full report bundle (JUnit XML + HTML coverage + Cobertura XML coverage),
# written to test-reports/ (gitignored — regenerate on demand, don't commit)
uv run pytest \
  --junitxml=test-reports/junit.xml \
  --cov=secure_endpoint_mcp \
  --cov-report=term-missing \
  --cov-report=html:test-reports/htmlcov \
  --cov-report=xml:test-reports/coverage.xml
```

Open `test-reports/htmlcov/index.html` for a line-by-line coverage view.

A point-in-time markdown snapshot of a full run (pass/fail counts + per-module
coverage table) is checked in under `docs/test-reports/`, one file per run, named
`YYYY-MM-DD-<label>.md`. Add a new snapshot after any change that's expected to move
coverage or test counts meaningfully (new feature, bug fix with regression test,
dependency upgrade) — see [`2026-09-09-baseline.md`](test-reports/2026-09-09-baseline.md)
for the format.
