# Secure Endpoint MCP Server — Dependency Upgrade Design

**Date:** 2026-09-03
**Source:** [ABS-299574](https://myabsolute.atlassian.net/browse/ABS-299574) — "Update dependencies of absolute public api mcp server"
**Scope:** Bump `fastmcp`/`mcp` and all transitive dependencies to latest, eliminate known CVEs, and add guardrails so this doesn't silently drift again.

## 1. Background

The repo has been running `fastmcp==2.12.4` / `mcp==1.15.0` for roughly a year. `pyproject.toml`
declares every dependency as a bare name with no version constraint at all, so the lockfile
drifted two `fastmcp` majors (2.x → 4.x) and one `mcp` major (1.x → 2.x) behind without any PR
ever surfacing the jump. The ticket was raised because:

- Professional services has a customer building a Claude integration against this server, so
  staying current and CVE-clean is now externally visible, not just internal hygiene.
- `pip-audit` against the currently installed environment found **8 known vulnerabilities across
  4 transitive packages**: `idna` (1 CVE, fix `3.15`), `pygments` (1 CVE, fix `2.20.0`), `requests`
  (1 CVE, fix `2.33.0`), `urllib3` (4 CVEs, fixes `2.6.0`/`2.6.3`/`2.7.0`). None are in direct
  dependencies — all are pulled in transitively — so a resolver bump is expected to clear them
  without any code change.
- The existing UAT remediation spec
  (`docs/spec/2026-08-25-secure-endpoint-mcp-uat-remediation-design.md`) separately flagged three
  defects living inside `fastmcp` itself (dangling `$ref` resolution, dropped array-body schemas,
  binary-response mis-decoding) and recommended checking whether a newer `fastmcp` release already
  fixes them. **That verification is explicitly out of scope for this ticket** (see §2) — it is
  tracked as its own follow-up, even though this upgrade is a prerequisite for ever running that
  check.

## 2. Goals / Non-goals

**Goals:**

- Bump `fastmcp` `2.12.4` → `4.0.2` and `mcp` `1.15.0` → `2.1.1` in a single jump (no staged
  2.x→3.x→4.x bisection), fixing whatever breaks along the way.
- Resolve all transitive dependencies to latest via `uv lock --upgrade` and confirm the 8
  `pip-audit` findings are cleared.
- Add version bounds to every direct dependency in `pyproject.toml` (currently unconstrained bare
  names) and add GitHub Dependabot for the Python ecosystem, so future upgrades arrive as small,
  reviewed PRs instead of another silent multi-major drift.

**Non-goals:**

- No UAT remediation work. Group A (`_prepare_jws_payload` bodyless-write fix), Group B (fastmcp
  internal `$ref`/array-body/binary-response defects), and every other item in the 2026-08-25 UAT
  design doc stay out of scope. If this upgrade happens to fix a Group B defect as a side effect,
  note it in the PR description for the record — do not scope-creep into verifying or fixing it
  here.
- No new tools, features, or behavior changes beyond what's required to keep the server working
  under the new dependency versions.
- No CI workflow redesign beyond adding the Dependabot config itself.

## 3. Key risk: `httpx` vs `httpx2` in the auth path

`mcp` 2.x's release notes state the SDK's own HTTP stack moved from `httpx`+`httpx-sse` to a new
`httpx2` package (`>=2.5.0`), with logger names, TLS trust-store behavior, and SSE `Accept` headers
changing accordingly. Release notes hedge on whether this landed in the final `2.0.0` stable or was
reverted before release.

This repo's entire auth mechanism is `AbsoluteAuthClient` (`client/auth_client.py`), a subclass of
`httpx.AsyncClient` whose overridden `request()` is handed to `FastMCPOpenAPI` as the backend HTTP
client used for every call to Absolute's API. That is a different layer from the MCP
client↔server transport `httpx2` note is describing, but the distinction must be confirmed by
reading actual `fastmcp==4.0.2` source against a real install — not inferred from changelog
wording — before any code changes are written. Specifically, confirm:

1. `FastMCPOpenAPI` (or its 4.x equivalent) still accepts a real `httpx.AsyncClient` (or subclass)
   as its backend client parameter.
2. The `mcp_component_fn` hook (`schema_fix.py`'s output-schema-disabling mechanism) and the
   `_route_map_fn` signature (`mcp_server.py`) are unchanged, or have a clear, mechanical migration
   path if renamed/reshaped.
3. Nothing this server actually uses collides with "FastMCP 3's deprecated APIs are gone" or the
   4.0.0 removal of server-initiated sampling/roots — this server doesn't use sampling or
   elicitation, so this is expected to be a non-issue, but must be confirmed rather than assumed.

## 4. Execution plan

1. **Spike/verify** — install `fastmcp==4.0.2` and `mcp==2.1.1` into a scratch venv and read the
   installed source directly for the three confirmations in §3. This determines the actual scope
   of code changes needed in step 3, rather than guessing from release notes.
2. **Add version bounds and re-lock** — change every bare dependency name in `pyproject.toml` to a
   compatible-range constraint (`fastmcp>=4.0,<5`, `mcp>=2.1,<3`, and similar bounds for `httpx`,
   `pydantic`, `pydantic-settings`, `structlog`, `html2text`, `python-dotenv`), then run
   `uv lock --upgrade` to regenerate `uv.lock` against the new bounds.
3. **Fix breaks** — resolve whatever `mypy`, `pytest`, `black`, and `isort` (the existing CI gates)
   surface, guided by step 1's findings. Expect at minimum a review of `schema_fix.py` and the
   `_route_map_fn`/`FastMCPOpenAPI` construction in `mcp_server.py`, since those are this server's
   only points of contact with fastmcp's internal APIs.
4. **Validate:**
   - Full CI suite (tests + mypy + black + isort) passing locally.
   - Live smoke test against a sandbox Absolute account: one read-only tool call and one
     bodyless-write tool call (mirroring the reproduction pattern in the 2026-08-25 UAT doc), to
     confirm `AbsoluteAuthClient` still signs and sends correctly at runtime, not just under the
     mocked-transport unit tests.
   - Manual MCP-client round trip (Claude Desktop or Claude Code) confirming tool discovery and a
     real tool call succeed over the actual stdio/http transport.
5. **Re-run `pip-audit`** against the upgraded environment to confirm all 8 findings from §1 are
   cleared. Note any that persist — that would mean the CVE lives in a dependency this bump didn't
   touch, and needs separate handling.
6. **Add `.github/dependabot.yml`** for the Python ecosystem (`pip`/`uv`), weekly cadence,
   targeting `pyproject.toml`, so future version drift arrives as reviewable PRs instead of
   silently accumulating.

## 5. Testing strategy

The existing suite (`test_auth_client.py`, `test_config.py`, `test_feature_flags.py`,
`test_mcp_server.py`) must keep passing with behavior unchanged; only touch a test's assertions
where the new dependency version forces an actual API signature change. Add new regression
coverage only if step 1's verification uncovers a real behavior change worth locking in — this is
a dependency bump, not a feature, so no speculative new tests. The live smoke test and manual MCP
client test in step 4 are one-time validation gates for this PR, not new permanent automated
tests — there are no sandbox credentials available in CI to run them repeatedly.

## 6. Rollback

Since no code beyond dependency-facing shims should change behavior for callers, rollback is a
straight revert of the PR (`pyproject.toml`, `uv.lock`, and whatever `schema_fix.py`/`mcp_server.py`
changes step 3 required). No data migration or external state is touched by this work.

## 7. Open questions

- Whether `httpx2` actually shipped in `mcp` 2.0.0 stable, and if so, whether it has any indirect
  effect on `FastMCPOpenAPI`'s backend client parameter — resolved by step 1 of the execution plan,
  not before.
- Whether any of the UAT Group B defects are incidentally fixed by the version bump. Not
  investigated as part of this ticket; worth a one-line note in the PR description if observed
  in passing during step 4's smoke testing, for whoever picks up the UAT follow-up ticket.
