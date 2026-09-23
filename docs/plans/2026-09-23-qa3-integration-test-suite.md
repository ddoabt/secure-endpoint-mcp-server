# QA3 Real Integration Test Suite Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a real, separate, manually-invoked integration test that spawns the actual built Docker image over real MCP stdio and proves — against the real QA3 sandbox — that the fastmcp dependency upgrade shipped and requests are correctly signed end-to-end.

**Architecture:** A new `tests_qa3/` directory, sibling to `tests/`, outside `pyproject.toml`'s `testpaths` so it never runs in default `uv run pytest` or CI. One reusable fixture (`mcp_client`) wraps the subprocess+JSON-RPC pattern already hand-validated twice this session; one test scenario consumes it.

**Tech Stack:** Python 3.13, `pytest`, `docker` CLI (subprocess), raw JSON-RPC over stdio — no new dependencies.

**Spec:** `docs/spec/2026-09-23-qa3-integration-test-suite-design.md`

## Global Constraints

- `tests_qa3/` must stay outside `pyproject.toml`'s `testpaths = ["tests"]` — never add it there, and never wire it into `.github/workflows/ci.yml`.
- Credentials come from environment variables only (`API_HOST`, `API_KEY`, `API_SECRET`) — never written to a file, never logged, never hardcoded as a real value anywhere in committed code.
- This round is read-only only — no write/mutating tool calls anywhere in this suite.
- Items 2-5 from the spec's deferred scope (§6) are explicitly out of scope for every task below — do not add them opportunistically.
- The exact assertion strings in Task 2's test are provisional (spec §3.2) — they must be confirmed/corrected against real captured output from a real run before the task is done, not shipped as untested guesses.

---

## Task 1: `mcp_client` fixture + marker registration

**Files:**
- Create: `tests_qa3/conftest.py`
- Modify: `pyproject.toml` (add `markers` under `[tool.pytest.ini_options]`)

**Interfaces:**
- Consumes: nothing (first task).
- Produces: a pytest fixture `mcp_client` yielding an `MCPClient` instance with methods `call_tool(name: str, arguments: dict) -> dict`, `list_tools() -> list[dict]`, and attributes `initialize_result: dict` and `stderr_lines: list[str]`. Task 2 consumes exactly this interface.

- [ ] **Step 1: Register the `qa3_integration` marker**

In `pyproject.toml`, change:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = "test_*.py"
python_functions = "test_*"
asyncio_mode = "auto"
```

to:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = "test_*.py"
python_functions = "test_*"
asyncio_mode = "auto"
markers = [
    "qa3_integration: real-network test against the QA3 sandbox, run manually only (uv run pytest tests_qa3/)",
]
```

- [ ] **Step 2: Write `tests_qa3/conftest.py`**

