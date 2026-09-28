---
name: 1c-live-ib
description: "Answer what only the running infobase knows — what the event log recorded, does a query parse, what an expression returns on real data — via edt-mcp (offline event log; debug session for live evaluation). Read-only by default; any mutation needs explicit user consent and a copy infobase."
argument-hint: "<lasterror | query text | expression>"
allowed-tools: mcp__edt-mcp__list_projects, mcp__edt-mcp__get_event_log, mcp__edt-mcp__validate_query, mcp__edt-mcp__get_applications, mcp__edt-mcp__list_configurations, mcp__edt-mcp__debug_launch, mcp__edt-mcp__debug_status, mcp__edt-mcp__evaluate_expression, mcp__edt-mcp__get_variables, mcp__edt-mcp__terminate_launch
---

# 1c-live-ib — the running infobase

Escalate here only when the configuration, the static validators and the docs cannot answer: "does it run *now* in *this* IB". Never a replacement for `1c-validate`. Available only when the tools are exposed in the session; fake "execution output" is a defect.

## Tools

| Need | Call | Notes |
|---|---|---|
| Last / recent error of the infobase | `get_event_log` | Reads `1Cv8.lgf` + `*.lgp` **offline** — no running session; filter by period, user, severity, event / comment / metadata substring, session. Returns infobase PII: quote only what the diagnosis needs |
| Does a query parse and resolve | `validate_query` | Offline, against the project's metadata; `dcsMode` for СКД — run it before any live evaluation |
| Which infobase / launch target | `get_applications`, `list_configurations` | The `applicationId` source; confirm it is a dev / test base |
| What an expression / query returns on real data | `debug_launch` → `wait_for_break` → `evaluate_expression` (+ `get_variables`) → `terminate_launch` | Needs an active debug session: stateful and heavy — use only when static checks cannot answer |

## Safety

- Read-only first: `get_event_log` and `validate_query` need no session; `evaluate_expression` only for non-mutating expressions.
- No `Записать()`, `Удалить()`, `НачалоТранзакции`, register movements without asking the user, naming the object and having a rollback plan; only on a copy infobase — on a production IB refuse and request a copy.
- No secrets or personal data in expressions or queries; they may be logged.
- Debug and `update_database` act on a real application: confirm the target is a dev / test base first, exactly as the deployment commands require.
- Gate 3a (`content/rules/verification-gates.md`) uses only the read-only tools.

Where it sits in the playbooks: error fixing (event log first), performance (profiling), new code (`validate_query`) — `content/rules/tooling-playbooks.md`. Tool catalog: `content/skills/mcp-1c-tools/docs/edt-mcp.md`.
