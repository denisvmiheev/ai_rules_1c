---
name: mcp-1c-tools
description: "Router for non-memory 1C MCP operations: select the server (edt-mcp / 1C-docs-mcp / 1c-ssl-mcp / 1c-templates-fastcode / v8std-local), operation skill with exact arguments, and fallback. Load before choosing a 1C tool; memory-only work enters through project-memory.md."
---

# MCP tools for 1C — router

Apply `content/rules/mcp-policy.md → Tool availability` before routing: `.dev.env` `TOOL_*` policy plus callable tools and verified scope. Config entries alone prove nothing. The same policy owns obligations, budgets and typed-error recovery. Search: `content/rules/mcp-first-search.md`; task sequences: `content/rules/tooling-playbooks.md`.

**Project scope is part of the call.** Nearly every `edt-mcp` tool requires `projectName` — confirm it with `list_projects` and pass it explicitly; never accept a default or a foreign workspace project as the current one. Module paths are `src/`-relative; metadata is addressed by FQN (`Catalog.Products`, `Catalog.Products.Attribute.Weight`, `Role.FullAccess`), English or Russian type tokens. A base configuration and each extension are separate EDT projects — pass the `projectName` of the contour you mean. Reuse a valid mapping until the workspace or server context changes.

## Need → operation skill

Load the skill for the operation, not this whole catalogue. Each skill lists the tools of its operation and how to call them; for a parameter-rich `edt-mcp` call read the live schema (`get_tool_guide('<toolName>')` when the schema is not enough).

| Need | Skill | Servers |
|---|---|---|
| Locate or read BSL code, module layout, members of a context | `content/skills/1c-code-search/SKILL.md` | edt |
| Facts about a metadata object: passport, attributes, tabular-part columns, objects by category or description | `content/skills/1c-meta-info/SKILL.md` | edt |
| Usages, call chains, downstream impact, register writers, extension changes | `content/skills/1c-impact/SKILL.md` | edt |
| Read forms, form structure, rendered layout, format specs before a form change | `content/skills/1c-form-inspect/SKILL.md` | edt, docs |
| Validate changed BSL, queries and metadata (Gates 1–3, 5) | `content/skills/1c-validate/SKILL.md` | edt, v8std |
| Platform reference, capability check, БСП API, routed standards, ITS, format specs | `content/skills/1c-platform-help/SKILL.md` | docs, ssl, v8std |
| Templates as the base | `content/skills/1c-templates-memory/SKILL.md` | templates |
| Memory recall / save (independent entry point) | `content/rules/project-memory.md` | cognee, openviking, templates |
| What only the running infobase knows: last event-log error, query / fragment on live data | `content/skills/1c-live-ib/SKILL.md` | edt |
| Check behaviour in the 1C interface (thin client): forms, fields, tables, commands, messages | `content/skills/1c-qa-testing/SKILL.md` + `content/rules/qa-testclient.md` | qa |
| Create / edit / remove metadata, forms, roles, DCS, MXL, infobases | `content/skills/1c-metadata-manage/SKILL.md` | edt typed API / scripts |
| Live 1C:EDT workspace: source format, model↔disk, DB update | `docs/edt-mcp.md`, `content/rules/edt-workflow.md` | edt |

## Server catalog

The operation table above is sufficient for ordinary routing. Read [server-catalog.md](docs/server-catalog.md) only to identify an unfamiliar server/alias, find a server-specific reference, or select an optional pre-alpha integration. Do not load the whole catalog before a known operation.

## Fallback chain

**Project source** (code, metadata, usages, forms, file locations): `edt-mcp` model-based search and navigation → `search_in_code` as a literal / regex retry → scoped native `Grep` / `Glob` / `Read` after a bounded miss, with a one-line fallback note. `edt-mcp` not exposed means native search immediately. Owner: `content/rules/mcp-first-search.md`.

**External knowledge** has no interchangeable fallback chain. Select only the relevant eligible provider. Missing templates: disclose no template check; memory: `content/rules/project-memory.md`; routed standards: `content/rules/help-corpus-retrieval.md`; validators: `content/rules/verification-gates.md`. An unavailable source never proves API absence or a passed check; block dependent design when authoritative facts are missing.

Open `docs/<server>.md` once per server per session and only for a mode or response shape the operation skill does not cover; the environment descriptor wins over any document here.
