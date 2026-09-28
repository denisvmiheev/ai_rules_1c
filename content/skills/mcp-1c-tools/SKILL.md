---
name: mcp-1c-tools
description: "Router for the 1C MCP ecosystem — which server answers which need, which operation skill holds the exact calls and parameter names, and the fallback chain. Load before selecting any edt-mcp / 1C-docs-mcp / 1c-ssl-mcp / 1c-templates-fastcode / v8std-local tool."
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
| Templates as the base, project memory recall / save | `content/skills/1c-templates-memory/SKILL.md` | templates, cognee, openviking |
| What only the running infobase knows: last event-log error, query / fragment on live data | `content/skills/1c-live-ib/SKILL.md` | edt |
| Create / edit / remove metadata, forms, roles, DCS, MXL, infobases | `content/skills/1c-metadata-manage/SKILL.md` | edt typed API / scripts |
| Live 1C:EDT workspace: source format, model↔disk, DB update | `docs/edt-mcp.md`, `content/rules/edt-workflow.md` | edt |

## Server catalog

| Server id | Purpose | Details |
|---|---|---|
| `edt-mcp` | Live 1C:EDT workspace: metadata discovery and typed mutation, BSL read/write, search, references, call hierarchy, validation markers, forms rendering, event log, DB update, debug / YAXUnit / profiling | `docs/edt-mcp.md` |
| `1C-docs-mcp` | Platform reference (`docsearch`, `docinfo`), routed `standards`, `formatspec` | `docs/1C-docs-mcp.md` |
| `1c-ssl-mcp` | БСП / SSL API search | `docs/1c-ssl-mcp.md` |
| `1c-templates-fastcode` | Code templates, project vector memory (`remember` / `recall`) | `docs/1c-templates-fastcode.md` |
| `v8std-local` | ITS «Стандарты разработки» mirror: standards search, diagnostic → standard mapping (ACC/АПК, BSLLS, EDT, v8-code-style), snippet classification | `docs/v8std.md` |
| `cognee`, `openviking` *(optional)* | Memory providers | `docs/memory-providers.md` |

The classic multi-server docker distribution (graph, code-metadata, syntax-checker, code-check, data servers) is **not** part of this ruleset's default set; `docs/edt-mcp.md → Remap from the classic docker distribution` maps its tools to `edt-mcp`.

### Optional pre-alpha servers

These are experimental projects, separate from the main servers above and not required by normal development gates. Client aliases vary; use the tools actually exposed in this session. Read their catalog only for a task that needs them; do not install, start a client, replay UI actions or write conversion files merely to check availability.

| Project / runtime server name | Purpose | Details |
|---|---|---|
| `MCP_Test` / `1C Visual UI Test` | Testing knowledge base, scenario preparation, test-client processes and UI replay | `docs/mcp-test.md` |
| `MCP_ConversionData20` / `1C Конвертация данных 2.0 — разработка правил обмена` | Metadata mapping and conversion-rule authoring/validation/export | `docs/mcp-conversion-data20.md` |

## Fallback chain

**Project source** (code, metadata, usages, forms, file locations): `edt-mcp` model-based search and navigation → `search_in_code` as a literal / regex retry → scoped native `Grep` / `Glob` / `Read` after a bounded miss, with a one-line fallback note. `edt-mcp` not exposed means native search immediately. Owner: `content/rules/mcp-first-search.md`.

**External knowledge** has no interchangeable fallback chain. Select only the relevant eligible provider. Missing templates: disclose no template check; memory: `content/rules/project-memory.md`; routed standards: `content/rules/help-corpus-retrieval.md`; validators: `content/rules/verification-gates.md`. An unavailable source never proves API absence or a passed check; block dependent design when authoritative facts are missing.

Open `docs/<server>.md` once per server per session and only for a mode or response shape the operation skill does not cover; the environment descriptor wins over any document here.
