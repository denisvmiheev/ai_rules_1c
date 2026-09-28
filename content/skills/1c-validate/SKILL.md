---
name: 1c-validate
description: "Validate changed 1C artifacts — BSL and metadata by EDT markers, query text against the project's metadata, the ITS clause behind a finding — within the verification-policy budget. Use after every BSL or metadata edit."
argument-hint: "<module path | object FQN> [severity]"
allowed-tools: mcp__edt-mcp__list_projects, mcp__edt-mcp__write_module_source, mcp__edt-mcp__revalidate_objects, mcp__edt-mcp__validate_query, mcp__edt-mcp__get_project_errors, mcp__edt-mcp__get_check_description, mcp__edt-mcp__get_problem_summary, mcp__edt-mcp__resync_to_disk, mcp__edt-mcp__apply_quick_fix, mcp__edt-mcp__validate_form_model, mcp__v8std-local__v8std_explain_diagnostics, mcp__v8std-local__v8std_get_page
---

# 1c-validate — the validator chain

Order for touched BSL on `edt-mcp`: refresh the markers (`revalidate_objects`, plus `validate_query` for new query text) → read the findings (`get_project_errors`) → explain and confirm (`get_check_description`, `get_problem_summary`), at the depth of `VERIFICATION_DEPTH` and within the budget of `content/rules/verification-policy.md → Validator budget` (one clean pass on the latest state; after a blocking fix one confirmation at `standard`, up to two at `full`; never a run on unchanged content). Metadata without BSL: `revalidate_objects` + `get_project_errors` scoped to the changed FQNs, once. Which gates a change needs — `content/rules/verification-gates.md`.

## Tools

Every call takes `projectName` (`list_projects`); take exact parameter names from the live schema (`get_tool_guide('<toolName>')`).

| Gate | Call | Notes |
|---|---|---|
| 1 Syntax | `write_module_source` pre-write check → `revalidate_objects` (`objects=[FQNs]`) → `get_project_errors` (`severity=ERRORS`) | `revalidate_objects` re-reads the disk first, so it also covers files edited outside EDT; a syntactically valid module can still break references |
| 1 Query text | `validate_query` (`dcsMode=true` for СКД) | Parse and semantic check against the project's metadata, no infobase |
| 2 Logic & performance | `get_project_errors` (`severity` BLOCKER / CRITICAL / MAJOR, `objects=[FQNs]`) + `get_check_description` | `ql-*` query checks, `begin-transaction`, standards checks; read what a check means before fixing or dismissing it |
| 3 Style | `get_project_errors` (MINOR / TRIVIAL) → `get_problem_summary` | Confirming totals pass; nothing worse than the pre-change baseline |
| 5 Metadata | `revalidate_objects` → `get_project_errors` on the changed FQNs | There is no XSD / raw-XML validator; EDT validation of the loaded model is the check |
| 5 Form structure | `validate_form_model` (one form FQN) | Computed from the model **now** (unlike `get_project_errors`): broken data paths, buttons without commands, handlers without procedures, duplicate ids |
| Auto-fix | `apply_quick_fix` | Applies EDT's official fix to **one** marker; a code change like any other — re-validate afterwards |
| ITS rationale for a check id | `v8std_explain_diagnostics` (+ `v8std_get_page`) on `v8std-local` | When `get_check_description` alone is unclear or a review needs the ITS citation |

## Reading the answers

- Blocking: any `ERRORS` / `BLOCKER` / `CRITICAL` finding, a syntax rejection from `write_module_source` / `validate_query`, and a logic, data-integrity, security, transaction or performance defect named by a finding. Style noise (`MINOR` / `TRIVIAL`) never starts another pass.
- `md-reference-intergrity` "lost reference" findings are a desync symptom: `resync_to_disk`, report first.
- `get_check_description` needs the check-descriptions folder configured in EDT's MCP preferences; a configuration error → `get_server_status`, keep the finding without the explanation.
- No AI per-method analysis and no prose review tool: on risk-bearing changes compensate with the internal review checklist (`content/skills/mcp-1c-tools/docs/edt-mcp.md → Coverage gaps`).
- A missing validator is graceful degradation with a `Risk:` line, never a silent skip. Typed answers map to actions by code — `content/rules/mcp-policy.md → C. Server answers → actions`.

Tool catalog: `content/skills/mcp-1c-tools/docs/edt-mcp.md`.
