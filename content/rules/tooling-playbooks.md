---
description: Per-task MCP tool playbooks — code writing, review, architecture, error fixing, performance, safe refactoring with impact analysis, metadata XML, forms, integrations. Load for code / review / debug / refactor / performance / metadata work.
alwaysApply: false
category: tooling
---

# Tool Usage by Task — Playbooks

Server catalog and routing — `content/skills/mcp-1c-tools/SKILL.md`; exact argument names and example calls — the operation skill it names (`1c-code-search`, `1c-meta-info`, `1c-impact`, `1c-form-inspect`, `1c-validate`, `1c-platform-help`, `1c-templates-memory`, `1c-live-ib`); search discipline — `content/rules/mcp-first-search.md`. In EDT projects (`.dev.env` `USE_EDT=true`) the playbooks apply unchanged and `content/rules/edt-workflow.md` adds the source-format check, EDT validation markers, EDT-side DB update and form snapshots.

## Minimum Evidence Matrix

Use the smallest set that closes the real context gaps; do not promote a task to a heavier path to satisfy a checklist.

| Task shape | Required before edit | Required after edit |
|---|---|---|
| **Quick-fix BSL** | The target module / procedure and any directly referenced helper needed to understand the change | The validator chain of the active `VERIFICATION_DEPTH` on the touched module (default `standard`: `revalidate_objects` → `get_project_errors` → `get_check_description` / `get_problem_summary`; `verification-policy.md → Verification depth levels`) |
| **Full-cycle BSL** | Common preamble below; `search_in_code` for local patterns; `get_metadata_details` / `get_metadata_objects` when metadata shape affects the code; platform / БСП / ITS docs only when a versioned API or standard matters | The validator chain; impact analysis when public surface or metadata usage changed (`verification-gates.md → Gate 4`) |
| **Metadata / forms** | Similar object / form examples (`get_objects_by_tags`), metadata lookup; **the mutation goes through the `1c-metadata-manage` skill** (hard gate, `AGENTS.md → Skills and Subagents`; on EDT projects the typed `edt-mcp` API) | `revalidate_objects` → `get_project_errors` on the changed FQNs; a form snapshot when layout is the point |
| **Integrations / platform APIs** | Existing integrations, templates, relevant БСП APIs, platform docs for exact API names and version availability, security requirements | The validator chain; ITS check when relying on an ITS standard |
| **Markdown / rules / docs** | Affected docs and the files they reference | Structural checks only: paths, links, anchors, duplicate / conflicting wording |

## Common preamble — every playbook that writes or changes BSL

0. **Platform-capability check** when the task enters a specialized domain (cryptography, СЛАУ / numerical methods, data analysis, collaboration system / bots, integration bus / queues, full-text search, regex, …) — `AGENTS.md → MCP Tool Calling → A.7`.
1. **`recall`** with the task's key terms — `content/rules/project-memory.md`.
2. **`templatesearch`** — task text verbatim; a hit is the base — `AGENTS.md → MCP Tool Calling → A.8–A.9`.

The steps below assume the preamble is done. Validator steps are `revalidate_objects` (+ `validate_query` for new query text) → `get_project_errors` → `get_check_description` / `get_problem_summary` within `verification-policy.md → Validator budget`.

## Writing New Code

Load `content/rules/coding-standards.md` first; forms — `forms.md`; non-trivial queries — `query-design.md`.

1. `get_metadata_details` — passport of the target object (structure, forms, dependencies, code, roles).
2. `search_in_code` — existing patterns in the configuration; `get_module_structure` / `read_method_source` — the module you will edit and an existing routine to reuse.
3. `get_metadata_objects` / `get_metadata_details` — attribute names and types; `get_content_assist` — members of a context.
4. `docinfo` (exact name) / `docsearch` (by description) — built-in functions; `ssl_search` — reusable БСП functions.
5. Validator chain; then `validate_query` for every new / non-trivial query string, especially after AI generation.

## Code Review

