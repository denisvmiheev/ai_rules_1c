---
description: Per-task MCP tool playbooks (writing code, review, refactoring — including the safe-refactoring method and mandatory pre-refactor impact analysis, error fixing, performance, forms, integrations, documentation)
alwaysApply: false
category: tooling
---

# Tool Usage by Task — Playbooks

The MCP server catalog, fallback order (`edt-mcp` model-based search/navigation → `search_in_code` literal/regex retry → native `Grep` for project-source search), and per-server tool descriptors live in the `mcp-1c-tools` skill (`content/skills/mcp-1c-tools/SKILL.md`, `docs/<server>.md`). `AGENTS.md` only defines the short obligation rules and points here.

> A project running the classic docker distribution instead of `edt-mcp` substitutes the corresponding tools per `content/skills/mcp-1c-tools/docs/edt-mcp.md`'s remap table (e.g. `search_in_code` → `codesearch`/`search_code`, `get_metadata_details` → `get_object_dossier`, `find_references`/`get_method_call_hierarchy` → `trace_impact`/`trace_call_chain`/`graph_dependencies`); the sequencing and evidence discipline below is unchanged.

## Minimum Evidence Matrix

Use the smallest set that closes the real context gaps. Do not promote a task to a heavier path just to satisfy a generic checklist.

| Task shape | Required before edit | Required after edit |
|---|---|---|
| **Quick-fix BSL** (one logical change in one module, no metadata / transaction / public-contract impact) | Read the target module / procedure and any directly referenced helper needed to understand the bug | `revalidate_objects/validate_query` → `get_project_errors` → `get_check_description/get_problem_summary` on the touched module; quick-fix reduces process overhead, not verification depth |
| **Full-cycle BSL** | `templatesearch` when a reusable pattern may exist; `search_in_code` for local patterns; `get_metadata_details` / `get_metadata_objects` when metadata shape affects the code; platform / БСП / ITS docs only when versioned API or standard behaviour matters; **platform-capability check** (`docsearch` → `docinfo`, + `ssl_search`) whenever the task enters a specialized domain — cryptography, СЛАУ / numerical methods, data analysis, collaboration system / bots, integration bus / queues, full-text search, regex (`AGENTS.md → MCP Tool Calling → A.7`) | `revalidate_objects/validate_query` → `get_project_errors` → `get_check_description/get_problem_summary`; impact analysis when public surface or metadata usage changed |
| **Metadata XML / forms** | Similar object/form examples, metadata lookup, `revalidate_objects`; **mutations go through the `1c-metadata-manage` skill** — hard gate, exceptions only per `SKILL.md → Hard rule` | `revalidate_objects`; metadata validation / form compilation where applicable |
| **Integrations / platform APIs** | Existing integrations, templates, relevant БСП APIs, platform docs for exact API names / version availability, security requirements | `revalidate_objects/validate_query` → `get_project_errors` → `get_check_description/get_problem_summary`; `v8std_search`/`v8std_explain_diagnostics` on `v8std-local` when relying on an ITS standard |
| **Markdown / rules / docs** | Read affected docs and referenced files needed for consistency | Structural checks only: paths, links, anchors, duplicate / conflicting wording |

## Writing New Code

Load `content/rules/coding-standards.md` first; for forms use `forms.md`, for non-trivial queries use `query-design.md`.

