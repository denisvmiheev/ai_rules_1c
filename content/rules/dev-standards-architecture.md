---
description: Development standards — architecture patterns, extensions, platform standards, code smells
alwaysApply: false
category: development
---

# Development Standards — Architecture & Platform

<!-- help-mcp-router -->

## Where this standard lives

**The normative text of this file is not inlined here.** It is one document of the `1c-standards` collection on the Help MCP server (`1C-docs-mcp`):

```
standards(name="dev-standards-architecture")     # this standard, entire - the normal call
standards(query="<what you need>")  # only when unsure which rule governs
```

`standards` is the tool for this collection. `docsearch` / `docinfo` serve the platform documentation and cannot reach it, and **no tool of this server takes a `corpus` argument**. Name resolution, paging, budget, and what to do when the server is not exposed - **`content/rules/help-corpus-retrieval.md`**.

**Retrieve before you apply.** Every section below is a heading with no body: acting on a section title without the text behind it is inventing the rule, not following it. Fetch the rule once by name rather than a query per section.

Pinned source, readable directly: <https://github.com/comol/ai_rules_1c/blob/410951e74fd3e6b7a763cf49757935b9a34d3f31/content/rules/dev-standards-architecture.md>

## Sections

Every heading this file has always had, reproduced so that existing `dev-standards-architecture.md` section references and anchor links still resolve - the same compatibility shape `dev-standards-core.md` uses. Each is a retrieval target, not a summary.

## 1. Architecture Patterns

### Code Placement

### "Result-Structure" Pattern

### "Early Return" Pattern

### "Value Table Search" Pattern

### Event Subscriptions

### New Metadata Objects Placement

### Background Jobs

### Defensive Type Checking

### Safe Structure Property Access

### Collection Normalization

## 2. Extensions

### Modification Priority

### Extension Directives

### Placement Rules (when `{NEW_OBJECTS_IN} = main_configuration`)

### Forms in Extensions

## 3. Platform Standards

### Async and Modality

### Client-Server Interaction

### Security

### Error Handling

### Dates

### Queries
- Verify metadata attributes (existence, names, types) **before** writing a query — see `AGENTS.md → MCP Tool Calling` (rule #3 «verify before writing»; metadata-first via `get_metadata_details` / `get_metadata_objects`).
- Look for existing query examples before writing complex queries (`templatesearch`, `search_code`).
- Query text formatting — on a new line at the same indentation level as the variable declaration:

```bsl
Запрос = Новый Запрос;
Запрос.Текст =
"ВЫБРАТЬ
|	Контрагенты.Ссылка КАК Ссылка,
|	Контрагенты.ИНН КАК ИНН
|ИЗ
|	Справочник.Контрагенты КАК Контрагенты";
```

- Always use an intermediate variable for query results. Method chaining is **PROHIBITED**:
  - Correct: `РезультатЗапроса = Запрос.Выполнить();`
  - Incorrect: `Запрос.Выполнить().Выгрузить()` / `Запрос.Выполнить().Выбрать()`.
- Always use `КАК` aliases for query fields (e.g. `Контрагенты.ИНН КАК ИНН`).
- **Queries inside loops are PROHIBITED.** Use batch queries with temporary tables. See `anti-patterns.md §1` and "Batch Query with Temp Table" template.
- Use `Запрос.УстановитьПараметр()` instead of string concatenation — prevents SQL injection and improves plan caching.
- For complex data retrieval prefer batch queries with temporary tables over multiple separate queries.
- Temporary tables — prefixed with `ВТ_`.
- Choose the join type from the required result semantics. Use `ВНУТРЕННЕЕ СОЕДИНЕНИЕ` when
  rows without a match must be discarded. Use `ЛЕВОЕ СОЕДИНЕНИЕ` when source rows must be
  preserved, and handle `NULL` explicitly.
- When accessing registers — filter by dimensions first (in virtual table parameters, not in `ГДЕ`).
- Do not modify register movements directly — only via the posting mechanism.
- When a limited result set is needed — use `ПЕРВЫЕ N`.
- Index all fields that participate in filters/joins via metadata.
- **Temp tables used in a join or `ОБЪЕДИНИТЬ` must be indexed.** Any `ПОМЕСТИТЬ ВТ_*` whose table later participates in a `СОЕДИНЕНИЕ` (mandatory), an `ОБЪЕДИНИТЬ` dedup, or feeds a `В (ВЫБРАТЬ …)` filter over a large set — add `ИНДЕКСИРОВАТЬ ПО` on the join / dedup key fields. Index the **2–3 most selective fields**, not the full column list — enumerating every field bloats the index and does not improve the plan. See `query-optimization.md → Temporary Table Indexing` and `anti-patterns.md §5a`.
- **Pre-collect, index, then join — before heavy work.** Replace correlated / per-row subqueries (`ИСТИНА В (ВЫБРАТЬ ПЕРВЫЕ 1 …)` in `ГДЕ`, subquery per row) with a temp table collected once, indexed, and joined via `ВНУТРЕННЕЕ СОЕДИНЕНИЕ`. Likewise, when a join feeds a `СГРУППИРОВАТЬ ПО` over a large source — join into an indexed temp table first, group afterwards. See `anti-patterns.md §3a` and `query-optimization.md → Pre-collect and Index Before Join / Group`.
- `ОБЪЕДИНИТЬ` (without `ВСЕ`) already deduplicates the combined result — `РАЗЛИЧНЫЕ` inside its operands is redundant double work; when duplicates are impossible, prefer `ОБЪЕДИНИТЬ ВСЕ`. Similarly, do not combine `РАЗЛИЧНЫЕ` with `СГРУППИРОВАТЬ ПО` over the same fields.

### Cross-Platform Compatibility

### Platform Version Compatibility

## 4. Data Access — Reference Attribute Access

**Direct dot-notation access on references** (e.g. `Контрагент.ИНН`) loads the entire object from the database. **[Project rule — stricter than ITS standard — [std496](https://v8std.ru/std/496/) "Чтение отдельных реквизитов объекта из базы данных".]** Outside trivial single-call handlers, the rule is a hard ban; in simple, non-loop code with one or two attributes ITS allows it, but the project default is to use dedicated БСП methods.

Use these methods instead:

| Method | Purpose | Example |
|--------|---------|---------|
| `ОбщегоНазначения.ЗначениеРеквизитаОбъекта` | Single attribute from one ref | `ОбщегоНазначения.ЗначениеРеквизитаОбъекта(Контрагент, "ИНН")` |
| `ОбщегоНазначения.ЗначенияРеквизитовОбъекта` | Multiple attributes from one ref | `ОбщегоНазначения.ЗначенияРеквизитовОбъекта(Контрагент, "ИНН, Наименование")` |
| `ОбщегоНазначения.ЗначениеРеквизитаОбъектов` | Same attribute from multiple refs | `ОбщегоНазначения.ЗначениеРеквизитаОбъектов(Контрагенты, "ИНН")` |
| `ОбщегоНазначения.ЗначенияРеквизитовОбъектов` | Multiple attributes from multiple refs | `ОбщегоНазначения.ЗначенияРеквизитовОбъектов(Контрагенты, "ИНН, КПП")` |

### Caching and Batch Retrieval

## 5. Performance Headlines

## 6. Code Smells (see `anti-patterns` rule for the full catalog)