```python
#
#  Copyright (c) 2025. Absolute Software Corporation. All rights reserved.
#
#  This software code is licensed under and subject to the terms of
#  the MIT License as set out in the License.txt file.
#

import json
import os
import subprocess
import threading
from typing import Any, Dict, Iterator, List

import pytest

_PLACEHOLDER_VALUES = {"dummy_key", "dummy_secret", "", None}


class MCPClient:
    """A real MCP client over stdio, driving the actual packaged Docker image.

    Speaks real JSON-RPC over the process's stdin/stdout -- no mocking
    anywhere in this call path. This is deliberately the same shape a real
    MCP client (Claude Code, Claude Desktop) uses, because the bug this suite
    exists to catch (fastmcp>=4.0 bypassing AbsoluteAuthClient's JWS signing)
    only shows up at this level, never in a test that imports Python
    functions directly.
    """

    def __init__(self, image: str) -> None:
        self._proc = subprocess.Popen(
            [
                "docker",
                "run",
                "-i",
                "--rm",
                "-e",
                "API_HOST",
                "-e",
                "API_KEY",
                "-e",
                "API_SECRET",
                "-e",
                "TRANSPORT_MODE=stdio",
                image,
            ],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
            bufsize=1,
            env=os.environ.copy(),
        )
        self.stderr_lines: List[str] = []
        self._stderr_thread = threading.Thread(target=self._drain_stderr, daemon=True)
        self._stderr_thread.start()
        self._next_id = 1
        self.initialize_result: Dict[str, Any] = self._handshake()

    def _drain_stderr(self) -> None:
        assert self._proc.stderr is not None
        for line in self._proc.stderr:
            self.stderr_lines.append(line.rstrip())

    def _send(self, message: Dict[str, Any]) -> None:
        assert self._proc.stdin is not None
        self._proc.stdin.write(json.dumps(message) + "\n")
        self._proc.stdin.flush()

    def _recv(self) -> Dict[str, Any]:
        assert self._proc.stdout is not None
        line = self._proc.stdout.readline()
        if not line:
            raise RuntimeError(
                "MCP server closed its stdout before responding "
                f"(stderr tail: {self.stderr_lines[-10:]})"
            )
        return dict(json.loads(line))

    def _handshake(self) -> Dict[str, Any]:
        self._send(
            {
                "jsonrpc": "2.0",
                "id": self._next_id,
                "method": "initialize",
                "params": {
                    "protocolVersion": "2024-11-05",
                    "capabilities": {},
                    "clientInfo": {"name": "tests_qa3", "version": "0.1.0"},
                },
            }
        )
        result = self._recv()
        self._next_id += 1
        self._send({"jsonrpc": "2.0", "method": "notifications/initialized"})
        return result

    def call_tool(self, name: str, arguments: Dict[str, Any]) -> Dict[str, Any]:
        self._send(
            {
                "jsonrpc": "2.0",
                "id": self._next_id,
                "method": "tools/call",
                "params": {"name": name, "arguments": arguments},
            }
        )
        self._next_id += 1
        return self._recv()

    def list_tools(self) -> List[Dict[str, Any]]:
        self._send({"jsonrpc": "2.0", "id": self._next_id, "method": "tools/list"})
        self._next_id += 1
        response = self._recv()
        return list(response.get("result", {}).get("tools", []))

    def close(self) -> None:
        assert self._proc.stdin is not None
        self._proc.stdin.close()
        try:
            self._proc.wait(timeout=15)
        except subprocess.TimeoutExpired:
            self._proc.kill()
        self._stderr_thread.join(timeout=2)


@pytest.fixture
def mcp_client() -> Iterator[MCPClient]:
    for name in ("API_HOST", "API_KEY", "API_SECRET"):
        value = os.environ.get(name)
        if value in _PLACEHOLDER_VALUES:
            raise pytest.UsageError(
                f"{name} is unset or a placeholder value. tests_qa3/ requires "
                "real QA3 credentials -- export API_HOST, API_KEY, API_SECRET "
                "yourself before running (see README.md's MCP Inspector "
                "section for the same pattern). Refusing to run against a "
                "placeholder value."
            )

    image = os.environ.get("MCP_QA3_IMAGE", "secure-endpoint-mcp-server:local")
    client = MCPClient(image)
    yield client
    client.close()
```

- [ ] **Step 3: Smoke-test the fixture with dummy credentials (plumbing only, no real signing claim)**

This step does NOT need real QA3 credentials — it only proves the fixture's process/JSON-RPC plumbing works, mirroring the very first Docker probe already done manually this session. Build the local image first if it doesn't exist (`docker build -t secure-endpoint-mcp-server:local .` from the repo root), then run:

```bash
API_HOST=https://api.absolute.com API_KEY=dummy_key API_SECRET=dummy_secret \
  uv run python -c "
import sys
sys.path.insert(0, 'tests_qa3')
from conftest import MCPClient
c = MCPClient('secure-endpoint-mcp-server:local')
print('serverInfo:', c.initialize_result['result']['serverInfo'])
tools = c.list_tools()
print('tool count:', len(tools))
c.close()
"
```

