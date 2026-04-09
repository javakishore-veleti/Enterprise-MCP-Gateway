# Development Plan — Enterprise-MCP-Gateway

A focused punch list to take this repo from a scaffold to the
architecture in `Docs/Design/architecture.drawio`. Each block builds
out one tab of that diagram so the diagram and the code stay in sync.

Legend: `[ ]` not started · `[~]` in progress · `[x]` done

## Block 1 — Project skeleton

- [ ] **T1.1** `pyproject.toml` with fastmcp, fastapi,
       pydantic-settings, structlog, opentelemetry-api +
       opentelemetry-sdk, pytest, pytest-asyncio, httpx, mongomock
       as the dev extras.
- [ ] **T1.2** `src/enterprise_mcp_gateway/` package layout —
       `auth/`, `registry/`, `tools/`, `server/`, `client/`,
       `agent/`, `audit/`, `config.py`, `logging.py`, `main.py`.
- [ ] **T1.3** `tests/` mirror the same tree.
- [ ] **T1.4** `pytest -q` green on a smoke test.

## Block 2 — Auth + scope (Tab 2)

- [ ] **T2.1** `auth/identity.py` — `CallerIdentity` dataclass,
       `authenticate(token, caller_id)` raising `AuthError` on
       missing / invalid token.
- [ ] **T2.2** `auth/scopes.py` — `VALID_SCOPES`, `require_scope`
       enforcing the per-tool allowed set.
- [ ] **T2.3** `auth/middleware.py` — wraps every dispatch:
       authenticate → scope check → call → audit row (pass and
       fail rows alike).
- [ ] **T2.4** `tests/auth/test_middleware.py` — 401 / 403 / happy
       path / scope mismatch.

## Block 3 — Tool registry (Tab 3)

- [ ] **T3.1** `registry/registry.py` — `ToolRegistry` with
       entries: `(name, callable, allowed_scopes, response_schema,
       ratelimit)`. Loaded at startup.
- [ ] **T3.2** `tools/record_lookup.py`, `tools/recent_activity.py`,
       `tools/case_queue.py`, `tools/write_disposition.py`,
       `tools/export_audit.py` — each wraps a fake adapter so the
       suite stays offline.
- [ ] **T3.3** `tests/registry/test_registry.py` — register +
       lookup + duplicate-name guard + scope read-back.

## Block 4 — FastMCP server (Tab 1)

- [ ] **T4.1** `server/server.py` — `build_mcp_server(registry,
       middleware)` registering each tool through the auth wrapper.
- [ ] **T4.2** `tests/server/test_server.py` — exercise the wire
       path with fastmcp's in-process `Client` (mirrors the
       Member-Event-Stream-Agent pattern).

## Block 5 — Reference MCP client + ADK agent

- [ ] **T5.1** `client/client.py` — thin Python MCP client with
       `call_tool(name, args, token)` helper.
- [ ] **T5.2** `agent/adk_agent.py` — `LlmAgent` (deferred
       google-adk import) wired to the gateway via the reference
       client; instruction tells it which tools exist.
- [ ] **T5.3** `tests/agent/test_adk_agent.py` — fake LLM picks a
       tool, gateway dispatches, audit row lands.

## Block 6 — Audit log (Tabs 2 + 5)

- [ ] **T6.1** `audit/store.py` — append-only Mongo collection,
       `(caller, tool, args_hash, outcome, ts, latency)`.
- [ ] **T6.2** `tests/audit/test_store.py` — happy + auth_fail +
       scope_denied rows all land.

## Block 7 — Observability (Tab 5)

- [ ] **T7.1** `logging.py` — structlog JSON output, contextvars
       carrying `request_id`, `caller_id`, `tool`, `scope`.
- [ ] **T7.2** `tracing.py` — OpenTelemetry spans on every
       dispatch; parent: incoming MCP request.
- [ ] **T7.3** `tests/observability/test_logs_and_spans.py` —
       assert one log line + one span + one audit row per call.

## Block 8 — CI

- [ ] **T8.1** `.github/workflows/ci.yml` — pytest on every push
       and PR to `main` against Python 3.13 with the dev extras.
- [ ] **T8.2** `pytest -q` green locally and in CI.

## Definition of done

1. `pytest -q` green locally and in GitHub Actions.
2. `uvicorn enterprise_mcp_gateway.main:app` starts cleanly.
3. The reference ADK agent picks `record_lookup`, the gateway
   dispatches under the `analyst` scope, and the audit log row
   lands within one second.
4. Calling the same tool with the wrong scope returns 403 and the
   audit row carries `outcome=scope_denied`.
5. Every block lands as its own commit.

## Status

Scaffold + architecture diagram + README walkthrough committed.
Block 1 is the next iteration.
