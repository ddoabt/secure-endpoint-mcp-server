# FastMCP/MCP Dependency Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bump `fastmcp` (2.12.4 → 4.x) and its transitive dependency tree to clear all 42 verified CVEs, add explicit version floors for the two security-critical transitive packages (`authlib`, `starlette`) that were invisible in `pyproject.toml`, and fix the resulting breaking-API fallout in this repo's code and tests.

**Architecture:** No architectural change — this is a dependency bump plus the mechanical code fixes it forces. `fastmcp` 4.x renamed/relocated the class this server instantiates (`FastMCPOpenAPI` → `FastMCP.from_openapi(...)`) and moved `MCPType` to a new module path; both call sites (`secure_endpoint_mcp/server/mcp_server.py` and `tests/test_mcp_server.py`) need updating to match. Everything else (route mapping, feature flags, JWS auth signing, schema-fix hook) keeps its current shape because the relevant hook signatures (`route_map_fn`, `mcp_component_fn`) are unchanged in 4.x.

**Tech Stack:** Python 3.13, `uv`/`uv.lock`, `fastmcp` 4.x, `mcp` 2.x, `authlib` 1.8.x, `starlette` 1.6.x, `pytest`/`pytest-asyncio`/`pytest-cov`, GitHub Actions CI, GitHub Dependabot.

**Spec:** `docs/spec/2026-09-03-fastmcp-dependency-upgrade-design.md` (see especially §1 for the verified 42-CVE count, §3/§3.1 for the `httpx2`/`OpenAPIProvider` rename findings, §5 for the `authlib`/`starlette` explicit-floor rationale, §6 for the testing-baseline tie-in).

## Global Constraints

