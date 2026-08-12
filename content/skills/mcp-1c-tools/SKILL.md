---
name: mcp-1c-tools
description: "Catalog of MCP servers for 1C development — search, code navigation, metadata, code review, docs, ITS, templates. Use whenever a 1C task requires calling tools from `edt-mcp`, `1c-ssl-mcp`, `1c-platform-mcp`, `1c-templates-fastcode`, or `v8std-local`. Each server has its own detail file under `docs/` — load it when you are about to call tools from that server, and only if the server is actually available in the current session."
---

# MCP tools for 1C — dispatcher

This skill is the single source of truth for the project's MCP server catalog, task→tool mapping, fallback order, and project-index search retries. Detailed per-tool descriptions for each server live in separate files under `docs/`. **Load a specific `docs/<server>.md` when you are about to call tools from that server and want to tune parameters; the server must be actually available in the current session** (its tools are exposed in the tool schema; the mere presence of an entry in `mcp-servers.json` does not count as availability).

## What is mandatory vs. conditional

- **Mandatory for risk-bearing 1C work.** If a relevant server is exposed, call the fitting MCP tool for BSL / metadata edits or review, metadata XML, forms, integrations, refactoring, performance, runtime errors, platform API checks, impact analysis, syntax / quality validation, and project-memory operations.
- **Conditional for external knowledge.** Use platform docs, БСП / SSL, and ITS MCP tools when the task depends on versioned platform behavior, reusable БСП APIs, or standards compliance. Do not call them for generic prose cleanup or rule-file editing unless such a fact is actually needed.
- **Not required for Markdown / rules / documentation-only work.** For rule files, README, commands documentation, and similar prose-only edits, validate structure, links, paths, and internal consistency instead of calling 1C project MCP tools.
- **Mandatory before parameter-rich calls.** Read `docs/<server>.md` before the first call in the session to every parameter-rich tool listed below, and re-read it when switching tools on that server. A genuinely simple one-shot lookup with obvious arguments may skip the detail file only when it is not in the parameter-rich list.

### Parameter-rich tools — read the live schema first

`edt-mcp`'s search / navigation / analysis tools (e.g. `search_in_code`, `get_metadata_objects`, `get_metadata_details`, `get_method_call_hierarchy`, `find_references`, `get_project_errors`) usually accept several optional filters, and defaults are not always the best starting point. This fork intentionally does **not** maintain a hard-coded parameter cheat-sheet for `edt-mcp` — its surface is large and evolves with the plugin, and Claude Code exposes each tool's real JSON schema at call time. Consult that schema before a parameter-rich call, and re-confirm when switching tools. `edt-mcp` also ships **`get_tool_guide('<toolName>')`** — the extended per-tool documentation deliberately kept out of `tools/list`; call it when the schema alone is not enough. Conceptual "when to use which tool" guidance — `docs/edt-mcp.md`.

Nearly every `edt-mcp` tool requires **`projectName`** — confirm it with `list_projects`. Module paths are `src/`-relative; metadata is addressed by FQN (`Catalog.Products`, `Catalog.Products.Attribute.Weight`, `Role.FullAccess`), with English or Russian type tokens.

If `docs/<server>.md` conflicts with the descriptor exposed by the current environment, the environment descriptor wins.

## When to use this skill

- Before writing code / a query / metadata XML — pick the MCP tool that best fits the task (template search, metadata check, diagnostics, analysis).
- Before implementing a specialized capability (cryptography, СЛАУ / numerical methods, data analysis, collaboration system / bots, integration bus / queues, full-text search, regular expressions, …) — run the platform-capability check: `docsearch` → `docinfo` on `1c-platform-mcp` (+ `ssl_search` where a БСП solution is plausible). Do not trust model memory about what the platform does or does not have. Canon — `AGENTS.md → MCP Tool Calling → A.7`; procedure and trigger domains — [`docs/1c-platform-mcp.md`](docs/1c-platform-mcp.md) → *Platform capability discovery*.
- For impact analysis and code navigation — use `edt-mcp` (see *Fallback chain* below); fall back to `Grep` / `Glob` only after it misses or is not exposed.
- For platform documentation (`docinfo` / `docsearch`) — platform-API facts, versioned behaviour.
- For ITS standards and their linked diagnostics (`v8std_search` / `v8std_explain_diagnostics` / `v8std_explain_snippet` / `v8std_get_page` / `v8std_get_related` on `v8std-local`) — methodology / code-standard facts, and mapping an ACC/АПК/BSLLS/v8-code-style diagnostic to its ITS rationale. See `docs/v8std.md`.
- For code templates and project memory (`templatesearch`, `remember`, `recall`). Before **`templatesearch`**: load `docs/1c-templates-fastcode.md → Query formulation (templatesearch only)` — that rule applies **only** to `templatesearch`, not to other MCP searches.

