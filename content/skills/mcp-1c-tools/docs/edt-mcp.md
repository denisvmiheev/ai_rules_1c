# edt-mcp — tool catalog

MCP server plugin running **inside 1C:EDT** (upstream: <https://github.com/DitriXNew/EDT-MCP>). It exposes the live EDT workspace: projects, metadata model, BSL modules, EDT validation markers, forms rendering, debug sessions, YAXUnit, profiling, infobases and launch configurations. In this fork it is the default 1C server, in place of upstream's classic docker distribution's `1c-code-metadata-mcp`, `1c-graph-metadata-mcp`, `1c-syntax-checker-mcp`, `1c-code-check-mcp`, and `1c-data-mcp` (see the remap and coverage-gap tables below).

> Load this file only if the `edt-mcp` server is actually available in the current session (its tools are visible in the agent tool schema).

> **Do not invent parameter names.** Claude Code exposes each tool's real JSON schema at call time — read it before any parameter-rich call. For extended per-tool documentation that is deliberately kept out of `tools/list`, call **`get_tool_guide('<toolName>')`**.

## Session basics

- **`projectName` is required by almost every tool.** Confirm the current project's name with `list_projects` when in doubt — a workspace can have more than one entry, and not every entry is a valid EDT project.
- **Module paths are `src/`-relative**, e.g. `CommonModules/MyModule/Module.bsl`, `Documents/SalesOrder/ObjectModule.bsl`.
- **Metadata is addressed by FQN**, e.g. `Catalog.Products`, `Catalog.Products.Attribute.Weight`, `Catalog.Products.Form.ItemForm`, `Role.FullAccess`. English **and** Russian type tokens are accepted (`Справочник.Товары`).
- **Progressive disclosure** (EDT Preferences → MCP Server) can hide toolsets. `list_toolsets` shows the groups and their visibility; `enable_toolset` reveals one.
- **`get_server_status`** is the self-diagnosis snapshot: port, versions, enabled tool counts, whether the check-descriptions folder is configured, and the two form-render JVM flags. Read it before blaming a blank form screenshot or a missing `get_check_description`.

## Toolsets

`core`, `metadata`, `code`, `debug`, `testing`, `profiling`, `forms`, `tags`, `translation`, `project`.

## Core — discovery and reading

| Tool | Purpose | When to use |
|---|---|---|
| **list_projects** | Workspace projects, state, natures | First orientation; confirm the `projectName` to pass everywhere else |
| **list_modules** | BSL modules of a project (path, module type, owner) | Discover module paths before reading / editing; filter by `metadataType`, `objectName`, `nameFilter` |
| **get_module_structure** | Procedures / functions with signatures, regions, `&AtServer`/`&AtClient`, export flag, line numbers | Understand a module before editing (`responseFormat=detailed` for parameters and doc-comments) |
| **read_module_source** | Module source, whole file or line range; returns a `contentHash` revision token | Read the edit target; the hash round-trips into `write_module_source.expectedHash` |
| **search_in_code** | Literal / regex full-text search across BSL modules | Primary code search. **Not** ru/en dialect-aware — for identifiers prefer `find_references` / `get_symbol_info` / `get_method_call_hierarchy` |
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
| **modify_metadata** | Set properties of an object / member; move-reorder form items (`parent`, `position`); rebind a form handler (`procedure`); set a dynamic-list `queryText`; author a **Role** (`rights`, `templates`, `roleProperties`, incl. RLS); author a **report's DCS** (`dcs`); author a **spreadsheet template** (`template`: cells, merges, areas, widths); edit membership lists (`content`: common-attribute owners, exchange-plan content, catalog owners, document register records, subsystem content) | This single tool replaces the old role / SKD / MXL / interface mini-tools |
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
| **get_markers** | Bookmarks and TODO / FIXME / XXX / HACK task markers | Technical-debt inventory (`TODO No.<task>:` per `dev-standards-code-style.md`) |
| **clean_project** | DISK → MODEL full rebuild + revalidation | Slow; discards unsaved in-memory model edits. Prefer `revalidate_objects` for one object |
| **resync_to_disk** | MODEL → DISK: export missing `.mdo`, report / clean dangling `Configuration.mdo` references | Fixes "object file does not exist" and `md-reference-intergrity` failures that block `update_database` |
| **update_database** | Apply the configuration to an infobase, full or incremental | Two-phase: call without `confirm` for a preview, then `confirm=true`. Auto-terminates 1C clients this EDT launched. Driven by `/update1cbase` |
| **create_infobase** / **delete_infobase** / **set_infobase_credentials** | File infobase create / register (incl. `standaloneServer` with a web URL) and stored credentials | Preparing a test infobase; `standaloneServer` + `mode=register` gives an HTTP-testable URL |
| **export_configuration_to_xml** / **import_configuration_from_xml** | XML dump / import into a **new** EDT project | Interop with a classic Designer-shaped repository |
| **build_external_objects** | Compile an external-objects project to `.epf` / `.erf` | Building external processors / reports |
| **create_project** / **delete_project** | EDT project lifecycle | Rare; destructive |
| **get_event_log** | Read a 1C infobase event log **without a running session** by parsing `1Cv8.lgf` + `*.lgp`; filters by period, user, event, severity, comment, metadata, session | The runtime-error source for `systematic-debugging.md` Phase 1. Returns infobase PII — do not paste user data into artifacts |
| **get_platform_documentation** | Platform types / global functions with members, pinned to the project's platform version | Exact platform API signature; complements `1c-platform-mcp` |
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
| **set_breakpoint** / **remove_breakpoint** / **list_breakpoints** / **wait_for_break** / **step** / **resume** | Breakpoint-debugging controls for `systematic-debugging.md` Phase 3 |
| **get_variables** / **set_variable** / **evaluate_expression** | Inspect and evaluate in the paused context. **Read-only by default** — never evaluate `Записать()` / `Удалить()` / DML without explicit user consent, and only on a copy infobase |
| **get_applications** | Infobases bound to the project (id, name, type, update state) — the `applicationId` source |
| **run_yaxunit_tests** / **debug_yaxunit_tests** | Run YAXUnit suites (JUnit markdown report) or launch them under the debugger; `updateBeforeLaunch` auto-rebuilds and updates first |
| **start_profiling** / **stop_profiling** / **get_profiling_results** | Performance measurement for `1c-performance-optimizer` work |

## Tags and translation

| Tool | Purpose |
|---|---|
| **get_tags** / **get_objects_by_tags** | Tag-based object discovery — find comparable existing objects / forms as examples |
| **generate_translation_strings** / **translate_configuration** / **get_translation_project_info** | LanguageTool-based configuration translation (extract, translate, project info) |

## Remap from the upstream docker distribution's classic-catalog servers

`edt-mcp` is a live-EDT plugin, not a docker container from the vendor distribution — it replaces the following upstream servers with a different architecture (semantic AST via EDT's own model, not a separate index):

| Upstream classic server / tool | `edt-mcp` equivalent | Confidence |
|---|---|---|
| `1c-code-metadata-mcp`: `metadatasearch` | `get_metadata_objects` / `get_metadata_details` | exact-equivalent |
| `1c-code-metadata-mcp`: `codesearch` | `search_in_code` | exact-equivalent |
| `1c-code-metadata-mcp`: `get_module_structure` | `get_module_structure` | exact-equivalent |
| `1c-code-metadata-mcp`: `get_method_call_hierarchy` | `get_method_call_hierarchy` | exact-equivalent (+ `outgoing` direction) |
| `1c-code-metadata-mcp`: `graph_dependencies` | `find_references` | equivalent, single-hop |
| `1c-code-metadata-mcp`: `search_forms` | `get_form_layout_snapshot` / `get_metadata_details` (form FQN) | equivalent |
| `1c-code-metadata-mcp`: `bsl_scope_members` | `get_content_assist` | approx (position-based, not type-based) |
| `1c-graph-metadata-mcp`: `search_code` / `search_metadata` / `search_metadata_by_description` | `search_in_code` / `get_metadata_objects` | equivalent, no separate semantic index |
| `1c-graph-metadata-mcp`: `trace_impact` / `trace_call_chain` (recursive, depth-configurable) | `find_references` + `get_method_call_hierarchy` (single/multi-hop through EDT's own index) | narrower — no standalone graph-DB traversal, see gaps |
| `1c-graph-metadata-mcp`: `get_object_dossier` | `get_metadata_details` | equivalent |
| `1c-graph-metadata-mcp`: `business_search` | `get_metadata_objects` (`nameFilter`) + `search_in_code` | approx |
| `1c-syntax-checker-mcp`: `syntaxcheck` / `syntaxcheck_file` | `write_module_source` (pre-write check) / `revalidate_objects` → `get_project_errors` | equivalent, integrated into the edit flow rather than a standalone checker |
| `1c-code-check-mcp`: `check_1c_code` / `review_1c_code` | `get_project_errors` (explained by `get_check_description`) | rule-based, not a prose AI review — see gaps |
| `1c-code-check-mcp`: `its_help` / `fetch_its` | `v8std-local` (`v8std_search` / `v8std_explain_diagnostics` / `v8std_get_page`, see `docs/v8std.md`) | equivalent, richer (adds diagnostic↔standard mapping) |
| `1c-data-mcp`: `vcexecutecode` / `vcexecutequery` | `debug_launch` → `evaluate_expression` (heavier-weight, needs a debug session) | narrower, no lightweight live-IB HTTP endpoint |
| `1c-data-mcp`: `validatequery` | `validate_query` | exact-equivalent |
| `1c-data-mcp`: `vcloggetlasterror` | `get_event_log` | equivalent, offline (no running session needed) |

`edt-mcp` also replaced a project-local `edt-codepilot` plugin generation earlier; see git history of this file for that remap if relevant.

## New capabilities the classic catalog did not have

- **`get_event_log`** — reads the infobase event log offline (no running 1C session), including for the runtime-error path in `systematic-debugging.md`.
- **`get_check_description`** — the full text of an EDT check (what it means, examples, how to fix). For a check that mirrors an ITS standard, pair it with `v8std_explain_diagnostics` on `v8std-local` (see `docs/v8std.md`) to pull the actual standard clause.
- **`get_form_screenshot` / `get_template_screenshot`** — visual verification of forms and MXL layouts.
- **`write_module_source` with a pre-write BSL syntax check and a lost-update guard** (`expectedHash` / `expectedSource`).
- **Two-phase destructive operations** — `delete_metadata`, `rename_metadata_object`, `update_database` all preview first, apply on `confirm=true`.
- **`rename_metadata_object` / `adopt_metadata_object`** — cascading rename and EDT "Add To Extension" as first-class operations.
- **`resync_to_disk` / `revalidate_objects` / `clean_project`** — explicit control of the MODEL ↔ DISK direction, the standard fix for BM-to-disk desync.
- **`get_markers`** — bookmarks and TODO/FIXME inventory.
- **`get_outgoing_structures`** — reconstructs informal `Структура` contracts between modules.
- **Infobase and launch-config lifecycle** — `create_infobase` (incl. standalone server with a web URL for HTTP/UI testing), `set_infobase_credentials`, `create_launch_config`, `list_configurations`.
- **Profiling** — `start_profiling` / `stop_profiling` / `get_profiling_results`.
- **Translation toolset** — `generate_translation_strings`, `translate_configuration`, `get_translation_project_info`.
- **Subsystems / common pictures** — `list_subsystems`, `get_subsystem_content`, `list_common_pictures`, `export_common_picture`.
- **`get_mcp_history`** — self-audit of the agent's own MCP traffic and its context cost.
- **`get_tool_guide` / `list_toolsets` / `enable_toolset`** — on-demand documentation and progressive disclosure.

## Coverage gaps

- **No AI per-method analysis.** No `check_1c_code`/`review_1c_code`-style AI pass. Logic / performance findings now come from EDT's own check engine (`get_project_errors`, explained by `get_check_description`) plus the internal review checklist in `dev-standards-code-style.md §8`. EDT checks do cover a real slice of this (`ql-*` query checks, `begin-transaction`, standards checks), but they are rule-based, not a reasoning pass — do not claim parity.
- **No dedicated prose code-review tool.** `get_project_errors` / `get_problem_summary` are marker-based, not a prose review.
- **ITS-standards search / fetch — covered by `v8std-local`, not by `edt-mcp` itself.** `1c-platform-mcp` (`docsearch` / `docinfo`) and `get_platform_documentation` cover platform APIs; `v8std-local` (`v8std_search` / `v8std_explain_diagnostics` / `v8std_get_page`, see `docs/v8std.md`) covers ITS methodology standards and their diagnostic mapping. Only when `v8std-local` is not exposed in the session does the old caveat apply — say so explicitly and treat an ITS claim as unverified.
- **No graph traversal engine.** `find_references` + `get_method_call_hierarchy` cover single-hop and call-chain navigation through EDT's index; multi-hop transitive impact across mixed relationship types has no equivalent (the classic distribution's Neo4j-backed `1c-graph-metadata-mcp` did this).
- **No standalone syntax-only checker for a free-floating snippet.** `write_module_source` syntax-checks what it writes and `validate_query` checks query text, but BSL validation otherwise expects the code to live in a real module of the workspace.
- **No XSD / raw-XML payload validation.** Metadata correctness is enforced by the typed API plus EDT validation (`revalidate_objects` → `get_project_errors`). Hand-edited `.mdo` / `.form` files are validated the same way, not against a schema.
- **No lightweight live-infobase data service.** Runtime data access goes through a debug/Designer session (`debug_launch` → `evaluate_expression` / `get_variables`) — stateful and heavier than the classic distribution's anonymous-HTTP `1c-data-mcp`. `get_event_log` is the one exception: it reads log files directly, no session needed.
- **No import directly from a live infobase.** `import_configuration_from_xml` needs an XML dump; there is no direct from-a-running-infobase import.
- **No session / lock / process introspection.** No dedicated infobase-locks / OS-process inventory tool. `get_applications`, `list_configurations`, `get_server_status` cover part of it; lock-conflict diagnosis falls back to the technological log per `locks-and-transactions.md §6`.