- Dependency bounds (from spec §2/§5, verified against a real `uv lock` resolution in this plan's research): `fastmcp>=4.0,<5`, `authlib>=1.8,<2`, `starlette>=1.6,<2`. `mcp` stays transitive (no direct `pyproject.toml` entry) — its version is controlled entirely by `fastmcp`'s own pin and resolved `2.x` automatically.
- Non-goal (spec §2): no UAT remediation work (Group A/B items in the 2026-08-25 UAT spec) — if this bump incidentally fixes one, note it in the PR description only, do not investigate further here.
- Non-goal (spec §2): no behavior changes beyond what's required to keep the server working under the new dependency versions. Prefer the smallest working diff over "cleaner" alternatives the new API surface makes possible (e.g. do not migrate `schema_fix.py` to the new `validate_output=False` constructor kwarg — the existing `mcp_component_fn` hook still works unchanged, so leave it).
- Existing CI gates (`.github/workflows/ci.yml`) must stay green: `black --check .`, `isort --check-only .`, `mypy .`, `pytest -q --maxfail=1 --disable-warnings --cov=secure_endpoint_mcp --cov-report=term-missing --cov-report=xml`. CI already runs with `--disable-warnings`, so the new `AuthlibDeprecationWarning` (see Task 3) will not fail CI on its own.
- Test baseline to hold or exceed (`docs/test-reports/2026-09-09-baseline.md`): **32 passed, 0 failed, 97% coverage (220/227 statements)**. Per `docs/testing.md`, add a new dated snapshot after this upgrade.
- No sandbox credentials are available in CI — the live smoke test and manual MCP-client round trip (Task 6) are one-time human validation gates, not new automated tests (spec §6).

---

## Task 1: Bump dependency bounds and re-lock

**Files:**
- Modify: `pyproject.toml:7-16` (the `dependencies` list)
- Modify: `uv.lock` (regenerated, not hand-edited)

**Interfaces:**
- Consumes: nothing (first task).
- Produces: an installed environment with `fastmcp>=4.0,<5`, `mcp` resolved to `2.x`, `authlib>=1.8,<2`, `starlette>=1.6,<2` — every later task in this plan runs against this environment.

- [ ] **Step 1: Edit `pyproject.toml`'s dependency list**

Replace the current bare-name `dependencies` list with bounded constraints, and add `authlib`/`starlette` as new explicit direct dependencies (they were transitive-only before; see spec §5 for why they need to be named directly):

```toml
dependencies = [
    "fastmcp>=4.0,<5",
    "httpx>=0.28,<1",
    "pydantic>=2.11,<3",
    "pydantic-settings>=2.10,<3",
    "pytest>=8.4.1",
    "python-dotenv>=1.1,<2",
    "structlog>=25.4,<26",
    "html2text>=2025.4.15,<2026",
    "authlib>=1.8,<2",
    "starlette>=1.6,<2",
]
```

(The bound values for `httpx`/`pydantic`/`pydantic-settings`/`python-dotenv`/`structlog`/`html2text` are the currently-locked versions from `uv.lock` as of this plan's writing — `0.28.1`, `2.11.7`, `2.10.1`, `1.1.1`, `25.4.0`, `2025.4.15` respectively — floored at their current minor/calendar version with an open range to the next major. `authlib`/`starlette` upper-bounded at `<2` because `authlib` 2.0 removes the `authlib.jose` module this repo imports (see Task 3), and `starlette` has no 2.x release yet as of this plan's writing.)

- [ ] **Step 2: Re-lock**

```bash
uv lock --upgrade
```

Expected: succeeds (no resolution conflict — verified in this plan's research by running this exact bump in an isolated scratch copy of this repo's `pyproject.toml`). Confirm the key packages landed where expected:

```bash
grep -A1 '^name = "fastmcp"$\|^name = "mcp"$\|^name = "authlib"$\|^name = "starlette"$\|^name = "requests"$\|^name = "urllib3"$' uv.lock
```

Expected: `fastmcp` version starts with `4.`, `mcp` starts with `2.`, `authlib` is `>= 1.8`, `starlette` is `>= 1.6`, and **no** `requests`/`urllib3` lines print at all (they're removed from the dependency graph entirely under `fastmcp` 4.x — verified in this plan's research; if either package still appears, stop and re-check the resolution before continuing, since that means the removal this plan assumes didn't happen).

- [ ] **Step 3: Verify all previously-flagged CVEs are actually cleared**

Don't trust the bump by assumption — check OSV.dev directly against whatever versions actually got locked:

```bash
python3 - <<'EOF'
import json, re, urllib.request

packages = ["authlib", "fastmcp", "mcp", "starlette", "cryptography", "idna", "pygments"]
locked = {}
text = open("uv.lock").read()
for pkg in packages:
    m = re.search(rf'name = "{pkg}"\nversion = "([^"]+)"', text)
    if m:
        locked[pkg] = m.group(1)

def query(name, version):
    body = {"package": {"name": name, "ecosystem": "PyPI"}, "version": version}
    req = urllib.request.Request(
        "https://api.osv.dev/v1/query",
        data=json.dumps(body).encode(),
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    with urllib.request.urlopen(req, timeout=30) as resp:
        return json.load(resp)

failures = []
for name, version in locked.items():
    res = query(name, version)
    vulns = res.get("vulns", [])
    status = "CLEAN" if not vulns else f"{len(vulns)} FINDING(S): {[v['id'] for v in vulns]}"
    print(f"{name}=={version}: {status}")
    if vulns:
        failures.append(name)

if failures:
    raise SystemExit(f"Still vulnerable: {failures}")
print("\nAll checked packages clean.")
EOF
```

Expected: `All checked packages clean.` printed at the end, 0 exit code. If any package still shows findings, the resolved version for that package landed below its fully-patched release (see spec §1's fix-version table) — pin it explicitly higher in `pyproject.toml` and re-run Step 2.

- [ ] **Step 4: Sync the environment and confirm the expected breakage**

```bash
uv sync --extra dev
uv run pytest -q
```

Expected: **FAILS** with `ModuleNotFoundError: No module named 'fastmcp.server.openapi'` (raised from `tests/test_mcp_server.py`'s `from fastmcp.server.openapi import MCPType` and from `secure_endpoint_mcp/server/mcp_server.py`'s `from fastmcp.server.openapi import FastMCPOpenAPI, MCPType`). This is the expected, known break this plan's research identified — `fastmcp` 4.x renamed the module and the class (see Task 2). If you see a *different* failure, stop and investigate before proceeding — it means something this plan didn't anticipate changed.

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml uv.lock
git commit -m "build: bump fastmcp to 4.x, add explicit authlib/starlette floors

Clears all 42 CVEs verified against OSV.dev in the 2026-09-03 dependency
upgrade spec. Test suite is expected red until the next commit (fastmcp
4.x renamed FastMCPOpenAPI -> FastMCP.from_openapi)."
```

---

## Task 2: Migrate `mcp_server.py` and its test to the `fastmcp` 4.x API

**Files:**
- Modify: `secure_endpoint_mcp/server/mcp_server.py:8-38,100-105`
- Modify: `tests/test_mcp_server.py:12,418-437`

**Interfaces:**
- Consumes: the `fastmcp>=4.0,<5` environment from Task 1. Concretely depends on these `fastmcp` 4.x facts, verified in this plan's research by installing `fastmcp==4.0.5` into a scratch venv and reading the installed source directly:
  - `fastmcp.server.openapi` no longer exists. `FastMCPOpenAPI` no longer exists anywhere.
  - `fastmcp.FastMCP.from_openapi(openapi_spec, client=..., route_map_fn=..., mcp_component_fn=..., ...)` is a classmethod that builds a `FastMCP` app with an `OpenAPIProvider` attached, and accepts the exact same keyword names (`openapi_spec`, `client`, `route_map_fn`, `mcp_component_fn`) this repo already passes.
  - `MCPType` (with the same members: `TOOL`, `RESOURCE`, `RESOURCE_TEMPLATE`, `EXCLUDE`) now lives at `fastmcp.server.providers.openapi.MCPType`.
  - `FastMCP.run_http_async(transport="http"|"sse", host=..., port=...)` and `FastMCP.run_stdio_async()` keep the same signatures this repo already calls in `mcp_server.py::start`, so `start()`/`stop()` need no changes.
  - `route_map_fn(route, route_type)` and `mcp_component_fn(route, component)` are called with the same two positional arguments as before, so `MCPServer._route_map_fn` and `schema_fix.create_schema_fixing_component_fn` need no changes.
  - Passing `AbsoluteAuthClient` (an `httpx.AsyncClient` subclass) as `client=` still works but now emits `FastMCPDeprecationWarning("Passing an httpx.AsyncClient to OpenAPIProvider is deprecated ... Pass an httpx2.AsyncClient instead.")` — accepted for this PR per the Global Constraints non-goal (no behavior changes beyond what's required); migrating `AbsoluteAuthClient` to `httpx2` is out of scope here and is captured as an open question in the spec (§8).
- Produces: `MCPServer.app: Optional[FastMCP]` (was `Optional[FastMCPOpenAPI]`) — no other module reads `MCPServer.app`'s type today, so this is a self-contained change.

- [ ] **Step 1: Fix the imports and type annotation in `mcp_server.py`**

In `secure_endpoint_mcp/server/mcp_server.py`, change:

```python
from fastmcp.server.openapi import FastMCPOpenAPI, MCPType
```

to:

```python
from fastmcp import FastMCP
from fastmcp.server.providers.openapi import MCPType
```

And change the field declaration:

```python
        # FastMCP app instance (created during initialize)
        self.app: Optional[FastMCPOpenAPI] = None
```

to:

```python
        # FastMCP app instance (created during initialize)
        self.app: Optional[FastMCP] = None
```

- [ ] **Step 2: Switch the construction call from `FastMCPOpenAPI(...)` to `FastMCP.from_openapi(...)`**

In `initialize()`, change:

```python
        self.app = FastMCPOpenAPI(
            openapi_spec=self.openapi_spec,
            client=self.http_client,
            route_map_fn=self._route_map_fn,
            mcp_component_fn=create_schema_fixing_component_fn(disable_validation=True),
        )
```

to:

```python
        self.app = FastMCP.from_openapi(
            openapi_spec=self.openapi_spec,
            client=self.http_client,
            route_map_fn=self._route_map_fn,
            mcp_component_fn=create_schema_fixing_component_fn(disable_validation=True),
        )
```

- [ ] **Step 3: Fix the import and mock target in `tests/test_mcp_server.py`**

Change the import at the top of the file:

```python
from fastmcp.server.openapi import MCPType
```

to:

```python
from fastmcp.server.providers.openapi import MCPType
```

Then in `test_initialize_with_remote_spec`, change the mock target and assertions:

```python
        # Mock the FastMCPOpenAPI constructor
        with mock.patch(
            "secure_endpoint_mcp.server.mcp_server.FastMCPOpenAPI"
        ) as mock_fastmcp_openapi:
            # Mock the _extract_api_groups_from_openapi method
            with mock.patch.object(
                server, "_extract_api_groups_from_openapi"
            ) as mock_extract:
                # Initialize the server
                await server.initialize()

                # Assert that _extract_api_groups_from_openapi was called
                mock_extract.assert_called_once()

                # Assert that FastMCPOpenAPI was called with the correct parameters
                mock_fastmcp_openapi.assert_called_once()
                args, kwargs = mock_fastmcp_openapi.call_args
                assert kwargs["openapi_spec"] == sample_openapi_spec
                assert kwargs["client"] == server.http_client
                assert kwargs["route_map_fn"] == server._route_map_fn
```

to:

```python
        # Mock the FastMCP class so FastMCP.from_openapi(...) is a spy
        with mock.patch(
            "secure_endpoint_mcp.server.mcp_server.FastMCP"
        ) as mock_fastmcp_class:
            # Mock the _extract_api_groups_from_openapi method
            with mock.patch.object(
                server, "_extract_api_groups_from_openapi"
            ) as mock_extract:
                # Initialize the server
                await server.initialize()

                # Assert that _extract_api_groups_from_openapi was called
                mock_extract.assert_called_once()

                # Assert that FastMCP.from_openapi was called with the correct parameters
                mock_fastmcp_class.from_openapi.assert_called_once()
                args, kwargs = mock_fastmcp_class.from_openapi.call_args
                assert kwargs["openapi_spec"] == sample_openapi_spec
                assert kwargs["client"] == server.http_client
                assert kwargs["route_map_fn"] == server._route_map_fn
```

- [ ] **Step 4: Run the full suite and confirm green**

```bash
uv run pytest -q
```

Expected: `32 passed` (same count as the 2026-09-09 baseline — this task only changes import paths and a mock target, not behavior, so the test count shouldn't move).

- [ ] **Step 5: Run the full CI gate locally**

```bash
uv run black --check secure_endpoint_mcp tests
uv run isort --check-only secure_endpoint_mcp tests
uv run mypy secure_endpoint_mcp
```

Expected: all three exit 0. (`mypy` matters here specifically — the `self.app: Optional[FastMCP]` annotation change is exactly the kind of thing `mypy` would catch if `FastMCP` weren't imported or the type didn't match what `FastMCP.from_openapi` actually returns.)

- [ ] **Step 6: Commit**

```bash
git add secure_endpoint_mcp/server/mcp_server.py tests/test_mcp_server.py
git commit -m "fix: migrate MCPServer to fastmcp 4.x's FastMCP.from_openapi API

fastmcp 4.x removed FastMCPOpenAPI and fastmcp.server.openapi entirely;
MCPType moved to fastmcp.server.providers.openapi. route_map_fn and
mcp_component_fn hook signatures are unchanged, so no other logic moves."
```

---

## Task 3: Lock in the new `authlib.jose` deprecation warning as a regression test

**Files:**
- Modify: `tests/test_auth_client.py` (add one new test function)

**Interfaces:**
- Consumes: `authlib>=1.8,<2` from Task 1. `secure_endpoint_mcp/client/auth_client.py:14`'s existing `from authlib.jose import JsonWebSignature` import is unchanged — this task only adds test coverage for a newly-observed side effect of that same import.
- Produces: a tripwire test that fails loudly if a future `authlib` upgrade either removes `authlib.jose` (the deprecation's stated end state, "before version 2.0.0") or silently drops the warning.

**Why this task exists:** verified in this plan's research that `authlib` 1.8.0 (unlike the currently-locked 1.6.0) emits `AuthlibDeprecationWarning: authlib.jose module is deprecated, please use joserfc instead. It will be compatible before version 2.0.0.` the moment `authlib.jose` is imported — which happens at module load time in `auth_client.py`. CI runs with `--disable-warnings` so this won't fail the build, but it's a real, newly-introduced behavior change worth locking in per the spec's testing strategy (§6: "Add new regression coverage only if step 1's verification uncovers a real behavior change worth locking in").

- [ ] **Step 1: Write the test**

Add to `tests/test_auth_client.py`:

```python
def test_authlib_jose_import_emits_deprecation_warning():
    """authlib>=1.8 deprecates the authlib.jose module (used by auth_client.py
    for JWS signing) in favor of joserfc, with removal planned for authlib
    2.0 (excluded by pyproject.toml's `authlib<2` bound). This test is a
    tripwire: if it starts failing, authlib.jose was either removed (migrate
    auth_client.py to joserfc) or stopped warning (safe to delete this test).
    """
    import importlib
    import warnings

    import authlib.jose
    from authlib.deprecate import AuthlibDeprecationWarning

    with warnings.catch_warnings(record=True) as caught:
        warnings.simplefilter("always")
        importlib.reload(authlib.jose)

        assert any(
            issubclass(w.category, AuthlibDeprecationWarning)
            and "authlib.jose module is deprecated" in str(w.message)
            for w in caught
        )
```

- [ ] **Step 2: Run it and confirm it passes immediately**

```bash
uv run pytest tests/test_auth_client.py::test_authlib_jose_import_emits_deprecation_warning -v
```

Expected: `PASSED`. Unlike a typical TDD cycle, there's no red step here — this test characterizes real third-party behavior that Task 1 already introduced; it isn't testing new code this task writes.

- [ ] **Step 3: Run the full suite to confirm no count regression**

```bash
uv run pytest -q
```

Expected: `33 passed` (32 from the baseline + this one new test).

- [ ] **Step 4: Commit**

```bash
git add tests/test_auth_client.py
git commit -m "test: lock in authlib.jose deprecation warning introduced by authlib 1.8

Tripwire for the eventual authlib 2.0 removal of authlib.jose (this repo
pins authlib<2, so migration to joserfc is a future, separate task)."
```

---

## Task 4: Add GitHub Dependabot config for the Python ecosystem

**Files:**
- Create: `.github/dependabot.yml`

**Interfaces:**
- Consumes: nothing from earlier tasks — independent of the code/dependency changes, included in this plan because the spec (§2, §4 step 6) scopes it into the same PR.
- Produces: a Dependabot config a reviewer can approve or reject independently of Tasks 1-3.

- [ ] **Step 1: Create the config**

```yaml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

- [ ] **Step 2: Validate the YAML syntax**

```bash
uv tool run yamllint .github/dependabot.yml
```

Expected: no output, exit 0 (a clean `yamllint` run prints nothing for a file with no style violations).

- [ ] **Step 3: Commit**

```bash
git add .github/dependabot.yml
git commit -m "ci: add Dependabot for the Python ecosystem

authlib/starlette are now explicit direct dependencies (Task 1), so
Dependabot tracks them by name instead of leaving them invisible inside
the next fastmcp/mcp bump."
```

---

## Task 5: Regenerate the coverage/test snapshot

**Files:**
- Create: `docs/test-reports/2026-09-21-fastmcp-4-upgrade.md`

**Interfaces:**
- Consumes: the fully green suite from Tasks 1-3 (must run after those land).
- Produces: a dated snapshot per `docs/testing.md`'s documented rule ("Add a new snapshot after any change that's expected to move coverage or test counts meaningfully ... dependency upgrade").

- [ ] **Step 1: Generate the report bundle**

```bash
uv run pytest \
  --junitxml=test-reports/junit.xml \
  --cov=secure_endpoint_mcp \
  --cov-report=term-missing \
  --cov-report=html:test-reports/htmlcov \
  --cov-report=xml:test-reports/coverage.xml
```

- [ ] **Step 2: Write the dated markdown snapshot**

Follow the exact format of `docs/test-reports/2026-09-09-baseline.md`. Fill in the real pass/fail counts and per-module coverage table from Step 1's terminal output — do not guess these numbers, copy them from the actual run. Example structure (values are placeholders for the actual run's output, which you must substitute):

```markdown
# Test Report — 2026-09-21 (fastmcp 4.x upgrade)

See [`docs/testing.md`](../testing.md) for how this was generated and how to
reproduce it.

## Environment

- Python 3.13.x, pytest <version>, pytest-asyncio <version>, pytest-cov <version>
- Command: `uv run pytest --junitxml=test-reports/junit.xml --cov=secure_endpoint_mcp --cov-report=term-missing --cov-report=html:test-reports/htmlcov --cov-report=xml:test-reports/coverage.xml`
- Commit: `<this task's commit sha>` (branch `<branch name>`)

## Result

**<N> passed, 0 failed, 0 skipped** in <T>s.

| Test file | Tests | Status |
|---|---|---|
| `tests/test_auth_client.py` | 6 | ✅ all passed |
| `tests/test_config.py` | 5 | ✅ all passed |
| `tests/test_feature_flags.py` | 9 | ✅ all passed |
| `tests/test_mcp_server.py` | 13 | ✅ all passed |

## Coverage — <X>% (<covered>/<total> statements)

<copy the per-module table from the terminal --cov-report=term-missing output>

## Comparison to 2026-09-09 baseline

- Test count: 32 -> <N> (+1 for the new authlib.jose deprecation tripwire, Task 3).
- Coverage: 97% -> <X>% (<call out any change and why, or "unchanged">).

Use this snapshot as the new baseline for future changes.
```

- [ ] **Step 3: Commit**

```bash
git add docs/test-reports/2026-09-21-fastmcp-4-upgrade.md
git commit -m "docs: add post-upgrade test-report snapshot

Per docs/testing.md's rule to snapshot after dependency upgrades."
```

---

## Task 6: Manual validation (not automated — one-time gate per spec §6)

**Files:** none modified — this task's deliverable is a paragraph in the PR description, not a commit.

**Interfaces:**
- Consumes: Tasks 1-3 merged into the branch, a real Absolute sandbox/QA account's API key+secret, and a local Claude Desktop or Claude Code MCP client config pointed at this server.

- [ ] **Step 1: Live read-only smoke test**

With real sandbox credentials in the environment (never commit these), start the server and call one read-only tool (e.g. `get_policy_group_v2`, used as a read-only repro target in the 2026-08-25 UAT spec) through an actual MCP client. Confirm: the request signs successfully via `AbsoluteAuthClient` (real `authlib` 1.8.x JWS signing, not mocked), and a `200` with real data comes back.

- [ ] **Step 2: Live bodyless-write smoke test**

Call one bodyless-write tool (e.g. `policy_groups_activate_policies`, the same safe/reversible repro target from the UAT spec — `deviceCount: 0` policy group, disabled target policy) through the same real MCP client. Confirm the request still signs and sends (this is the exact code path Task 2 touched: `AbsoluteAuthClient` passed as `client=` into `FastMCP.from_openapi`). The *response* may still be the pre-existing `503` bug described in the UAT spec's Group A (that fix is explicitly out of scope here) — the thing being validated is that the request is signed and sent at all, not that the response is correct.

- [ ] **Step 3: Manual MCP-client round trip**

Point Claude Desktop or Claude Code at this server over its actual configured transport (stdio or HTTP, whichever this deployment uses) and confirm: tool discovery lists the expected tools, and at least one real tool call succeeds end-to-end through the real transport (not the mocked unit-test transport).

- [ ] **Step 4: Record results in the PR description**

Write one paragraph per step above stating pass/fail and any observations (including, per spec §8's open question, whether any UAT Group B symptom appeared fixed or changed as a side effect — note it for the record, do not investigate further).

---

## Self-Review

**Spec coverage:**
- §2 Goals (bump + bounds + Dependabot) → Tasks 1, 4.
- §2 Goals (explicit `authlib`/`starlette` floors) → Task 1, Step 1.
- §3/§3.1 (`httpx2` coexistence confirmed, `OpenAPIProvider`/`FastMCP.from_openapi` rename) → Task 2 (grounded in real source read from a scratch `fastmcp==4.0.5` install, not release notes).
- §4 step 1 (spike/verify) → done as part of writing this plan (recorded in Task 2's Interfaces block) rather than as a separate execution-time task, since re-doing it at execution time would just reproduce the same findings.
- §4 step 3 (fix breaks: `schema_fix.py`/`mcp_server.py`) → Task 2. `schema_fix.py` itself needs no change (verified `OpenAPITool.output_schema` is still a plain settable attribute in 4.x), noted explicitly in Task 2's Interfaces and the Global Constraints non-goal.
- §4 step 4 (validate: CI gate + live smoke test + manual round trip) → Task 2 Step 5 (CI gate) and Task 6 (live/manual).
- §4 step 5 (re-verify CVEs cleared) → Task 1 Step 3.
- §4 step 6 / §2 Goals (Dependabot) → Task 4.
- §5 (best-practice rationale for which transitive deps get explicit floors) → reflected directly in Task 1's dependency list (only `authlib`/`starlette` added; `cryptography`/`idna`/`pygments` left alone; `requests`/`urllib3` expected gone).
- §6 Testing strategy (existing suite unchanged behavior, new coverage only for real discovered behavior change) → Task 2 (no new tests, just fixes) and Task 3 (new test, justified by a real discovered behavior change: the `authlib.jose` deprecation warning).
- Baseline tie-in (`docs/testing.md`'s "snapshot after dependency upgrades" rule, raised in this session before this plan existed) → Task 5.
- §7 Rollback: no dedicated task — rollback is "revert these commits," which is already true given each task is a self-contained commit in sequence.
- §8 Open questions: `httpx2` question already resolved (noted in Task 2's Interfaces); UAT Group B incidental-fix note folded into Task 6 Step 4.

**Placeholder scan:** no "TBD"/"handle appropriately" patterns. Task 5's markdown template has explicit placeholder *values* (`<N>`, `<X>%`) but they're inside a "copy the real numbers here" instruction, not a skipped implementation step — the surrounding instructions are explicit that these must come from the actual run's output, not be invented.

**Type consistency:** `self.app: Optional[FastMCP]` (Task 2) is the only type touched; no other task references it. `mock_fastmcp_class.from_openapi` (Task 2, test) matches the real call site `FastMCP.from_openapi(...)` (Task 2, source) exactly.

---

Plan complete and saved to `docs/plans/2026-09-21-fastmcp-dependency-upgrade.md`. Two execution options:

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?**