0. **Platform-capability check** — mandatory when the task requires a specialized capability (cryptography / digital signatures, СЛАУ / numerical methods, data analysis / ML, collaboration system / bots, integration bus / message queues, full-text search, regular expressions, and similar): `docsearch` by capability description → `docinfo` for exact names found, plus `ssl_search` where a БСП solution is plausible — **before** designing a custom implementation. **If found and usable — build on the platform / БСП mechanism** (`AGENTS.md → A.7`, `1c-platform-mcp.md → Using a found platform mechanism`); do not hand-roll a parallel equivalent.
1. **recall** (`1c-templates-fastcode`) — project-memory lookup with the task's key terms (object name, subsystem, error text) at the start of any non-trivial task, per `AGENTS.md → Project memory`. Skip for genuinely greenfield topics the project has never touched.
2. **templatesearch** — **`templatesearch` only** (`AGENTS.md → A.8`): pre-flight in `1c-templates-fastcode.md → Query formulation (templatesearch only)`; pass user's task verbatim; keyword salad is a defect. **If a matching template is returned — use it as the base** (`AGENTS.md → A.9`, `1c-templates-fastcode.md → Using a found template`); adapt only what the task requires; do not rewrite from scratch.
3. **get_metadata_details** — full passport of the target metadata object (structure, forms, dependencies, code, roles) in a single call.
4. **search_in_code** — review existing patterns in the configuration.
5. **read_method_source** — read an existing procedure/function for reuse.
6. **get_module_structure** — overview of the module you intend to edit.
7. **get_metadata_objects** / **get_metadata_details** — verify metadata structure and attribute types.
8. **get_content_assist** — discover available methods/properties of a context.
9. **docinfo** — verify built-in functions by exact name; **docsearch** — search by description.
10. **ssl_search** — find reusable БСП functions.
11. **revalidate_objects/validate_query** — refresh markers for the touched objects; `write_module_source` already syntax-checked the write.
12. **get_project_errors** — read the EDT findings (severity `BLOCKER`/`CRITICAL`/`MAJOR` first); `get_check_description` explains a check id.
13. **get_problem_summary** — confirming totals pass after the fixes.
14. **validate_query** (`edt-mcp`) — when the change introduces a new / non-trivial query string (module code, DCS data set, dynamic list), validate it offline before delivery. If the session also exposes AI-authoring MCP tools, treat their output as a draft and re-validate the same as hand-written code.

## Code Review

1. **search_in_code** — verify pattern compliance.
2. **find_references** — object-level impact analysis of the change.
3. **get_method_call_hierarchy** — routine-level BSL call chains, callers/callees.
4. **get_metadata_objects** / **get_metadata_details** — correct metadata usage.
5. **docinfo** — verify method/property existence; **docsearch** — search by description.
6. **revalidate_objects/validate_query** — reject syntax-broken input before reviewing.
7. **get_project_errors** (+ **get_check_description**) — bugs, performance and standards findings.
8. **get_problem_summary** — severity totals; style compliance and marker-level findings; pair a finding's check id with **v8std_explain_diagnostics** (`v8std-local`) for the ITS standard behind it (see `docs/v8std.md`).

## Architecture Design

