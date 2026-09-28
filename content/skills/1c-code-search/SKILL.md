---
name: 1c-code-search
description: "Find BSL code in a 1C project — routines by name, code by exact literal or identifier, module layout, members of a context — through edt-mcp before any Grep. Use whenever a task needs to locate or read a fragment of BSL it does not yet have a path for."
argument-hint: "<what to find> [literal | symbol]"
allowed-tools: mcp__edt-mcp__list_projects, mcp__edt-mcp__list_modules, mcp__edt-mcp__search_in_code, mcp__edt-mcp__get_module_structure, mcp__edt-mcp__read_method_source, mcp__edt-mcp__read_module_source, mcp__edt-mcp__go_to_definition, mcp__edt-mcp__get_symbol_info, mcp__edt-mcp__get_content_assist
---

# 1c-code-search — locate BSL code

Project-source search is MCP-first: `edt-mcp` model-based navigation → `search_in_code` literal / regex retry → scoped native `Grep` after a bounded miss with a one-line fallback note. `edt-mcp` not exposed means native search immediately. `content/rules/mcp-first-search.md` owns retrieval, freshness and the native exceptions.

## Tools

Every call takes `projectName` — confirm it once with `list_projects`. Read the live schema (`get_tool_guide('<toolName>')` when it is not enough) before a parameter-rich call; do not invent parameter names.

| Need | Call | Key parameters |
|---|---|---|
| Module paths of an object / type | `list_modules` | `metadataType`, `objectName`, `nameFilter` |
| Module layout before editing | `get_module_structure` | module path (`src/`-relative); `responseFormat=detailed` for parameters and doc-comments |
| One routine body | `read_method_source` | module path, method name |
| A line range or the whole module (edit target, small file) | `read_module_source` | module path, `startLine` / `endLine`; returns `contentHash` |
| Where a symbol is defined | `go_to_definition` | `Module.Method`, or a bare method + `modulePath`, or a metadata FQN |
| What an identifier is at a position | `get_symbol_info` | module path, line, column |
| Members of a context / events of a form item | `get_content_assist` | module path, line, column; `member_type="methods" \| "properties" \| "events"` |
| Exact identifier, literal, query fragment, error text | `search_in_code` | query, `isRegex`, `fileMask`, `metadataType`, `outputMode=files` |

`search_in_code` is a literal / regex textual sweep: **not** semantic and **not** ru/en dialect aware — a symbol spelled the other way does not match. For identifiers prefer `go_to_definition` / `find_references` (`1c-impact`) / `get_method_call_hierarchy`.

## Miss handling

1. Reformulate once when useful: a narrower `fileMask` / `metadataType`, the regex form, or a structural tool instead of text.
2. A miss right after local edits may be a stale model: `resync_to_disk` or `revalidate_objects` first (`content/rules/edt-workflow.md → Model vs disk`), not more searches.
3. A typed server answer maps to an action by code — `content/rules/mcp-policy.md → C. Server answers → actions`; never probe the same gap through sibling tools.
4. Then native `Grep` / `Read`, with the note. Reading a file already located here is normal work.

Rare modes and the full tool catalog: `content/skills/mcp-1c-tools/docs/edt-mcp.md`.
