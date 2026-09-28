# edt-mcp — tool catalog

MCP server plugin running **inside 1C:EDT** (upstream: <https://github.com/DitriXNew/EDT-MCP>). It exposes the live EDT workspace: projects, metadata model, BSL modules, EDT validation markers, forms rendering, debug sessions, YAXUnit, profiling, infobases and launch configurations. In this ruleset it is the **primary 1C server**: code search, navigation, metadata, validation, mutation and infobase update all go through it. It is installed into EDT itself by `content/commands/install-edt-mcp.md` and exposed over Streamable HTTP on loopback.

> Load this file only if the `edt-mcp` server is actually available in the current session (its tools are visible in the agent tool schema). A client-config entry or a responding `/health` endpoint proves the plugin, not the session. Process rules for EDT projects — `content/rules/edt-workflow.md`; this file is the server-side catalog.

> **Do not invent parameter names.** Claude Code exposes each tool's real JSON schema at call time — read it before any parameter-rich call. For extended per-tool documentation that is deliberately kept out of `tools/list`, call **`get_tool_guide('<toolName>')`**. This ruleset deliberately does not restate parameter lists: the server version on the user's machine is the truth.

This server is **conditional** in the sense that it exists only on a workstation where EDT is installed, running, and holding the project workspace. A closed EDT means the tools disappear, not that the project changed.

## Session basics

- **`projectName` is required by almost every tool.** Confirm it with `list_projects` when in doubt; a workspace entry that is not an EDT project is not a valid target.
- **Module paths are `src/`-relative**, e.g. `CommonModules/MyModule/Module.bsl`, `Documents/SalesOrder/ObjectModule.bsl`.
- **Metadata is addressed by FQN**, e.g. `Catalog.Products`, `Catalog.Products.Attribute.Weight`, `Catalog.Products.Form.ItemForm`, `Role.FullAccess`. English **and** Russian type tokens are accepted (`Справочник.Товары`).
- **Progressive disclosure** (EDT Preferences → MCP Server) can hide toolsets. `list_toolsets` shows the groups and their visibility; `enable_toolset` reveals one, then re-request `tools/list`. A tool that "does not exist" in an EDT session is usually a hidden toolset, not a missing feature.
- **`get_server_status`** is the self-diagnosis snapshot: port, versions, enabled tool counts, whether the check-descriptions folder is configured, and the two form-render JVM flags. Read it before blaming a blank form screenshot or a missing `get_check_description`.

## Toolsets

`core`, `metadata`, `code`, `debug`, `testing`, `profiling`, `forms`, `tags`, `translation`, `project`. Composition follows the installed plugin version — `list_toolsets` is authoritative when this file and the session disagree.

## Core — discovery and reading

| Tool | Purpose | When to use |
|---|---|---|
| **list_projects** | Workspace projects, state, natures | First orientation; confirm the `projectName` to pass everywhere else |
| **list_modules** | BSL modules of a project (path, module type, owner) | Discover module paths before reading / editing; filter by `metadataType`, `objectName`, `nameFilter` |
| **get_module_structure** | Procedures / functions with signatures, regions, `&AtServer`/`&AtClient`, export flag, line numbers | Understand a module before editing (`responseFormat=detailed` for parameters and doc-comments) |
| **read_module_source** | Module source, whole file or line range; returns a `contentHash` revision token | Read the edit target; the hash round-trips into `write_module_source.expectedHash` |
| **search_in_code** | Literal / regex full-text search across BSL modules | Primary code search. **Not** semantic and **not** ru/en dialect-aware — for identifiers prefer `find_references` / `get_symbol_info` / `get_method_call_hierarchy` |
| **get_metadata_objects** | Flat list of configuration objects (Name, Synonym, Type, modules) | Discover what exists; filter by `metadataType` / `nameFilter` |
| **get_metadata_details** | Full properties of one or more objects by FQN; a **Form** FQN renders the form structure, a **Role** FQN renders the rights matrix + RLS | The metadata passport. `assignable: true` returns the property schema `modify_metadata` can set |
| **get_tool_guide** | Extended per-tool documentation on demand | Before a parameter-rich call whose schema alone is not enough |
| **list_toolsets** / **enable_toolset** | Progressive-disclosure management | Only when tools are missing from `tools/list` |
| **get_server_status** / **get_edt_version** | Server / EDT self-diagnosis | Explain blank screenshots, plain-text mode, missing checks folder |

## Code — navigation, authoring, queries

| Tool | Purpose | When to use |
|---|---|---|
| **read_method_source** | One procedure / function body by name | Cheaper than a full-module read when one routine matters |
| **write_module_source** | Write BSL: `searchReplace` (default), `replace`, `append`; **runs a BSL syntax check before writing**; `expectedHash` / `expectedSource` guard against lost updates | Preferred path for BSL edits while `edt-mcp` is exposed — the pre-write syntax check is a validation gate for free. Native `Edit` stays the fallback |
| **find_references** | Every usage of a metadata object: BSL (with lines), other metadata, forms, roles, subsystems | "Where is X used?" — impact analysis before rename / removal / signature change |
| **get_method_call_hierarchy** | `callers` / `callees` / `outgoing` via semantic AST (resolves ru/en spellings) | Call chains, impact analysis; `outgoing` aggregates a module's distinct call targets |
| **get_outgoing_structures** | Literal top-level keys of the `Структура` passed as the first argument of each outgoing call | Reconstruct an informal structure contract between modules |
| **go_to_definition** | Where a symbol is defined (`Module.Method`, bare method + `modulePath`, or a metadata FQN) | Inverse of `find_references` |
| **get_symbol_info** | Type / hover info at line+column | Resolve what an identifier is at a concrete position |
| **get_content_assist** | Completion proposals at line+column | Discover an object's API members; disambiguate an expression |
| **validate_query** | Validate 1C query text against the project's metadata (`dcsMode` for СКД) | Check a query **before** embedding it in BSL or a DCS data set |

## Metadata — typed mutation API

No hand-written XML: metadata is created and edited through a typed FQN API that also refactors references.

| Tool | Purpose | Notes |
|---|---|---|
| **create_metadata** | Create a top object (`Catalog.Products`) or a member (`Catalog.Products.Attribute.Weight`, `InformationRegister.Prices.Dimension.Product`, `Enum.Colors.EnumValue.Red`, form items / attributes / commands / handlers) | Common-module flag presets (`commonModuleKind`, `serverCall`, `privileged`, `returnValuesReuse`); form creation with `generateContent` / `objectFields` / `setAsDefault`; `callType` (`Before`/`After`/`Instead`) for **extension** event interception; `normalizeYo` normalizes «ё» in names |
| **modify_metadata** | Set properties of an object / member; move-reorder form items (`parent`, `position`); rebind a form handler (`procedure`); set a dynamic-list `queryText`; author a **Role** (`rights`, `templates`, `roleProperties`, incl. RLS); author a **report's DCS** (`dcs`); author a **spreadsheet template** (`template`: cells, merges, areas, widths); edit membership lists (`content`: common-attribute owners, exchange-plan content, catalog owners, document register records, subsystem content) | This single tool covers role / SKD / MXL / interface authoring |
| **delete_metadata** | Delete an object or member, cascading reference cleanup | Two-phase: preview, then `confirm=true`; blocked on un-cleanable references unless `force=true` |
| **rename_metadata_object** | Rename with cascade across BSL, forms, metadata | Two-phase preview → `confirm=true`; `disableIndices` skips optional change points |
| **adopt_metadata_object** | Adopt a base object / member into a configuration **extension** (EDT "Add To Extension") | Adopting BSL methods is **not** covered |
| **list_subsystems** / **get_subsystem_content** | Subsystem tree and its content | Placing new objects into subsystems |
| **get_configuration_properties** | Configuration-level properties (compatibility mode, name, …) | Confirm the platform version instead of trusting memory |
| **list_common_pictures** / **export_common_picture** | Common-picture inventory and export | Icons for forms / commands |
| **list_configurations** / **create_launch_config** / **delete_launch_config** | EDT launch configurations (runtime client / Attach) with running state | Discovery step before `debug_launch`, `run_yaxunit_tests`, `update_database` |

## Project — validation, DB update, logs

| Tool | Purpose | When to use |
|---|---|---|
| **revalidate_objects** | Revalidate the project or specific FQNs; refreshes from disk first | After any edit made outside EDT (`.mdo` / `.bsl` / `.form`) — the cheap, targeted alternative to `clean_project` |
| **get_project_errors** | EDT validation markers with check id, message, object, severity, and a module path + line locator | The validation workhorse: filter by `severity` (`ERRORS`/`BLOCKER`/`CRITICAL`/`MAJOR`/`MINOR`/`TRIVIAL`), `checkId`, `objects` |
| **get_problem_summary** | Marker counts grouped by project and severity | Confirming pass / health check after a fix |
| **get_check_description** | Full explanation of a check by symbolic id or short UID | Understand and justify a finding; requires the check-descriptions folder configured in MCP preferences |
| **get_markers** | Bookmarks and TODO / FIXME / XXX / HACK task markers | Technical-debt inventory (`TODO No.<task>:` per the code-style standard) |
| **clean_project** | DISK → MODEL full rebuild + revalidation | Slow; discards unsaved in-memory model edits. Prefer `revalidate_objects` for one object |
| **resync_to_disk** | MODEL → DISK: export missing `.mdo`, report / clean dangling `Configuration.mdo` references | Fixes "object file does not exist" and `md-reference-intergrity` failures that block `update_database` |
| **update_database** | Apply the configuration to an infobase, full or incremental | Two-phase: call without `confirm` for a preview, then `confirm=true`. Auto-terminates 1C clients this EDT launched. Driven by `/update1cbase` |
| **create_infobase** / **delete_infobase** / **set_infobase_credentials** | File infobase create / register (incl. `standaloneServer` with a web URL) and stored credentials | Preparing a test infobase; `standaloneServer` + `mode=register` gives an HTTP-testable URL |
| **export_configuration_to_xml** / **import_configuration_from_xml** | XML dump / import into a **new** EDT project | Interop with a classic Designer-shaped repository |
| **build_external_objects** | Compile an external-objects project to `.epf` / `.erf` | Building external processors / reports |
| **create_project** / **delete_project** | EDT project lifecycle | Rare; destructive |
| **get_event_log** | Read a 1C infobase event log **without a running session** by parsing `1Cv8.lgf` + `*.lgp`; filters by period, user, event, severity, comment, metadata, session | The runtime-error source for the systematic-debugging Phase 1. Returns infobase PII — do not paste user data into artifacts |
| **get_platform_documentation** | Platform types / global functions with members, pinned to the project's platform version | Exact platform API signature; complements `docinfo` on `1C-docs-mcp` |
| **get_mcp_history** | This server's own request/response ring, with per-tool context-usage stats | Self-audit: which calls are burning context / failing |
| **list_git_branches** / **create_git_branch** / **switch_git_branch** / **set_branch_infobase** | Branch operations bound to EDT, incl. per-branch infobase binding | Branch switching that must keep EDT and the infobase in sync; plain git still goes through the shell |

## Forms

| Tool | Purpose | When to use |
|---|---|---|
| **get_form_layout_snapshot** | YAML snapshot of the calculated WYSIWYG layout (bounds, element types, display properties) | Inspect or diff what a form actually renders |
| **get_form_screenshot** | PNG of the form's WYSIWYG editor | Visual check of a layout change |
| **get_template_screenshot** | PNG of a spreadsheet template | Visual check of an MXL layout |

> Both form renderers require EDT started with `-DnativeFormBufferedLayoutRender=true`; without it the output is blank — that is a missing JVM flag, not a bad call (`get_server_status` reports both flags). **Form structure** (items, attributes, commands, handlers) is read through `get_metadata_details` on a form FQN and mutated through `create_metadata` / `modify_metadata` / `delete_metadata`.

## Debug, testing, profiling

| Tool | Purpose |
|---|---|
| **debug_launch** / **terminate_launch** / **debug_status** | Start (`restartIfRunning`, `updateBeforeLaunch`) / stop / inspect a debug session; Attach configs debug server-side code |
| **set_breakpoint** / **remove_breakpoint** / **list_breakpoints** / **wait_for_break** / **step** / **resume** | Breakpoint-debugging controls for the systematic-debugging Phase 3 |
| **get_variables** / **set_variable** / **evaluate_expression** | Inspect and evaluate in the paused context. **Read-only by default** — never evaluate `Записать()` / `Удалить()` / DML without explicit user consent, and only on a copy infobase |
| **get_applications** | Infobases bound to the project (id, name, type, update state) — the `applicationId` source |
| **run_yaxunit_tests** / **debug_yaxunit_tests** | Run YAXUnit suites (JUnit markdown report) or launch them under the debugger; `updateBeforeLaunch` auto-rebuilds and updates first |
| **get_job_status** / **cancel_job** / **ask_workmate** | Background jobs; `ask_workmate` delegates a question to the 1C:Workmate plugin as a job — poll with `get_job_status` instead of re-launching |
| **start_profiling** / **stop_profiling** / **get_profiling_results** | Performance measurement for a performance task |

## Tags and translation

| Tool | Purpose |
|---|---|
| **get_tags** / **get_objects_by_tags** | Tag-based object discovery — find comparable existing objects / forms as examples |
| **generate_translation_strings** / **translate_configuration** / **get_translation_project_info** | LanguageTool-based configuration translation (extract, translate, project info) |

## Which server first

| Question | Server |
|---|---|
| Code search, references, call chains, metadata facts, forms — in the project | `edt-mcp` |
| BSL / query / metadata verdicts, what is broken **in the workspace right now** | `edt-mcp` (`revalidate_objects` → `get_project_errors` / `get_problem_summary`) |
| Exact references / definitions as EDT resolves them; content assist at a caret | `edt-mcp` |
| Metadata mutation in an **EDT-format** (MDO) tree | `edt-mcp` (`create_metadata` / `modify_metadata` / …) — through the `1c-metadata-manage` gate |
| Metadata mutation in a **Designer XML dump** | `1c-metadata-manage` skill scripts — unchanged hard gate |
| Applying configuration changes to an infobase, launching, debugging, profiling | `edt-mcp` |
| Platform documentation, routed standards, file-format specs | `1C-docs-mcp` (`get_platform_documentation` on `edt-mcp` is pinned to the project's version) |
| БСП / SSL API | `1c-ssl-mcp` |
| ITS standards and diagnostic → standard mapping | `v8std-local` |
| Code templates, project memory | `1c-templates-fastcode` |

`search_in_code` is a literal / regex textual sweep and is **not** ru/en dialect aware — it misses the other BSL language variant. Use `find_references` / `get_method_call_hierarchy` / `go_to_definition` for identifiers, and this tool for textual confirmation and literals.

## Remap from the classic docker distribution

Other rulesets and older notes name the classic multi-server distribution's tools. Their `edt-mcp` counterparts:

| Classic tool(s) | `edt-mcp` | Confidence |
|---|---|---|
| `search_code` / `codesearch` | `search_in_code` | approx — literal / regex, no semantic or BM25 lane |
| `search_function` | `get_module_structure` → `read_method_source`, or `go_to_definition` | equivalent |
| `bsl_scope_members` | `get_content_assist` | approx (position-based, not type-based) |
| `get_module_structure` | `get_module_structure` | exact-equivalent |
| `search_metadata` / `metadatasearch` / `search_metadata_by_description` / `business_search` | `get_metadata_objects` (name / type / synonym filter); `get_objects_by_tags` | approx — no semantic business search |
| `get_object_dossier` / `get_metadata_details` / `resolve_qualified_name` | `get_metadata_details` | exact-equivalent (also covers roles and forms) |
| `trace_impact` / `graph_dependencies` / `find_usages_of_object` / `find_objects_using_object` / `find_register_movement_docs` | `find_references` (+ `get_method_call_hierarchy`) | approx — single hop, no transitive graph |
| `trace_call_chain` / `get_method_call_hierarchy` | `get_method_call_hierarchy` | exact-equivalent (+ `outgoing`) |
| `search_forms` / `inspect_form_layout` / `get_form_artifact` / `get_form_structure` | `get_objects_by_tags` (comparable forms), `get_metadata_details` (form FQN), `get_form_layout_snapshot`, `get_form_screenshot` | equivalent, richer |
| `get_xsd_schema` / `verify_xml` | typed `create_metadata` / `modify_metadata` + `revalidate_objects` → `get_project_errors` | **architecturally different** — see gaps |
| `syntaxcheck` / `syntaxcheck_file` | pre-write check of `write_module_source` + `revalidate_objects` → `get_project_errors` | equivalent (two steps: refresh, then read markers) |
| `check_1c_code` / `review_1c_code` | `get_project_errors` → `get_check_description` / `get_problem_summary` | approx — rule-based, no AI review |
| `ask_1c_ai` / `rewrite_1c_code` / `modify_1c_code` / `answer_metadata_question` | — | **no replacement** (`ask_workmate` delegates to the 1C:Workmate plugin when installed) |
| `its_help` / `fetch_its` | `v8std_search` / `v8std_explain_diagnostics` / `v8std_get_page` on `v8std-local` | equivalent |
| `helpsearch` | `docsearch` / `docinfo` on `1C-docs-mcp`; `get_platform_documentation` | equivalent |
| `validatequery` | `validate_query` | exact-equivalent (offline, project metadata; `dcsMode` for СКД) |
| `vcexecutequery` / `vcexecutecode` | `get_applications` → `debug_launch` → `evaluate_expression` | approx — needs a debug session, heavy |
| `vcloggetlasterror` | `get_event_log` | exact-equivalent (offline) |
| `list_graph_projects` / graph `project_id` | `list_projects` / `projectName` | equivalent |

## Coverage gaps

- **No AI per-method analysis and no prose code-review tool.** Logic / performance findings come from EDT's own check engine (`get_project_errors`, explained by `get_check_description`) plus the internal review checklist in the code-style standard. EDT checks cover a real slice (`ql-*` query checks, `begin-transaction`, standards checks), but they are rule-based, not a reasoning pass — do not claim parity.
- **No semantic / hybrid code search and no graph traversal engine.** `search_in_code` is literal / regex; `find_references` + `get_method_call_hierarchy` cover single-hop and call-chain navigation through EDT's index; multi-hop transitive impact across mixed relationship types has no equivalent.
- **ITS-standards search / fetch — covered by `v8std-local`, not by `edt-mcp` itself.** `docsearch` / `docinfo` and `get_platform_documentation` cover platform APIs; `v8std-local` covers ITS methodology standards and their diagnostic mapping. Only when `v8std-local` is not exposed in the session does the old caveat apply — say so explicitly and treat an ITS claim as unverified.
- **No standalone syntax-only checker for a free-floating snippet.** `write_module_source` syntax-checks what it writes and `validate_query` checks query text, but BSL validation otherwise expects the code to live in a real module of the workspace.
- **No XSD / raw-XML payload validation.** Metadata correctness is enforced by the typed API plus EDT validation (`revalidate_objects` → `get_project_errors`). Hand-edited `.mdo` / `.form` files are validated the same way, not against a schema.
- **No live-infobase data service.** Runtime data access goes through a debug/Designer session (`debug_launch` → `evaluate_expression` / `get_variables`) — stateful and heavy. `get_event_log` is the one exception: it reads log files directly, no session needed.
- **No import directly from a live infobase.** `import_configuration_from_xml` needs an XML dump; `create_project` builds a project from scratch.
- **No agent-orchestration / QA-scenario tools.** Use the native `Agent` tool and skills; project memory stays on `1c-templates-fastcode` (`remember` / `recall`); YAXUnit tests are written by hand and run with `run_yaxunit_tests`.
- **No session / lock / process introspection.** `get_applications`, `list_configurations`, `get_server_status` and the `webUrl` returned by `create_infobase(applicationKind='standaloneServer')` cover part of it; lock-conflict diagnosis falls back to the technological log per the locks-and-transactions standard.

## Safety and discipline

- **Destructive operations require consent by default** — `delete_metadata`, `rename_metadata_object`, `delete_project`, `delete_infobase`, `update_database`, and `modify_metadata` when it changes a data type. The prompt is the user's decision point. Never disable it on the user's behalf: the "Allow all" consent level and the `EDT_MCP_DESTRUCTIVE_CONSENT=allow` override are for unattended automation the user explicitly asked for, and you must say what they remove.
- **Full trust surface.** The server exposes filesystem, code-writing, DB-update and debug operations to every connected client. Loopback binding is the default and should stay; a non-loopback bind requires an auth token and a trusted network. Never print or commit that token.
- **The model is the authority, the disk lags.** `resync_to_disk` flushes model → disk and reports desync. Reconcile before mixing `edt-mcp` with file-level tools, and never alternate owners on one artifact — canon: `content/rules/edt-workflow.md → Model vs disk — the synchronization rule`.
- **Debug / profiling / `update_database` act on a real application.** Confirm the target is a dev / test base first, exactly as the deployment slash commands require.
- **`import_configuration_from_xml` creates a new project**, it does not update the current one. Confirm with the user before any export/import round trip.
- **Budget rules are unchanged** (`content/rules/mcp-policy.md` section C): no repeated calls against unchanged state, and EDT markers do not open a retry loop of their own. `get_mcp_history` shows what this session already asked — use it instead of re-running a call to remember its result.