Expected: prints `serverInfo: {'name': 'OpenAPI Server', 'version': '4.0.5'}` (or whatever patch version is currently locked — must start with `4.`) and a tool count of at least 1. This is the same result already observed manually earlier this session with dummy credentials.

Also confirm the fail-fast guard works — run without exporting `API_KEY`:

```bash
unset API_KEY
uv run pytest tests_qa3/ -v 2>&1 | tail -20
```

Expected: fails fast with the `pytest.UsageError` message about placeholder/unset credentials — NOT a docker/connection error. (There's no test file yet at this point in the plan, so this specifically confirms collection doesn't error out and the fixture's guard fires correctly once Task 2 adds a consuming test — rerun this exact check at the end of Task 2 instead if `tests_qa3/` has no collectible tests yet and pytest reports "no tests ran" instead.)

- [ ] **Step 4: Run the existing default suite to confirm nothing else changed**

```bash
uv run pytest -q
```

Expected: `34 passed` (unchanged from before this task — this task only adds a new file and a `markers` entry, neither of which affects `tests/`).

- [ ] **Step 5: Run the CI gates locally**

```bash
uv run black --check tests_qa3
uv run isort --check-only tests_qa3
uv run mypy tests_qa3
```

Expected: all three exit 0. (`tests_qa3/` isn't covered by the `tests.*` mypy override that relaxes `disallow_untyped_defs`, so `conftest.py` must be fully typed as written above — it already is.)

- [ ] **Step 6: Commit**

```bash
git add tests_qa3/conftest.py pyproject.toml
git commit -m "test: add mcp_client fixture for real QA3 integration tests

Wraps the subprocess+stdio JSON-RPC pattern already hand-validated
twice this session into a reusable pytest fixture. tests_qa3/ stays
outside testpaths so it never runs in default pytest or CI."
```

---

## Task 2: The dependency-upgrade proof scenario

**Files:**
- Create: `tests_qa3/tier1_regressions/test_dependency_upgrade_integration.py`

**Interfaces:**
- Consumes: the `mcp_client` fixture from Task 1 (`call_tool`, `initialize_result`, `stderr_lines`).
- Produces: nothing further tasks depend on — this is the plan's terminal test scenario for this round.

**Why this needs a real run before it's done:** the spec (§3.2) explicitly flags the exact assertion strings as provisional — the real log line format, the real `serverInfo` shape, and the real `get_devices` response envelope must be confirmed against actual captured output, not assumed. This step requires real QA3 credentials (`API_HOST`, `API_KEY`, `API_SECRET` pointing at the sandbox) — obtain them the same way used earlier this session (the already-configured `absolute-security-local` or `absolute-security` Claude Code MCP entries carry them; export them as shell env vars, never paste the secret into a committed file or into this plan).

- [ ] **Step 1: Write the test**

```python
#
#  Copyright (c) 2025. Absolute Software Corporation. All rights reserved.
#
#  This software code is licensed under and subject to the terms of
#  the MIT License as set out in the License.txt file.
#

import json

import pytest


@pytest.mark.qa3_integration
def test_fastmcp_upgrade_signs_and_reaches_real_api(mcp_client):
    """
    Prove, against the real QA3 sandbox, that:
    1. The running Docker image actually shipped fastmcp>=4.0 (the upgrade
       this suite exists to verify).
    2. A real tool call succeeds end-to-end through the real MCP client
       protocol -- not mocked, not a Python-function import.
    3. The request was actually signed and reached the real Absolute API --
       the exact property the Critical send()-bypass bug violated (see
       docs/spec/2026-09-21-fastmcp-dependency-upgrade.md and fix commit
       3af5cfa).
    """
    server_info = mcp_client.initialize_result["result"]["serverInfo"]
    assert server_info["name"] == "OpenAPI Server"
    major, minor = (int(p) for p in server_info["version"].split(".")[:2])
    assert (major, minor) >= (
        4,
        0,
    ), f"expected fastmcp>=4.0 in the running image, got {server_info['version']}"

    result = mcp_client.call_tool("get_devices", {"pageSize": "1"})
    assert "error" not in result, result.get("error")
    payload = json.loads(result["result"]["content"][0]["text"])
    assert isinstance(payload.get("data"), list)

    jws_lines = [
        line
        for line in mcp_client.stderr_lines
        if "/jws/validate" in line and "HTTP Request: POST" in line
    ]
    assert jws_lines, (
        "no signed request to /jws/validate observed in container logs -- "
        f"stderr tail: {mcp_client.stderr_lines[-15:]}"
    )
    assert any("200 OK" in line for line in jws_lines), jws_lines
```

- [ ] **Step 2: Run it against real credentials and confirm/correct every assertion against real output**

```bash
export API_HOST=https://api.qa3.pd.aws-ca-central-1.faultless.ca
export API_KEY=<real value -- do not commit or paste into any file>
export API_SECRET=<real value -- do not commit or paste into any file>
uv run pytest tests_qa3/ -v -m qa3_integration
```

If any assertion fails because the real shape differs from what's drafted above (a different log message format, a different envelope key, a different `serverInfo` field name), fix the assertion to match the real, correct behavior — do not weaken what it verifies (the three numbered properties in the docstring must all still be checked) to force a pass. If the test fails because the underlying behavior is actually broken, stop and report that — do not paper over a real regression.

Expected once corrected: `1 passed`.

- [ ] **Step 3: Re-run the fail-fast guard check from Task 1 Step 3 now that a real test exists to collect**

```bash
unset API_KEY
uv run pytest tests_qa3/ -v -m qa3_integration 2>&1 | tail -20
```

Expected: fails with the `pytest.UsageError` guard message, not a docker/connection error.

- [ ] **Step 4: Confirm the default suite and CI gates are still unaffected**

```bash
uv run pytest -q
uv run black --check tests_qa3
uv run isort --check-only tests_qa3
uv run mypy tests_qa3
```

Expected: `34 passed` for the default suite; all three gates exit 0.

- [ ] **Step 5: Commit**

```bash
git add tests_qa3/tier1_regressions/test_dependency_upgrade_integration.py
git commit -m "test: add real QA3 scenario proving fastmcp upgrade + JWS signing

Confirmed against a real run against the QA3 sandbox -- see commit
message body or PR description for the captured evidence line."
```

Include in the commit message body (or note for the PR description) the actual `POST .../jws/validate ... 200 OK` line observed, as durable evidence this was really run, not just written.

---

## Task 3: Dated report snapshot

**Files:**
- Create: `docs/test-reports/YYYY-MM-DD-qa3-integration.md` (use today's actual date)

**Interfaces:**
- Consumes: the real, passing run from Task 2 Step 2 — must run after Task 2, using its actual captured output.

- [ ] **Step 1: Write the report using the real output from Task 2's run**

Follow the exact structure of `docs/test-reports/2026-09-09-baseline.md`, adapted for this suite. Do not invent any value — every number/line below must be copied from Task 2's actual terminal output.

```markdown
# Test Report — <today's date> (QA3 real integration)

See [`docs/spec/2026-09-23-qa3-integration-test-suite-design.md`](../spec/2026-09-23-qa3-integration-test-suite-design.md)
for the design this report validates, and [`tests_qa3/`](../../tests_qa3/)
for the suite itself.

## Environment

- Python 3.13.x, pytest <version from real run>
- Image: `secure-endpoint-mcp-server:local` (built from this branch's `Dockerfile`)
- Target: QA3 sandbox (`API_HOST=https://api.qa3.pd.aws-ca-central-1.faultless.ca`)
- Command: `uv run pytest tests_qa3/ -v -m qa3_integration`
- Commit: `<the commit that added this report>` (branch `docs/init-spec-documents`)

## Result

**1 passed** in <real elapsed time>s.

| Test | Status |
|---|---|
| `test_fastmcp_upgrade_signs_and_reaches_real_api` | ✅ passed |

## What this proves

- The running image reported `fastmcp` version `<real version from serverInfo>` (>= 4.0 required).
- `get_devices` succeeded end-to-end through the real MCP protocol (not mocked).
- Real signing evidence observed in container logs:
  ```
  <the real "HTTP Request: POST .../jws/validate ... 200 OK" line, copied verbatim>
  ```

This is the first automated (rather than manual/ad-hoc) confirmation that the
Critical `send()`-bypass fix (commit `3af5cfa`) holds against a real account,
not just a mocked unit test or a hand-run probe script.

## Deferred scope

Items 2-5 from `docs/spec/2026-09-23-qa3-integration-test-suite-design.md`
§6 (bodyless-write regression, dangling `$ref`, dropped array-body schema,
binary-response decoding) and general Tier 2 health checks are not covered
by this report — tracked as future work in that spec.
```

- [ ] **Step 2: Commit**

```bash
git add docs/test-reports/<today's date>-qa3-integration.md
git commit -m "docs: add QA3 real integration test report

First automated confirmation the Critical send()-bypass fix (3af5cfa)
holds against a real account."
```

---

## Self-Review

**Spec coverage:**
- §2 Goals (prove signing + real version) → Task 2's three-part assertion.
- §3 Architecture (`tests_qa3/` sibling directory, outside testpaths) → Task 1.
- §3.1 `mcp_client` fixture (exact interface: `call_tool`, `list_tools`, `initialize_result`, `stderr_lines`, fail-fast credential guard, `MCP_QA3_IMAGE` override) → Task 1 Step 2, implemented verbatim.
- §3.2 scenario (with explicit "confirm against real output" caveat) → Task 2, Step 2 is exactly that confirmation step, not skipped.
- §4 Credentials & execution (env vars only, never logged/committed, `uv run pytest tests_qa3/`, never in CI) → Global Constraints + every task's run commands.
- §5 Reporting (dated markdown, same convention as existing snapshots) → Task 3.
- §6 Deferred scope → explicitly excluded via Global Constraints and referenced (not built) in Task 3's report template.
- §7 Risks/rollback (read-only, no state mutation, easy to delete) → no task performs any write call; `get_devices` is the only real API call anywhere in this plan.
- Self-review note about marker registration (avoiding "unknown mark" warning) → Task 1 Step 1.

**Placeholder scan:** Task 3's report template has bracketed placeholders (`<real version from serverInfo>`, etc.) but the surrounding instruction is explicit that these must be copied from Task 2's actual output, not invented — same pattern already used successfully in the prior dependency-upgrade plan's Task 5.

**Type consistency:** `MCPClient`'s methods (`call_tool`, `list_tools`, attributes `initialize_result`, `stderr_lines`) are defined once in Task 1 and consumed with the exact same names in Task 2 — no drift.

---

Plan complete and saved to `docs/plans/2026-09-23-qa3-integration-test-suite.md`. Two execution options:

**1. Subagent-Driven (recommended for Task 1 only)** — Task 1 needs no real credentials (dummy creds suffice for its smoke test) and is fully mechanical, so a subagent can implement + a reviewer can verify it independently. Tasks 2 and 3 need real QA3 secrets to actually confirm/run — handing a fresh subagent a live credential in its dispatch prompt means that secret gets pasted into more transcripts/logs than necessary, so I'd recommend doing those two inline myself instead, sourcing the credentials the same safe way used earlier this session (never printed).

**2. Inline Execution** — I execute all three tasks in this session directly, with checkpoints between them.

Which approach?