> Short obligation rules and verification budgets live in `AGENTS.md → MCP Tool Calling` (sections A, B, C). This skill owns the MCP catalog, routing, and fallback details.

## Server catalog

This fork's default runs five MCP servers relevant to 1C development. `edt-mcp` (the MCP plugin running inside 1C:EDT, <https://github.com/DitriXNew/EDT-MCP>) is the primary code/metadata server here, in place of upstream's classic docker distribution (a dedicated metadata/code-search server, a graph-based (Neo4j/Cypher) metadata server, a BSL syntax-checker server, an AI code-review server, and a live-infobase HTTP data server) — the remap table and the remaining coverage gaps live in `docs/edt-mcp.md`. A project that needs the classic multi-server distribution instead of EDT can restore it from this repository's git history (`content/mcp-servers.json` and `content/skills/mcp-1c-tools/docs/` before this fork's edt-mcp/v8std-local consolidation) or from upstream `comol/ai_rules_1c` directly.

| Server (id) | Purpose | Details |
|---|---|---|
| **edt-mcp** | 1C:EDT MCP plugin: metadata discovery & typed mutation, BSL read/write & navigation, call hierarchy / impact analysis, EDT validation markers, forms rendering, debug & YAXUnit & profiling, infobases / DB update, event log, git branches, translation | [`docs/edt-mcp.md`](docs/edt-mcp.md) |
| **1c-templates-fastcode** | Code template library + project vector memory (`remember` / `recall`) | [`docs/1c-templates-fastcode.md`](docs/1c-templates-fastcode.md) |
| **1c-ssl-mcp** | Standard Subsystems Library (БСП / SSL) search | [`docs/1c-ssl-mcp.md`](docs/1c-ssl-mcp.md) |
| **1c-platform-mcp** | 1C platform documentation (search by description / by exact name) | [`docs/1c-platform-mcp.md`](docs/1c-platform-mcp.md) |
| **v8std-local** | ITS "Стандарты разработки 1С" (v8std.ru mirror) — standards search, diagnostic-to-standard mapping (ACC/АПК, BSLLS, EDT, v8-code-style), snippet classification | [`docs/v8std.md`](docs/v8std.md) |

## Fallback chain (highest priority to lowest)

Use only the applicable branch; stop as soon as the collected evidence is sufficient. Before each call, check that it closes a concrete context gap and is not a duplicate of an earlier call.

### Project-source search before `Grep` / `Glob` / `rg`

Native discovery tools (`Grep` / `rg`, `Glob` / file search by pattern, directory listing, sequential `Read`-scanning) substitute only the project-indexing layer — locating files by mask (`**/*.bsl`, `**/*.xml`) or bulk-reading modules "to get oriented" is the same fallback as `Grep`, not a separate free action. Before falling back to any of them for 1C project-source search, exhaust:

1. `edt-mcp` — metadata / code search and navigation as appropriate: `get_metadata_objects`, `get_metadata_details`, `search_in_code`, `get_module_structure`, `read_method_source`, `get_method_call_hierarchy`, `find_references`, `get_content_assist`, etc. — see `docs/edt-mcp.md` for the full tool groups.
2. `search_in_code` as a literal / regex retry (`isRegex`, `fileMask`, `metadataType`, `outputMode=files`) — **only after** the model-based tools above did not find enough. Typical scenarios: exact identifier, fragment of a query text, event-handler name, error message, literal string. Note that `search_in_code` is purely textual and **not** ru/en dialect-aware — a symbol spelled the other way will not match; for identifiers use `find_references` / `go_to_definition` / `get_method_call_hierarchy` instead.
3. Only then a native discovery tool (`Grep` / `rg` / `Glob` / `Read`-scanning) — with a mandatory short note in the response listing which project-index MCP attempts were tried and why they did not return what was needed.

