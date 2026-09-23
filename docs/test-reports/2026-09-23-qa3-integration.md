# Test Report — 2026-09-23 (QA3 real integration)

See [`docs/spec/2026-09-23-qa3-integration-test-suite-design.md`](../spec/2026-09-23-qa3-integration-test-suite-design.md)
for the design this report validates, and [`tests_qa3/`](../../tests_qa3/)
for the suite itself.

## Environment

- Python 3.13.3, pytest 9.1.1
- Image: `secure-endpoint-mcp-server:local` (built from this branch's `Dockerfile`)
- Target: QA3 sandbox (`API_HOST=https://api.qa3.pd.aws-ca-central-1.faultless.ca`)
- Command: `uv run pytest tests_qa3/ -v -m qa3_integration`
- Commit: this report's own commit (branch `docs/init-spec-documents`)

## Result

**1 passed** in 4.90s.

| Test | Status |
|---|---|
| `test_fastmcp_upgrade_signs_and_reaches_real_api` | ✅ passed |

## What this proves

- The running image reported `fastmcp` version `4.0.5` (>= 4.0 required).
- `get_devices` succeeded end-to-end through the real MCP protocol (not mocked).
- Real signing evidence observed in container logs:
  ```
  HTTP Request: POST https://api.qa3.pd.aws-ca-central-1.faultless.ca/jws/validate "HTTP/1.1 200 OK"
  ```

This is the first automated (rather than manual/ad-hoc) confirmation that the
Critical `send()`-bypass fix (commit `3af5cfa`) holds against a real account,
not just a mocked unit test or a hand-run probe script.

## Deferred scope

Items 2-5 from `docs/spec/2026-09-23-qa3-integration-test-suite-design.md`
§6 (bodyless-write regression, dangling `$ref`, dropped array-body schema,
binary-response decoding) and general Tier 2 health checks are not covered
by this report — tracked as future work in that spec.

Four minor findings from this round's task and final reviews are also tracked
as future work rather than fixed here (none weaken what this report proves):

- `MCPClient.close()` doesn't `wait()` after `kill()` on a timeout, leaving a
  brief zombie process entry.
- If `MCPClient.__init__` raises during `_handshake()` (e.g. a hung
  container), the already-spawned `docker run` subprocess is never cleaned
  up, since the fixture only calls `close()` after a successful `yield`.
- The `mcp_client` test-file parameter is typed `Any` rather than the real
  `MCPClient` class (a cleaner import-based typing was available but not used).
- The scenario checks JSON-RPC-level failure only, not the MCP tool-level
  `isError` flag inside a successful envelope.
