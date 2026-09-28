---
name: 1c-impact
description: "Usages, dependencies, call chains and change impact of 1C objects and routines — callers, callees, references from BSL, forms, roles and subsystems, register movements — via edt-mcp. Mandatory before renames, removals, refactoring and public-contract changes (Gate 4)."
argument-hint: "<Kind.Name | routine> [callers | callees | outgoing]"
allowed-tools: mcp__edt-mcp__list_projects, mcp__edt-mcp__find_references, mcp__edt-mcp__get_method_call_hierarchy, mcp__edt-mcp__go_to_definition, mcp__edt-mcp__get_outgoing_structures, mcp__edt-mcp__get_metadata_details, mcp__edt-mcp__search_in_code
---

# 1c-impact — usages, call graph, change impact

Evidence for `content/rules/verification-gates.md → Gate 4` and for the pre-refactor analysis of `content/rules/tooling-playbooks.md → Refactoring`. Refactoring blind while `edt-mcp` is exposed is a defect; when it is not, follow Gate 4 graceful degradation.

## Tools

Every call takes `projectName` (`list_projects`); take exact parameter names from the live schema.

| Need | Call | Notes |
|---|---|---|
| Every usage of a metadata object: BSL (with lines), other metadata, forms, roles, subsystems | `find_references` | What breaks if the object is renamed, removed or retyped; also the type-reference check before removing a defined type |
| Who calls a routine / what a routine calls | `get_method_call_hierarchy` | `callers` / `callees`; `outgoing` aggregates a module's distinct call targets; resolves ru/en spellings |
| Where a symbol is defined | `go_to_definition` | The inverse of `find_references` |
| Informal `Структура` contracts between modules | `get_outgoing_structures` | Literal top-level keys of the structure passed as the first argument of each outgoing call |
| Documents writing movements into a register | `find_references` on the register FQN, then `search_in_code` for `Движения.<Регистр>` | Recorder movements are declared in metadata; direct BSL writers need the text search |
| Extension changes to a base object | `get_metadata_details` on the extension project's FQN | Extensions are separate EDT projects: pass each contour's `projectName` |

## Rules

- **Single-hop by design.** There is no transitive graph engine: for multi-hop impact walk the hierarchy level by level and say so in the delivery evidence (`content/skills/mcp-1c-tools/docs/edt-mcp.md → Coverage gaps`).
- Callers of a public `Экспорт` routine are a promotion trigger (`content/rules/verification-policy.md`); the caller list belongs in the delivery evidence.
- A reference list from a stale model understates impact: after edits made outside EDT run `resync_to_disk` / `revalidate_objects` first.
- `search_in_code` is textual and not dialect aware — it complements, never replaces, `find_references` / `get_method_call_hierarchy`.
- Typed answers map to actions by code — `content/rules/mcp-policy.md → C. Server answers → actions`.