1. `search_in_code` — pattern compliance.
2. `find_references` — object-level impact; `get_method_call_hierarchy` — callers / callees.
3. `get_metadata_objects` / `get_metadata_details` — metadata usage; `docinfo` / `docsearch` — method existence.
4. Validator chain (`revalidate_objects` first — never review syntax-broken code).
5. `v8std_explain_diagnostics` (`v8std-local`) — the ITS standard behind a finding.

## Architecture Design

0. **Platform-capability check** — when the designed solution involves a specialized domain (cryptography, СЛАУ / numerical methods, data analysis, collaboration system / bots, integration bus / queues, full-text search, regex), verify via `docsearch` → `docinfo` (+ `ssl_search`) whether the platform / БСП already provides the mechanism before designing a custom one — `AGENTS.md → MCP Tool Calling → A.7`.
1. `get_metadata_details` — key objects; `get_metadata_objects` — existing structure.
2. `find_references` — dependency map for the objects involved (single hop through EDT's index — no recursive graph traversal, `docs/edt-mcp.md → Coverage gaps`).
3. `search_in_code` — existing architectural patterns; `get_method_call_hierarchy` — coupling.
4. `templatesearch` — architectural templates. Coverage gap: no free-form AI Q&A and no configuration-specific documentation search — rely on `docsearch` / `docinfo` / `ssl_search` plus reading the configuration directly.

## Error Fixing

Method — `standards(name="systematic-debugging")` (`DEBUG_FAST_PATH` for directly evidenced root causes).

1. `recall` — recurring errors and their fixes are stored there.
2. `get_event_log` — exact text, timestamp and affected metadata of the failing event, read offline (no running session); skip when the scenario is not in the connected infobase's log. The Configurator's `ОтчетПоЖурналуРегистрации` is the fallback only when the log files are unreachable.
3. `revalidate_objects` → `get_project_errors` — syntax and logic defects.
4. `get_module_structure` / `read_method_source`, `search_in_code` — locate and understand the failing routine; `get_method_call_hierarchy` — propagation.
5. `docinfo` / `docsearch`, `get_metadata_objects` / `get_metadata_details` — verify names the code relies on.
6. `validate_query` (offline) → `get_applications` + `evaluate_expression` (read-only, debug session — heavier than a lightweight live-data check; never a mutation without explicit consent, `docs/edt-mcp.md → Safety and discipline`) — confirm a data-state or platform-behaviour hypothesis in the live IB, only when static checks cannot answer.

## Performance Optimization

0. If the slow artifact is or contains a query — `query-design.md` and `content/skills/1c-metadata-manage/docs/query-optimization.md → Mandatory Optimization Checklist`, item by item, even when no MCP server is exposed.
1. `search_in_code` (literal fragments of slow patterns) — slow patterns; `get_method_call_hierarchy` — hot chains; `find_references` — cascades; `start_profiling` → `stop_profiling` → `get_profiling_results` — measure instead of guessing.
2. `get_metadata_details` — indexes and structure; establish the baseline result and the measured bottleneck.
3. Reuse the common preamble's `templatesearch` result as the base when it fits; search again only for a newly identified gap. `v8std_search` (`v8std-local`) — applicable ITS performance standards **before** choosing the rewrite. Confirm the candidate preserves row multiplicity, values and other required behaviour.
4. Adapt the matching template; run the validator chain after the edit.
5. `validate_query` → a bounded read-only live comparison (`evaluate_expression`, test or copy IB) — compare results with the baseline, including duplicates and overlapping conditions, then compare performance. Parsing alone proves neither metadata resolution nor result equivalence (`verification-gates.md → Gate 3a`).

## Refactoring

**Method.** Top-down analysis first: map entry points, callers, touched registers / metadata and the observable behaviour — no edit before you can state them. Bottom-up edits: lowest-level helpers first, callers integrate only after the helpers are clean and verified. No "while we're here" edits (`AGENTS.md → Surgical Changes`).

**Pre-refactor impact analysis is mandatory** — steps 1–3 before touching the first line; when the impact MCPs are not exposed follow `verification-gates.md → Gate 4` graceful degradation, never refactor blind.

**Pre-refactor impact analysis (steps 1–3) is mandatory** — run it before touching the first line. If the impact-analysis tools are not exposed in the session, follow the graceful-degradation procedure from `verification-gates.md → Gate 4`; do not refactor blind.

1. `get_metadata_details` — passport of the object.
2. `find_references` — what breaks on change, every type reference before renaming / removing; `get_method_call_hierarchy` — all callers.
3. `search_in_code` — every related code pattern; after the refactor again — no old references remain.
4. The closing gate from `verification-gates.md` once, reusing fresh Stage 3 evidence. A refactor large enough for the subagent pipeline — `subagent-pipeline.md → Stage 3` (`1c-refactoring`).

## Generating / Modifying Metadata XML

Step 0 is the execution decision: load the **`1c-metadata-manage`** skill (`SKILL.md` → domain doc) **before** writing or modifying any metadata — the mutation is driven by the skill's tools (hard gate, `AGENTS.md → Skills and Subagents`; exceptions in `SKILL.md → Hard rule`); on EDT projects that path is the typed `edt-mcp` API (`create_metadata` / `modify_metadata` / `delete_metadata` / `rename_metadata_object` / `adopt_metadata_object`). MCP calls gather evidence around the run, they do not replace it.

1. `get_metadata_objects` / `get_objects_by_tags` — similar objects as examples; `get_metadata_details` (`assignable: true`) — the property schema the mutation can set.
2. The mutation itself; direct execution vs. the `1c-metadata-manager` subagent per the skill's Dispatch Strategy. Destructive operations are two-phase (preview, then `confirm=true`).
3. `revalidate_objects` → `get_project_errors` on the changed FQNs; fix and re-validate.

## Form Analysis and Generation

Same gate: forms are created or changed through the skill (`docs/form-manage.md`; on EDT projects `create_metadata` / `modify_metadata` on form FQNs) or the `1c-metadata-manager` subagent; design rules — `forms.md` (router). Hand-writing a form while the skill is available is a defect.

1. `get_objects_by_tags` — similar existing forms; `get_metadata_details` (form FQN) / `get_form_layout_snapshot` — their structure (elements, bindings, commands, events).
2. `get_metadata_objects` — objects for references.
3. The skill's form operations; `validate_form_model` → `revalidate_objects` + `get_project_errors`; `get_form_layout_snapshot` to confirm the layout; fix and re-validate.

## Integrations

Domain rules — `integrations-add.md`.

1. `ssl_search` — ready-made БСП subsystems («Интернет-поддержка пользователей», «Обмен данными», «Получение файлов из Интернета», «Цифровая подпись»); `templatesearch` — integration templates.
2. `search_in_code` («HTTP запрос», «отправка JSON», «парсинг ответа») — existing integrations; `list_modules` + `get_module_structure` — the integration common module (`*HTTPClient`, `*Integration`, `*Exchange`).
3. `docinfo` — platform types by exact name (`HTTPСоединение`, `HTTPЗапрос`, `ЧтениеJSON`, `ЗаписьJSON`, `ЗаписьXML`, `ЧтениеXML`); `docsearch` when the name is unknown.
4. `v8std_search` (`v8std-local`) — long-running operations, secure password storage, asynchronous external components.
5. Validator chain.

## Documentation

`search_in_code` / `get_module_structure` — code to document; `get_metadata_objects` / `get_metadata_details` — structure; `docinfo` / `docsearch` / `get_platform_documentation` — platform documentation; `v8std_search` (`v8std-local`) — methodological articles.

## Comparing Platform Versions

No dedicated diff tool. `get_platform_documentation` (`edt-mcp`, pinned to the project's platform version) and the `since` / availability fields in `docinfo` / `docsearch` citations answer "is it available in version X"; for a cross-version comparison read the platform release notes.
