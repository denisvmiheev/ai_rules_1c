---
description: MCP-first search — project-index MCP tools get a bounded attempt before Grep, Glob, directory listing or Read-scanning, with a one-line "what was tried" note on fallback. Load before any code, metadata, usage or file search in a 1C project.
alwaysApply: false
category: tooling
---

# MCP-first search discipline

For any 1C **project-source search** (code, metadata, usages, call chains, structure, forms, layouts, **and locating the source files themselves**) — MCP project-index tools come **first**. Native discovery tools — `Grep` / `rg`, `Glob` / file search by pattern (`**/*.bsl`, `**/*.xml`), directory listing, semantic codebase search, sequential `Read`-scanning of modules — are the **last resort**, gated by an explicit justification note.

> In this ruleset the project-index server is `edt-mcp` (the 1C:EDT plugin) — see `content/skills/mcp-1c-tools/docs/edt-mcp.md`. The priority and fallback discipline below applies to whatever project-index tools the session exposes.

**What counts as search.** Any action whose goal is to *locate* code, metadata, or files you do not yet have an exact path or qualified name for — including "getting oriented" sweeps at the start of a task (globbing the source tree, listing directories, reading modules one after another to see what is there). All of it falls under the hard rule below. Reading a specific file **already located via MCP** — to edit it, or to see a found fragment in full context — is normal work, not search, and needs no justification.

This file is the single owner of the search discipline and of the project-source fallback chain; `AGENTS.md → MCP Tool Calling → A.4`, the `mcp-1c-tools` router, the operation skills and every subagent prompt point here. It applies to the parent and to every subagent.

**Policy and scope before tools.** Apply `TOOL_EDT` (`content/rules/mcp-policy.md → Tool availability`). Skip an `off` server without probes. Without an eligible index, scoped native search starts immediately; a missing `required` index remains an unmet requirement even if local reads gather context.

**Explicit project on scoped calls.** Every `edt-mcp` call takes `projectName`. Extension work and a workspace with several projects require the `projectName` of the contour you mean (`list_projects`); state it once and supply it on every call, including retries. Never rely on an omitted or default project, the first returned project or a configuration display name. Resolve ambiguity from existing mappings, then ask only if still unresolved; meanwhile scoped local work can continue.

---

## Hard rule

1. **Before any native discovery call on project source** (`Grep` / `rg`, `Glob` / file search, directory listing, semantic codebase search, `Read`-scanning, full-module `Read` for the sake of one routine — see *Full-file `Read` — fragment first* below), you MUST first exhaust the eligible project-index path. Only a content query counts: `list_projects`, schema or status lookups and memory `recall` are not the index attempt.
   1. `edt-mcp` search / navigation tools as applicable — `get_metadata_objects`, `get_metadata_details`, `find_references`, `get_method_call_hierarchy`, `go_to_definition`, `get_module_structure`, `get_objects_by_tags` / `get_tags`, `get_content_assist`, `get_form_layout_snapshot`. Full catalog — `content/skills/mcp-1c-tools/docs/edt-mcp.md`.
   2. `search_in_code` as a literal / regex retry (`isRegex`, `fileMask`, `metadataType`, `outputMode=files`) — **only** after step 1 returned not enough. Typical triggers: exact identifier, fragment of a query, event-handler name, error text, literal string. It is purely textual and **not** ru/en dialect-aware; `edt-mcp` has no `grep` / `glob` tools of its own, so file-level pattern search falls straight through to the native tools below.
2. **Only then `Grep` / `Glob` or another native discovery tool** — and only when you can state, in one or two sentences inside the response, **which MCP attempts were tried and why they did not return what was needed**. Silent fallback is a defect regardless of which native tool it lands on — `Grep`, a file-pattern search, or a chain of `Read` calls used as a manual scanner. **"Exhaust" is bounded:** one well-tuned call per applicable angle, with one reformulation only when it can help — not an open-ended loop. Once that missed (nothing found, irrelevant hits, non-actionable output), falling back to native tools is the **correct next move**, not a defect; burning further MCP calls just to satisfy this rule is blind chaining (`AGENTS.md → A.4`).
3. **Tune the query before re-calling.** If the first MCP search returned nothing and query reformulation can help, reformulate once: broaden / narrow the query, a narrower `fileMask` / `metadataType`, the regex form, or a structural tool instead of text. Use only parameters exposed by that tool (its live schema, `get_tool_guide`), with the names from the operation skill (`1c-code-search`, `1c-meta-info`, `1c-impact`, `1c-form-inspect`).
   **A closed answer stays closed.** A typed error or a warning naming a missing capability is the server's answer for this session: take the action its code maps to in `content/rules/mcp-policy.md → C. Server answers → actions` in one step. Probing the same gap through sibling tools is blind chaining, not diligence.
