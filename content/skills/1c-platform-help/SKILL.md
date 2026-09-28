---
name: 1c-platform-help
description: "External 1C knowledge — platform syntax reference, the platform-capability check before a custom specialized mechanism, БСП / SSL APIs, routed standards, file-format specs, ITS docs — via the docs, SSL and v8std MCP. Use only when such a fact affects the task."
argument-hint: "<name | capability description> [version]"
allowed-tools: mcp__1C-docs-mcp__docsearch, mcp__1C-docs-mcp__docinfo, mcp__1C-docs-mcp__standards, mcp__1C-docs-mcp__formatspec, mcp__1c-ssl-mcp__ssl_search, mcp__v8std-local__v8std_search, mcp__v8std-local__v8std_explain_diagnostics, mcp__v8std-local__v8std_explain_snippet, mcp__v8std-local__v8std_get_page, mcp__v8std-local__v8std_get_related, mcp__edt-mcp__get_platform_documentation
---

# 1c-platform-help — platform, БСП, standards, ITS

Conditional knowledge: call when the task depends on versioned platform behaviour, a reusable БСП API or standards compliance; never for prose cleanup. Model memory is not evidence of a platform feature's absence.

## Tools and exact arguments

| Need | Call | Arguments |
|---|---|---|
| Known type / method | `docinfo` | `name` (`"ТаблицаЗначений"`, `"Массив.Найти"`), optional `doc_type`, `scope`, `detail_level="compact"` |
| Feature by description | `docsearch` | `query` (Russian description of the capability), `top_k`, `scope="syntax" \| "docs" \| "all"` |
| Routed project standard | `standards` | `name="<rule stem>"` (`anti-patterns`, `dev-standards-architecture`); `query=` searches the collection; no args = catalogue — never `docsearch` for a standard |
| On-disk XML format | `formatspec` | `name=` / `query=` |
| БСП reusable API | `ssl_search` | `query` (exact symbol or Russian description), `limit=5`, `detail="compact"`; fetch one hit with `doc_id`, `detail="full"` |
| ITS standard by topic | `v8std_search` (`v8std-local`) | `query`; then `v8std_get_page` on the returned id for **every** document relied on |
| ITS clause behind a check id (ACC / АПК, BSLLS, EDT, v8-code-style) | `v8std_explain_diagnostics` | the check id; `v8std_get_related` widens context |
| A snippet against ITS standards | `v8std_explain_snippet` | the BSL / query snippet |
| Platform API pinned to the project's version | `get_platform_documentation` (`edt-mcp`) | `projectName`, type / global-function name |

There is no `corpus` argument on the docs server; `scope` never reaches `standards` / `formatspec`.

## Calls

```json
{"tool": "docinfo", "args": {"name": "ХранилищеЗначения", "detail_level": "compact"}}
{"tool": "docsearch", "args": {"query": "решение системы линейных уравнений", "scope": "all", "top_k": 5}}
{"tool": "standards", "args": {"name": "locks-and-transactions"}}
{"tool": "ssl_search", "args": {"query": "ОбщегоНазначения.ЗначениеРеквизитаОбъекта", "detail": "compact"}}
{"tool": "v8std_search", "args": {"query": "длительные операции"}}
```

## Platform-capability check (hard gate, `AGENTS.md → A.7`)

Before a custom implementation in a specialized domain (cryptography, СЛАУ, data analysis, collaboration system / bots, integration bus / queues, full-text search, regex, archives, geo, background work): `docsearch` by capability, 1–2 reformulations → `docinfo` for every exact name found → `ssl_search` where a БСП solution is plausible. Found and usable → build on it, custom code for glue only. Partial fit → `CONFUSION`. Rejecting a found mechanism needs doc-confirmed incompatibility, stated in the answer.

## Reading the answers

`outcome: "not_found"` on the docs server and `not_found` on SSL are finished answers, not retries; `outcome: "unavailable"` / `status: degraded` means the docs index is not built — report once, treat the dependent fact as unverified. `ambiguous_name` on SSL returns candidates — refine by signature. Documents are paged (`next_cursor`), a first page is not the whole standard. Typed answers map to actions by code — `content/rules/mcp-policy.md → C. Server answers → actions`.

Details: `content/skills/mcp-1c-tools/docs/1C-docs-mcp.md`, `content/skills/mcp-1c-tools/docs/1c-ssl-mcp.md`, `content/skills/mcp-1c-tools/docs/v8std.md`, `content/rules/help-corpus-retrieval.md`.
