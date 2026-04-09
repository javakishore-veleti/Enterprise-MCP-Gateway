# Enterprise-MCP-Gateway

A production-shaped Model Context Protocol gateway exposing internal
enterprise tools to LLM clients under token auth, per-tool scope
enforcement, structured logging and tracing. Includes a FastMCP server,
a reference MCP client, and a Google ADK agent that consumes the
gateway end-to-end.

This README is a direct walkthrough of `Docs/Design/architecture.drawio`.
Each section maps one-to-one to a tab in that file.

## Business domain

Enterprises that have started embedding LLMs into internal workflows
hit the same wall: the LLM is useful only if it can reach **real
business systems** — record stores, case queues, workflow services,
audit databases — and it has to do that under the same auth and
audit rules a human user would. The Model Context Protocol (MCP) is
the emerging convention for that "tools the LLM is allowed to call"
surface, but most reference deployments treat MCP as a developer toy
without auth, scoping, or an audit trail.

This service is the production-shaped version: a single MCP gateway
in front of every internal tool, with token auth, per-tool scope
enforcement, structured observability, and an immutable audit log.

## Business features

| Feature | What the user gets |
|---|---|
| One MCP endpoint, many tools | Internal teams register their tools once with the gateway; every LLM client uses the same endpoint. |
| Token auth | Bearer tokens are verified on every call. No token, no dispatch. |
| Per-tool scope enforcement | Each tool declares its allowed scopes (e.g. `analyst`, `ops`, `supervisor`). Calls outside the allowed set are rejected with `403`. |
| Structured logging | Every dispatch emits a JSON log line carrying `request_id`, `caller_id`, `tool`, `scope`, latency, outcome. |
| Distributed tracing | OpenTelemetry spans per dispatch, parented to the incoming MCP request, so a tool call is traceable across services. |
| Immutable audit log | Append-only `(caller, tool, args_hash, outcome, ts, latency)` rows in a separate collection — what compliance asks for after an incident. |
| Reference client + ADK agent | Two consumers ship with the repo: a thin Python MCP client for ad-hoc use, and a Google ADK agent that picks tools and chains calls. |

## Architecture (mapped to `Docs/Design/architecture.drawio`)

### Tab 1 — System Overview

Five components and the dispatch path through them:

- **Google ADK agent** and **MCP reference client** are the two
  in-repo consumers. Any third-party MCP client also lands here.
- → **Auth + scope middleware** verifies the bearer token and checks
  the caller's scope against the requested tool's allowed scopes.
- → **FastMCP server** holds the tool registry and dispatches the
  call to the right Python function.
- → **Internal enterprise tools** (adapters / SDK calls / DB reads)
  do the real work and return the result.
- The dispatch fans out a row to the **audit log** alongside the
  reply.

### Tab 2 — Auth + Scope Flow

Decision diagram every incoming MCP call runs through:

1. Token present + valid? — no → `401 unauthorized`, audit row with
   `outcome=auth_fail`.
2. Caller in the tool's allowed scopes? — no → `403 forbidden`, audit
   row with `outcome=scope_denied`.
3. Dispatch to the tool function with a structured log + trace span.
4. Audit log append carries `(caller, tool, scope, outcome, ts,
   latency)` — pass *and* fail rows are recorded.

### Tab 3 — Tool Catalog

The illustrative registry the gateway ships with:

| Tool | Allowed scopes | Backed by |
|---|---|---|
| `record_lookup(record_id)` | analyst, ops | record store adapter |
| `recent_activity(record_id, limit)` | analyst, ops | activity feed adapter |
| `case_queue(filter)` | ops | case service adapter |
| `write_disposition(record, action)` | ops, supervisor | workflow service adapter |
| `export_audit(window)` | supervisor, compliance | audit log adapter |

`ToolRegistry` is the seam — each entry carries `name`, `callable`,
`allowed_scopes`, `response_schema`, and a rate limit. Adding a new
tool is one registry call; auth enforcement is automatic.

### Tab 4 — Sequence: ADK agent end-to-end

UML sequence of one full call: ADK agent reasons and picks a tool →
MCP client formats the call → auth middleware verifies token + scope →
FastMCP server dispatches → tool function returns the result → audit
log append → MCP response → parsed result back to the agent. The
agent may then chain a second tool with the same dispatch path.

### Tab 5 — Observability

The three observability surfaces and how each request lights them up:

- **Structured logger** (structlog, JSON output, every line carries
  `request_id`, `caller_id`, `tool`, `scope`).
- **Tracing** (OpenTelemetry spans per dispatch, parent: incoming
  MCP request).
- **Audit log** (append-only Mongo collection, immutable).

The Pytest test surface covers: auth happy path, `401`, `403`, scope
mismatch, unknown tool, and audit append. Every request emits one
log line, one trace span, and one audit row.

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| MCP server | FastMCP | reference Python implementation of MCP, async, easy tool registration |
| HTTP | FastAPI (under FastMCP) | async, Pydantic models, OpenAPI for the admin / health surfaces |
| Auth | bearer token + scope set | minimal viable auth; pluggable for OAuth / JWT / mTLS in production |
| Tool adapters | plain Python callables | each adapter wraps one upstream system; easy to mock in tests |
| Logging | structlog | JSON output, contextvars for `request_id` propagation |
| Tracing | OpenTelemetry | spans on every dispatch; exporters configurable per deployment |
| Audit log | MongoDB collection | append-only; insert-only writes; queried by compliance tools |
| Tests | Pytest | covers auth happy + sad paths, scope rules, audit append, registry behavior |

## Why this exists

This is a portfolio bridge repo authored to plug the most JD-specific
gap (FastMCP, MCP server / client, Google ADK, auth + scoping,
structured logging) by building the production-shaped version of an
MCP gateway end to end. The architecture diagram is the authoritative
source — this README walks through it for readers who want the prose
version.

## Status

Scaffolded. The architecture diagram is committed; tool registry,
auth middleware, and the reference ADK consumer are the next
iterations. See `Docs/Design/architecture.drawio` for the target
architecture.