4. **No-change repeats are forbidden.** Do not re-run the same MCP call against the same unchanged state. A new call must change parameters substantively, or the project state must have changed (file edit, new generation, resumed session).
5. **A negative result needs freshness evidence.** An empty search response proves "not found" only when the EDT model represents the relevant project state: after edits made outside EDT (git checkout, a script, a slash command) run `resync_to_disk` / `revalidate_objects` first (`content/rules/edt-workflow.md → Model vs disk`); `get_server_status` shows the workspace state. When evidence is absent, stale, or contradicted by fresh local edits, call the result inconclusive and continue to the next documented MCP or disk fallback. Do not add a health call merely to decorate a search whose absence you do not rely on.

External-knowledge servers (`1c-templates-fastcode`, `1c-ssl-mcp`, `1C-docs-mcp`, `v8std-local`) have **no `Grep` / `rg` equivalent** — they are called only when their knowledge is needed, not as part of the fallback above. See `content/skills/mcp-1c-tools/docs/edt-mcp.md → Coverage gaps` for what `edt-mcp` does not cover (no dedicated BSL-syntax-only checker for a free-floating snippet, among others).

**Read-only metadata questions:** retrieve only what answers the question: names / synonyms via `get_metadata_objects` for object identification, `get_metadata_details` for a field list. Do not expand an identification hit into full details unless its structure is needed.

One of them also holds **this project's own routed standards**: the `1c-standards` collection of `1C-docs-mcp`, reached with the `standards` tool (never with `docsearch` / `docinfo`, which cannot see it). That is a *rule* lookup, not a project-source search — it is outside the chain above and has its own contract in `content/rules/help-corpus-retrieval.md`. Reading a rule file on disk is likewise ordinary work, not search.

---

## Full-file `Read` — fragment first

`Read` of a whole BSL module to *find or understand one routine* is the same fallback as `Grep`. When the need is a fragment — retrieve at fragment level first: `get_module_structure(projectName, modulePath)` for the layout, `read_method_source` for one routine's full body, `read_module_source(startLine, endLine)` for a line range, `get_content_assist` / `get_symbol_info` for what is available at a position.

Full-file `Read` is **normal work** — no MCP attempt, no justification note — when:

- the file is your **direct edit target** (reading before editing is mandatory, not a fallback);
- the file is small (a few screens) — slicing it via MCP costs more than reading it;
- you (or the user) **just edited** the file — the MCP index may lag behind the disk; the disk state is the authority;
- whole-module context genuinely is the task (module-wide review, refactor, region restructuring);
- the fragment returned by MCP is truncated or insufficient and full context is needed to proceed.

---

## Quick first-pick table

The table names tool roles on `edt-mcp`; a workspace with several projects needs `projectName` on every call (`content/skills/mcp-1c-tools/SKILL.md`).

| Need | First call (MCP) | If empty — next |
|---|---|---|
| Find BSL code by behaviour / description | `search_in_code` | reformulate the query, or narrow/broaden scope, before falling back |
| Find BSL code by exact identifier / literal | `search_in_code` (`isRegex` / `fileMask` / `metadataType`) | `find_references` / `go_to_definition` for a symbol → only then native `Grep` |
| Find a routine by name | `get_module_structure` → `read_method_source` | `go_to_definition` / `search_in_code` → `Grep` |
| Understand a metadata object | `get_metadata_details` | reformulate the object name / re-check via `get_metadata_objects` |
| Metadata search by name / structure | `get_metadata_objects` | `search_in_code` if the index misses |
| Metadata search by Russian description / synonym | `get_metadata_objects` | `search_in_code` |
| Usages of an object | `find_references` | broaden scope / reformulate |
| Impact of an object change | `find_references` | `get_method_call_hierarchy` for the call-chain angle |
| Call graph (who calls / who is called) | `get_method_call_hierarchy` | reformulate the routine reference |
| Locate the source files / modules of an object | `get_metadata_details` (paths in the details) → `get_module_structure` | `get_metadata_objects` |
| "Get oriented" in an unfamiliar configuration | `get_metadata_objects` | `list_modules` — **not** `Glob **/*.bsl` |
| Module structure overview | `get_module_structure` | `get_form_layout_snapshot` for forms |
| Form layout | `get_form_layout_snapshot` | `get_objects_by_tags`/`get_tags` |
| Canonical pattern / template | **`templatesearch` only** — task description verbatim; pre-flight `1c-templates-fastcode.md → Query formulation (templatesearch only)` (`AGENTS.md → A.8`) (+ `ssl_search` for БСП) | — |
| Platform API verification | `docinfo` or `docsearch` | `get_platform_documentation` (`edt-mcp`) |
| A routed project standard (`anti-patterns`, `dev-standards-architecture`, …) | `standards(name="<rule stem>")` — **not** `docsearch`; `content/rules/help-corpus-retrieval.md` | `standards(query=…)` → `standards()` catalogue |
| On-disk XML shape of a form / role / DCS / MXL (Designer dumps) | `formatspec(name=…)` or `formatspec(query=…)` | — (for `.mdo` / `.form` the typed `edt-mcp` API and `get_metadata_details` replace the XML specs) |
| Does the platform ship a mechanism for X (СЛАУ, crypto, data analysis, bus, bots, …)? | `docsearch` (capability description) → `docinfo` per found name (`AGENTS.md → A.7`) | `ssl_search` for a БСП-level solution |
| ITS standards (topic / prose) | `v8std_search` on `v8std-local` | `v8std_get_page` on a returned id for the full text |
| ITS standard behind a diagnostic (ACC/АПК, BSLLS, EDT, v8-code-style) | `v8std_explain_diagnostics` on `v8std-local` | `v8std_get_related` to widen context |

