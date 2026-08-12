# v8std-local — tool catalog

Read-only mirror of [v8std.ru](https://v8std.ru/) — ИТС "Стандарты и методики разработки конфигураций для платформы 1С:Предприятие 8". This fork's default server set does not run the upstream `1c-code-check-mcp` server (whose `its_help`/`fetch_its` were the original ITS-lookup path); `v8std-local` is the fork owner's replacement, and it goes further: every standard (`stdNNN`) is cross-linked to the analyzer diagnostics that enforce it (ACC/АПК, BSLLS, EDT native checks, v8-code-style), which `its_help`/`fetch_its` did not do.

> Load this file only if the `v8std-local` server is actually available in the current session. If a project still runs `1c-code-check-mcp`, `its_help` → `fetch_its` remains a valid alternate path for ITS lookups.

## What it is and is not

- **Read-only knowledge source.** No analyzer run, no project inspection, no code changes. It answers "what does the standard say" and "which diagnostic enforces it" — not "does my code comply" (that is `get_project_errors` / `get_check_description` on `edt-mcp`).
- **Complements, does not replace, `1c-platform-mcp`.** `docsearch`/`docinfo` (platform-mcp) answer platform-API questions ("does `Массив` have a `Найти` method"). `v8std-local` answers methodology / standards questions ("should this common module be named with `КлиентСервер` postfix", "which ITS standard bans `?()`"). Use both when a review or design decision needs both angles.

## Tools

| Tool | Purpose | When to use |
|---|---|---|
| **v8std_search** | Hybrid (BM25 + semantic) search over standards, diagnostics, patterns, service pages. Accepts natural Russian phrases, `stdNNN`, or a diagnostic name. | Discovery when no exact id is known — arbitrary prose query, a topic ("модальные окна", "именование общих модулей"). |
| **v8std_explain_diagnostics** | Batch lookup: analyzer diagnostic codes → grouped standard clauses. Accepts `acc 1245`, `АПК:361`, `bslls:AssignAliasFieldsInQuery`, `v8cs:*`. | You have one or more diagnostic codes (from `get_project_errors` / `get_check_description` on `edt-mcp`, or from a linter run) and need the ITS rationale behind them. |
| **v8std_explain_snippet** | Short BSL/SDBL fragment → applicable standards + likely diagnostics with confidence. | You have a code fragment (not prose) and want to know which standards/diagnostics apply — e.g. `ВЫБРАТЬ РАЗРЕШЕННЫЕ`, `ОткрытьФормуМодально`, `Предупреждение`. |
| **v8std_get_page** | Full clean Markdown of a standard/diagnostic/pattern page by id, alias, or URL. | Always the second call after `search` / `explain_diagnostics` / `explain_snippet` / `get_related` return an id — read the authoritative text before citing or acting on a rule. |
| **v8std_get_related** | From a known standard/diagnostic id, list related standards/diagnostics/checks. | Building the full rule context around a finding — "what else is connected to this standard" — for code review or remediation planning. |

## Query flow

```
unknown topic ──► v8std_search(query)               ─┐
diagnostic code ─► v8std_explain_diagnostics(codes)   ├─► ids ─► v8std_get_page(id)  (full text)
code fragment ───► v8std_explain_snippet(snippet)     ┘              │
                                                                      ▼
                                                          v8std_get_related(id)  (widen context)
```

`v8std_search` / `v8std_explain_diagnostics` / `v8std_explain_snippet` are **discovery** — they return ids, titles, URLs, short previews. `v8std_get_page` is the only tool that returns the full rule text; do not cite or act on a standard from a search snippet alone — read the page.

## Notes

- **Diagnostic-code format for `v8std_explain_diagnostics`.** Loose matching confirmed: `"bslls:UsingModalWindows"`, `"acc 1245"` (space, no colon) both resolved correctly. Prefer the exact id shape returned by `get_project_errors` / `get_check_description` (`edt-mcp`) when chaining from a live finding.
- **`v8std_explain_snippet` is not for prose.** A phrase like "модальные окна" belongs to `v8std_search`; a code token or call (`ОткрытьФормуМодально`, `Вопрос(...)`) belongs to `v8std_explain_snippet`.
- **Every standard page carries `source_urls`** back to `its.1c.ru/db/v8std#content:<id>` — cite that when the delivery report or a code-review finding needs an authoritative external reference.
- **`body_limit` on `v8std_get_page`** (default 12000) truncates long pages — for a page with many numbered clauses, call again with a higher `body_limit` if the truncated tail matters, rather than guessing the rest.

## Typical uses in a project

- **Code review (`1c-code-reviewer`).** After `get_project_errors` returns findings with check ids, batch them through `v8std_explain_diagnostics` to attach the ITS rationale to each finding instead of citing the check id bare.
- **Error fixing (`1c-error-fixer`).** A BSLLS / АПК warning's fix is sometimes non-obvious from the message alone — `v8std_explain_diagnostics` → `v8std_get_page` gives the standard's worked example.
- **Architecture / design (`1c-architect`, `1c-arch-reviewer`).** When a design decision needs to cite a project convention against the ITS baseline (e.g. "[Project rule — stricter than ITS standard]" markers in `dev-standards-code-style.md` / `anti-patterns.md`) — verify the actual ITS wording via `v8std_search` / `v8std_get_page` before stating the delta, instead of paraphrasing from memory.
- **Spec authoring (`1c-analytic`, OpenSpec artifacts).** Any `design.md` / `proposal.md` claim that cites "ITS standard" as justification must now be MCP-verified per `sdd-integrations.md` — no more "ITS standards — coverage gap" exemption.
- **Performance review (`1c-performance-optimizer`).** `v8std_search` for query / loop anti-pattern topics surfaces the matching ITS standard alongside the project's own `anti-patterns.md` catalog.
