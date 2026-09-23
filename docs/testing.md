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

## 4. Real Integration Tests (`tests_qa3/`)

`tests_qa3/` is a **separate** suite from `tests/` — it has real network I/O, spawns
the actual built Docker image, and talks to a real QA3 sandbox account. It exists to
catch exactly the class of bug unit tests structurally cannot: `tests/` mocks the
transport layer, so a bug in *how this server integrates with fastmcp/httpx* (like the
`send()`-vs-`request()` JWS-signing bypass fixed in commit `3af5cfa`) can pass every
unit test while being completely broken in production. See
[`docs/spec/2026-09-23-qa3-integration-test-suite-design.md`](spec/2026-09-23-qa3-integration-test-suite-design.md)
for the full design rationale.

**This suite never runs in default `uv run pytest` or in CI.** `pyproject.toml`'s
`testpaths = ["tests"]` excludes it by construction — it's invisible unless you point
pytest at it directly.

### Credential handling — read this before running it

The suite needs real `API_HOST`, `API_KEY`, `API_SECRET` for the QA3 sandbox. **Never
put these in a file that could get committed** — no `.env`, no hardcoded values
anywhere in `tests_qa3/`. The fixture (`tests_qa3/conftest.py`) only ever reads them
from environment variables, and fails fast with a clear error if they're unset or a
placeholder value — that's intentional, not a bug, so don't work around it by setting
dummy values (the suite proves nothing against a placeholder).

Export them into your shell for the one terminal session you're testing in, then run
the suite in that same session:

```bash
export API_HOST=https://api.qa3.pd.aws-ca-central-1.faultless.ca
export API_KEY=<your real QA3 key>
export API_SECRET=<your real QA3 secret>
uv run pytest tests_qa3/ -v -m qa3_integration
```

If your credentials already live in a tool config you don't want to hand-copy from
(e.g. a Claude Code MCP server entry in `~/.claude.json`), extract and `export` them
into your shell programmatically rather than pasting the values anywhere — the point
is that the secret should only ever exist in your shell's process environment and the
test container's process environment, never in a file, a script argument, or anything
you might `git add`.

### Running against the built image

```bash
docker build -t secure-endpoint-mcp-server:local .
uv run pytest tests_qa3/ -v -m qa3_integration
```

`MCP_QA3_IMAGE` overrides the image tag if you want to test something other than the
local build (e.g. a published `ghcr.io/...` release).

### Reports

Same convention as `docs/test-reports/` above — a dated snapshot capturing pass/fail
and *observational evidence only* (a version string, an HTTP status line) — never a
credential value. See
[`2026-09-23-qa3-integration.md`](test-reports/2026-09-23-qa3-integration.md) for the
format.