Native discovery tools (`Grep`, `Glob` / file search, directory listing, bulk `Read`) are absent from this table on purpose — they are not a first pick for any of these needs. Parameter names for each `edt-mcp` tool are **not** hard-coded here — consult the tool's live schema (`AGENTS.md → MCP Tool Calling → C.5`).

---

## Configurations with extensions

The base configuration and each extension are **separate EDT projects** (`list_projects`): pass the `projectName` of the contour you mean on every call and never treat a result from one contour as evidence about another. An object adopted into an extension shows its base object plus the extension additions — read both for the effective picture. Absence in an extension needs a synced model for that project (`resync_to_disk` / `revalidate_objects`); claims about a running infobase need evidence for that named target. Canon — `content/rules/extension-workspace.md`.

---

## The EDT workspace

`edt-mcp` is the project index and the live workspace at once, so the chain above already ranks it first; its literal, non-dialect-aware `search_in_code` ranks with `Grep` for identifiers, not with the structural tools. Reconcile the EDT model and the disk before trusting either — `content/rules/edt-workflow.md → Model vs disk — the synchronization rule`; tool choice — `edt-workflow.md → Search and navigation`.

---

## When `Grep` / `Glob` / `Read` are legitimately the right tool

This rule is a **bounded priority, not a prohibition** — the agent must always stay operational without MCP and must be free to search on its own after an MCP miss. Native tools are appropriate, with no need for an MCP attempt first, when:

- the project-index MCP server (`edt-mcp`) is **not exposed** in the current session — the whole chain collapses; work with native tools normally and state the unavailability once, in one line;
- an MCP result **looks wrong or stale** (contradicts known facts, predates fresh local edits) — verifying or overriding it against the disk state via `Grep` / `Read` is legitimate; after local edits the disk is the authority, not the index;
- the target is **outside the MCP index**: non-BSL / non-metadata files (`.md` documentation, `.json` / `.yaml` configs, slash-command sources, rule files, `openspec/` artifacts, deployment logs), text fixtures, sample payloads, generated reports under `handoffs/` / `dist/` / build output;
- a file you have already read in this session and are scanning for a literal string locally;
- reading a file whose path came from an MCP result (edit target, full context of a found fragment) — see also *Full-file `Read` — fragment first* above.

And after an MCP attempt that **missed** (tuned call + documented retry returned nothing relevant) — fall back immediately with the one-line "what was tried" note; no further MCP calls are owed to this rule.

In all remaining 1C project-source cases — follow the hard rule above.

---

## Response gate

Before delivering a result that involved any native discovery tool (`Grep` / `Glob` / file search / `Read`-scanning) on project source, include a short line in the response, e.g.:

> *Tried `search_in_code` for "..." (empty), `get_module_structure` for "..." (no match); fell back to `Grep` for the literal `<...>`.*

One or two sentences. No bullet list of every parameter tried.

---

## Success criteria

- ✅ The `edt-mcp` project-index path was attempted before any native discovery call (`Grep` / `Glob` / file search / directory listing / `Read`-scanning) on 1C project source — when `edt-mcp` is exposed.
- ✅ No "getting oriented" sweeps (source-tree globbing, bulk module reading) while `edt-mcp` is exposed.
- ✅ Fragment-level retrieval preferred over full-module `Read` when the need is one routine — outside the normal-work cases listed above.
- ✅ Each failed MCP call closed a concrete context gap before the next call (no blind chaining, no "just to be safe").
- ✅ After a missed MCP attempt — or with servers not exposed — fallback proceeded immediately; no ritual MCP calls made only to satisfy this rule.
- ✅ A typed error or a named missing-index warning was treated as the final answer of that lane; no sibling-tool probing of the same gap.
- ✅ Native-tool usage on project source is justified inline.
- ✅ No duplicated calls against unchanged state.
- ✅ A definitive "not found" conclusion is backed by freshness evidence (model synced with the disk); otherwise the MCP miss is reported as inconclusive and the documented fallback is used.
- ✅ For a base configuration with extensions, the `projectName` of the relevant contour was used on every call. Source / model conclusions were distinguished from evidence about a running infobase.
