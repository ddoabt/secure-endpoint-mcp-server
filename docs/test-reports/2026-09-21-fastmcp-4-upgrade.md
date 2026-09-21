# Test Report — 2026-09-21 (fastmcp 4.x upgrade)

See [`docs/testing.md`](../testing.md) for how this was generated and how to
reproduce it.

## Environment

- Python 3.13.3, pytest 9.1.1, pytest-asyncio 1.4.0, pytest-cov 7.1.0
- Command: `uv run pytest --junitxml=test-reports/junit.xml --cov=secure_endpoint_mcp --cov-report=term-missing --cov-report=html:test-reports/htmlcov --cov-report=xml:test-reports/coverage.xml`
- Commit: `31ff7ba` (branch `docs/init-spec-documents`)

## Result

**33 passed, 0 failed, 0 skipped** in 1.55s.

| Test file | Tests | Status |
|---|---|---|
| `tests/test_auth_client.py` | 6 | ✅ all passed |
| `tests/test_config.py` | 5 | ✅ all passed |
| `tests/test_feature_flags.py` | 9 | ✅ all passed |
| `tests/test_mcp_server.py` | 13 | ✅ all passed |

## Coverage — 97% (221/228 statements)

| Module | Stmts | Miss | Cover | Missing lines |
|---|---|---|---|---|
| `secure_endpoint_mcp/__init__.py` | 0 | 0 | 100% | — |
| `secure_endpoint_mcp/client/__init__.py` | 0 | 0 | 100% | — |
| `secure_endpoint_mcp/client/auth_client.py` | 46 | 1 | 98% | 147 |
| `secure_endpoint_mcp/config/__init__.py` | 0 | 0 | 100% | — |
| `secure_endpoint_mcp/config/logging.py` | 17 | 0 | 100% | — |
| `secure_endpoint_mcp/config/settings.py` | 30 | 0 | 100% | — |
| `secure_endpoint_mcp/feature_flags/__init__.py` | 0 | 0 | 100% | — |
| `secure_endpoint_mcp/feature_flags/manager.py` | 22 | 1 | 95% | 50 |
| `secure_endpoint_mcp/server/__init__.py` | 0 | 0 | 100% | — |
| `secure_endpoint_mcp/server/mcp_server.py` | 107 | 5 | 95% | 97-99, 160, 220 |
| `secure_endpoint_mcp/server/schema_fix.py` | 6 | 0 | 100% | — |
| **TOTAL** | **228** | **7** | **97%** | |

### Known coverage gaps (not regressions — pre-existing at this baseline)

- `auth_client.py:147` — the branch in `get`/`post` that appends to an already
  non-empty query string when extra `params` are passed.
- `feature_flags/manager.py:50` — a branch inside `is_api_enabled`'s group-membership
  loop.
- `mcp_server.py:97-99` — the generic `except Exception` path in `initialize`
  (previously lines 96-98; shifted by one line during the `FastMCP.from_openapi`
  migration).
- `mcp_server.py:160` — the line after the `httpx.get` call in `_fetch_openapi_spec`
  (the success path beyond the mocked response); previously line 159.
- `mcp_server.py:220` — the `ValueError` branch in `start` for an unsupported
  transport mode string that isn't one of the `TransportMode` enum's serialized
  values; previously line 219.

## Comparison to 2026-09-09 baseline

- Test count: 32 -> 33 (+1 for the new authlib.jose deprecation tripwire test in
  `tests/test_auth_client.py`, added in Task 3).
- Coverage: 97% -> 97% (unchanged as a percentage; the statement count grew from
  227 to 228 and covered statements from 220 to 221, because the `mcp_server.py`
  migration to `FastMCP.from_openapi` added one net statement while preserving the
  same 5 uncovered lines, now shifted down by one due to the added line).

Use this snapshot as the new baseline for future changes.