The chain is a **bounded priority, not a prohibition**: when `edt-mcp` is not exposed, the chain collapses and native tools apply immediately (one-line note). When a tuned call plus the documented retry missed — fall back without spending further MCP calls on this rule. After fresh local edits the index may be stale — reading the disk state directly is legitimate. For understanding a single routine prefer fragment-level retrieval (`get_module_structure`, `read_method_source`) over a full-module `Read` (`read_module_source` when the whole module is genuinely needed); reading a direct edit target or an MCP-located file is normal work. Boundary cases — `content/rules/mcp-first-search.md`.

### External knowledge

These servers have no `Grep` / `rg` equivalent; call them only when their knowledge is needed:

1. `1c-templates-fastcode` — code templates and project memory (`templatesearch`, `remember`, `recall`). **`templatesearch` only:** query pre-flight (`A.8`) + **reuse found template** (`A.9`, `docs/1c-templates-fastcode.md → Using a found template`).
2. `1c-ssl-mcp` — БСП / SSL reusable APIs and patterns.
3. `1c-platform-mcp` — versioned platform documentation; also the mandatory platform-capability check before hand-rolling a specialized mechanism (see `docs/1c-platform-mcp.md → Platform capability discovery`).
4. `v8std-local` — ITS standards search and standard↔diagnostic mapping (`v8std_search` / `v8std_explain_diagnostics` / `v8std_explain_snippet` / `v8std_get_page` / `v8std_get_related`); see `docs/v8std.md`.
5. `edt-mcp` — diagnostics-based analysis (`revalidate_objects`/`get_project_errors`/`get_check_description`) in place of a prose AI code review, which this fork's default set does not have (see `docs/edt-mcp.md → Coverage gaps`).

## Quick map: "task → MCP tool"

| Task | Tool (`edt-mcp` unless noted) |
|---|---|
| BSL code search | `search_in_code` |
| Metadata object structure | `get_metadata_details` |
| Impact analysis before refactoring | `find_references` + `get_method_call_hierarchy` (single/multi-hop; no recursive graph engine — see `docs/edt-mcp.md → Coverage gaps`) |
| Call graph | `get_method_call_hierarchy` |
| Metadata search by name / structure | `get_metadata_objects` |
| Object usage search | `find_references` |
| Description / synonym / comment search | `get_metadata_objects` (fall back to `search_in_code` if the index misses) |
| BSL syntax / diagnostics after an edit | `revalidate_objects` / `validate_query` → `get_project_errors` → `get_check_description` / `get_problem_summary` |
| Read / write BSL | `read_module_source` / `read_method_source` → `write_module_source` (pre-write syntax check + `expectedHash` guard) |
| Explain a validation finding | `get_check_description` (symbolic id or the short UID from `get_project_errors`) |
| Create / edit / delete metadata, forms, roles, DCS, MXL | `create_metadata` / `modify_metadata` / `delete_metadata` / `rename_metadata_object` / `adopt_metadata_object` — through the `1c-metadata-manage` hard gate |
| Form structure / rendered layout / screenshot | `get_metadata_details` (form FQN) / `get_form_layout_snapshot` / `get_form_screenshot` |
| Runtime error from the infobase | `get_event_log` (offline log-file read, no running session) |
| Push configuration into the infobase | `list_configurations` / `get_applications` → `update_database` (preview → `confirm=true`) — via `/update1cbase` |
| Platform documentation | `docsearch` / `docinfo` on `1c-platform-mcp` (or `get_platform_documentation` on `edt-mcp`, pinned to the project's platform version) |
| БСП / SSL reuse | `ssl_search` on `1c-ssl-mcp` |
| Code templates | `templatesearch` on `1c-templates-fastcode` |
| ITS standard behind a diagnostic (ACC/АПК, BSLLS, EDT, v8-code-style) | `v8std_explain_diagnostics` on `v8std-local` |
| ITS standard by topic / prose | `v8std_search` on `v8std-local` |
| Full text of a found standard/diagnostic | `v8std_get_page` on `v8std-local` |

Step-by-step playbooks per task type (writing code, review, architecture, error fixing, performance, refactoring, metadata XML, forms, integrations, documentation, comparing platform versions) — `content/rules/tooling-playbooks.md`.