0. **Platform-capability check** — when the designed solution involves a specialized domain (cryptography, СЛАУ / numerical methods, data analysis, collaboration system / bots, integration bus / queues, full-text search, regex), verify via `docsearch` → `docinfo` (+ `ssl_search`) whether the platform / БСП already provides the mechanism before designing a custom one — `AGENTS.md → MCP Tool Calling → A.7`.
1. **get_metadata_details** — passport of key metadata objects.
2. **get_metadata_objects** / **get_metadata_details** — existing metadata structure.
3. **find_references** — usages / reference map for the objects involved (single/multi-hop through EDT's own index — no recursive graph-database traversal, see `docs/edt-mcp.md → Coverage gaps`).
4. **search_in_code** — existing architectural patterns.
5. **get_method_call_hierarchy** — code coupling and call chains.
6. **templatesearch** — architectural templates.
7. **Coverage gap** — `edt-mcp` has no free-form AI-Q&A tool and no configuration-specific documentation search; rely on `docsearch`/`docinfo`/`ssl_search` plus reading the configuration directly.

## Error Fixing

1. **recall** (`1c-templates-fastcode`) — check project memory for the error text / object name first: recurring errors and their fixes are stored there per `AGENTS.md → Project memory`.
2. **get_event_log** (`edt-mcp`) — pull the failing event from the infobase event log offline (no running session): filter by period, user, severity, event / comment / metadata substring. Fall back to the Configurator's `ОтчетПоЖурналуРегистрации` / `ВыгрузитьЖурналРегистрации` only when the log files are unreachable.
3. **revalidate_objects/validate_query** — syntax errors.
4. **get_project_errors** — logic and performance issues.
5. **get_module_structure** / **read_method_source** — locate and read the failing procedure/function.
6. **search_in_code** — related patterns.
7. **read_module_source** — module context around the error.
8. **get_method_call_hierarchy** — how the error propagates through the call chain.
9. **docinfo** — verify function/method names; **docsearch** — fallback by description.
10. **get_metadata_objects** / **get_metadata_details** — verify metadata names and attributes.
11. **validate_query** (`edt-mcp`) — when the suspect path is a query string, validate it offline before deeper investigation.
12. **get_applications** + **evaluate_expression** (`edt-mcp`, debug/Designer session — heavier-weight than a lightweight live-data check, see `docs/edt-mcp.md → Coverage gaps`) — read-only query or BSL-fragment check against the live IB to confirm a data-state or platform-version-specific hypothesis, only when a session is already open and static checks cannot answer the question. Default to read-only; **never** evaluate a mutation without explicit user consent.

## Performance Optimization

0. **Query tuning?** If the slow artifact is (or contains) a query — load `query-design.md` (router) and `content/skills/1c-metadata-manage/docs/query-optimization.md`, and walk its *Mandatory Optimization Checklist* item by item (temp-table indexing, redundant `РАЗЛИЧНЫЕ`, correlated subqueries, virtual-table parameters / periodicity, join-before-grouping). This applies to every «оптимизируй запрос» task even when no MCP server is exposed.
1. **search_in_code** — locate slow patterns ("медленный запрос", "цикл по выборке").
2. **get_method_call_hierarchy** — identify hot call chains.
3. **find_references** — objects that cause cascading issues in the affected code paths.
4. **get_metadata_objects** / **get_metadata_details** — verify indexes and metadata structure.
5. **get_project_errors** — bottleneck analysis; re-validate the final state with
   `revalidate_objects/validate_query` → `get_project_errors` → `get_check_description/get_problem_summary`.
6. **templatesearch** — optimized templates.
7. **v8std_search** (`v8std-local`) — ITS performance-standard clauses by topic ("оптимизация запроса", "индексация временных таблиц"); see `docs/v8std.md`.
8. **validate_query** (`edt-mcp`) — validate the rewritten query offline; if a live-data comparison (row counts, Cartesian explosions, virtual-table state) is genuinely needed, use `debug_launch` + `evaluate_expression` on a test or copy IB only (heavier-weight debug-session tool, not a lightweight live-data service — see `docs/edt-mcp.md → Coverage gaps`).

## Refactoring

**Method — sequencing before tools.** Refactoring is high-risk because the user-visible behaviour must stay identical while the code shape changes:

1. **Top-down analysis first.** Map the entire chain of calls and data flow for the area you intend to change. Do not start editing until you can answer: what are the entry points, who calls what, what registers / metadata are touched, what is the observable behaviour.
2. **Bottom-up edits.** Start with the lowest-level utility functions and work upward. Higher-level callers integrate the refactored helpers only after the helpers themselves are clean and verified.
3. **No "while we're here" edits.** Out-of-scope cleanup belongs to a separate, explicit task — see `AGENTS.md → Surgical Changes`.

**Pre-refactor impact analysis (steps 1–4 below) is mandatory** — run it before touching the first line. If the impact-analysis MCPs are not exposed in the session, follow the graceful-degradation procedure from `verification-gates.md → Gate 4`; do not refactor blind.

1. **get_metadata_details** — passport of the object being refactored.
2. **find_references** — what breaks on change; every type reference before renaming/removing.
3. **get_method_call_hierarchy** — all callers.
4. **search_in_code** — every code pattern related to the object.
5. **search_in_code** (re-run after the edit) — post-refactor verification that no old references remain.
7. Run the full closing gate from `verification-checklist.md` once. It applies
   `revalidate_objects/validate_query` → `get_project_errors` → `get_check_description/get_problem_summary` in order and reuses fresh Stage 3
   evidence instead of repeating validators on unchanged code.

If the refactor is large enough to enter the subagent pipeline — delegate per
`subagent-pipeline.md → Stage 3` (`1c-refactoring`).

## Generating / Modifying Metadata XML

**Step 0 is the execution decision, not a formality.** Load the **`1c-metadata-manage`** skill (`SKILL.md` → domain doc, e.g. `meta-manage.md`) **before** writing or modifying any XML — the mutation itself is driven by the skill's tools (hard gate: `AGENTS.md → Skills and Subagents`, exceptions in `SKILL.md → Hard rule`). MCP calls below gather evidence around the skill run, they do not replace it.

1. **1c-metadata-manage skill** — read `SKILL.md`, dispatch to the domain doc; decide direct execution vs. `1c-metadata-manager` subagent per its Dispatch Strategy.
2. **get_metadata_objects** (`names_only=true`) — similar objects as examples.
3. **revalidate_objects** — XSD schema for the target metadata type.
4. Execute the mutation via the skill's tools (scaffold / edit / compile scripts) against the schema and examples.
5. **revalidate_objects** + the skill's validation scripts — validate; fix errors and re-validate.

## Form Analysis and Generation

**Step 0 — same hard gate as above.** Creating or structurally modifying `Form.xml` / layouts goes through the **`1c-metadata-manage`** skill (`form-manage.md`, form-compile DSL) or the `1c-metadata-manager` subagent; load `forms.md` (router) for the design rules. Hand-writing `Form.xml` while the skill is available is a defect (`SKILL.md → Hard rule`).

1. **1c-metadata-manage skill** — `SKILL.md` → `form-manage.md` (+ `form-compile-dsl.md` for generation); direct execution vs. subagent per Dispatch Strategy.
2. **get_objects_by_tags/get_tags** — similar existing forms in the configuration.
3. **get_form_layout_snapshot** — structure of the found form (elements, bindings, commands, events).
4. **get_metadata_objects** (`names_only=true`) — metadata objects for XML references.
5. **revalidate_objects** (`"Форма"`) — XSD schema of `Form.xml`.
6. Generate / modify the form via the skill's tools (form-scaffold / form-edit / form-compile), based on the examples and schema.
7. **revalidate_objects** + form validation (`form-validate`) — validate; fix errors and re-validate.

## Integrations

Use this playbook when writing HTTP services / clients, REST integrations, file or message-queue exchanges, webhooks. Domain rules — `integrations-add.md`.

1. **ssl_search** — check for ready-made БСП subsystems ("Интернет-поддержка пользователей", "Обмен данными", "Получение файлов из Интернета", "Цифровая подпись").
2. **templatesearch** — integration templates (HTTP request, JSON parsing, signed payloads, retry policy).
3. **search_in_code** (`semantic` mode) — existing integrations in the configuration ("HTTP запрос", "отправка JSON", "парсинг ответа").
4. **docinfo** — verify platform types by exact name (`HTTPСоединение`, `HTTPЗапрос`, `ЧтениеJSON`, `ЗаписьJSON`, `ЗаписьXML`, `ЧтениеXML`).
5. **docsearch** — fallback when the exact platform-API name is unknown.
6. **revalidate_objects** — when the contract is XML with a known schema.
7. **v8std_search** (`v8std-local`) — ITS articles on long-running operations, secure password storage, asynchronous external components; see `docs/v8std.md`.
8. **list_modules** + **get_module_structure** — locate or extend the integration common module (typically `*HTTPClient`, `*Integration`, `*Exchange`).
9. After implementation: **revalidate_objects/validate_query** → **get_project_errors** → **get_check_description/get_problem_summary**.

## Documentation

1. **search_in_code** — find code to document.
2. **get_metadata_objects** / **get_metadata_details** — metadata structure.
3. **get_module_structure** — list of procedures/functions.
4. **docinfo** — documentation by exact name; **docsearch** — search by description.
5. **get_platform_documentation** — existing help articles.
6. **v8std_search** (`v8std-local`) — ITS methodology articles by topic; see `docs/v8std.md`.
7. **Coverage gap** — no version-specific / cross-version platform-documentation-diff tool. For comparing platform versions, use `docinfo`/`docsearch` against the project's actual `PLATFORM_VERSION` and read the platform release notes directly when a cross-version comparison is genuinely needed.
