---
description: Systematic 4-phase debugging methodology adapted for 1C (reproduce → hypothesize → experiment → fix), with a fast path for directly evidenced root causes (DEBUG_FAST_PATH in .dev.env)
alwaysApply: false
category: quality
---

# Systematic Debugging — 1C Adaptation

**When to load this file:** any task that involves diagnosing a bug, runtime error, regression, performance regression, or unexpected behavior — whether the parent agent is debugging directly or delegating to the `1c-error-fixer` / `1c-performance-optimizer` subagent.

**Goal:** replace ad-hoc trial-and-error with a structured root-cause loop. Skipping a phase is a defect — unless the bug qualifies for the **fast path** below, which is a documented shortcut, not a skipped phase.

The methodology is adapted from the `systematic-debugging` skill of [obra/superpowers](https://github.com/obra/superpowers) and combined with 1C platform mechanics (debugger, `ЖурналРегистрации`, `ОтчетПоЖурналуРегистрации`, `ПоказатьЗначение`, `СообщитьПользователю`, `Replay` of background jobs, technological log).

<!-- help-mcp-router -->

## Where this standard lives

**The normative text of this file is not inlined here.** It is one document of the `1c-standards` collection on the Help MCP server (`1C-docs-mcp`):

```
standards(name="systematic-debugging")     # this standard, entire - the normal call
standards(query="<what you need>")  # only when unsure which rule governs
```

`standards` is the tool for this collection. `docsearch` / `docinfo` serve the platform documentation and cannot reach it, and **no tool of this server takes a `corpus` argument**. Name resolution, paging, budget, and what to do when the server is not exposed - **`content/rules/help-corpus-retrieval.md`**.

**Retrieve before you apply.** Every section below is a heading with no body: acting on a section title without the text behind it is inventing the rule, not following it. Fetch the rule once by name rather than a query per section.

Pinned source, readable directly: <https://github.com/comol/ai_rules_1c/blob/410951e74fd3e6b7a763cf49757935b9a34d3f31/content/rules/systematic-debugging.md>

## Sections

Every heading this file has always had, reproduced so that existing `systematic-debugging.md` section references and anchor links still resolve - the same compatibility shape `dev-standards-core.md` uses. Each is a retrieval target, not a summary.

## Core principle

## Fast path — for directly evidenced root causes

## The four phases (full loop)

### Phase 1 — Reproduce

Goal: a deterministic, minimal reproduction case in a controlled environment.

Required outputs of this phase — **in the scope relevant to the bug** (a pure code-level logic error rarely needs the role set or locale; a rights / RLS / session bug always does):

- exact infobase (file or SQL — record the connection string), platform version, configuration / extension versions;
- exact user, role set, session parameters, locale;
- exact input data (document number / catalog reference / register record key) — copy or anonymize, do not paraphrase;
- exact reproduction steps (UI clicks, form, command, or API call);
- exact observed result (error message, stack trace, wrong value, slow timing);
- exact expected result.

Tools to use:

- **`vcloggetlasterror`** (`1c-data-mcp`, when the server is exposed in the session) — pull the most recent error from `ЖурналРегистрации` of the live IB without leaving the agent: timestamp, event, affected metadata object, data presentation, full description. Run it immediately after the user's repro attempt to confirm an error landed in the log and on what object. Limitation: only the single most recent record with `УровеньЖурналаРегистрации.Ошибка` in the last 24 h. Wider filters / older records → fall back to the Configurator's `ОтчетПоЖурналуРегистрации` or to a custom `ВыгрузитьЖурналРегистрации` wrapped in `vcexecutecode`.
- **Configurator → Debug** (`Отладка → Подключиться`) to attach to the running session, capture the call stack, inspect locals.
- **`ЖурналРегистрации`** filter by date range / user / event / metadata to find the failing call. For high-volume errors prefer `ВыгрузитьЖурналРегистрации` to a `ТаблицаЗначений` for offline analysis.
- **Technological log** (`logcfg.xml`) for platform-level events (DBMS errors, deadlocks, lock conflicts, timeouts) when the application log is not enough.
- **`ОбработкаПроведения`** + the `Replay` mechanism for documents — re-post the failing document under the debugger.
- **MCP**: `search_in_code` to find the exact procedure that printed the error message, `get_module_structure` to locate it by name and see its surroundings.

If you cannot reproduce, **stop**. Ask the user for missing input data, screenshots, the exact step sequence, or a copy of the infobase. Do not guess.

### Phase 2 — Hypothesize

Goal: a small set of falsifiable hypotheses about the root cause, ranked by likelihood.

For each hypothesis state:

- **Statement** — exactly which code path / metadata / data state causes the symptom.
- **Falsifying experiment** — what would prove the hypothesis wrong (a log line that should appear but doesn't; a value that should be `Неопределено`; a query that should return an empty result; a lock that should not be acquired).
- **Cost of the experiment** — cheap (read-only query / `ПоказатьЗначение`) vs. expensive (rebuild infobase, reload data).

Tools to use:

- **`get_method_call_hierarchy`** to map all call paths that reach the failing routine, and the routines it calls.
- **`find_references`** for object-level dependencies of the owning module / metadata object (registers, referenced objects, movements), not as a routine-call query.
- **`docinfo`** to verify that a built-in function actually does what you assume — many bugs are platform-version-dependent (`ТекущаяДатаСеанса` vs `ТекущаяДата`, `НайтиПоНаименованию` collation, `ПолучитьСтруктуруХраненияБазыДанных` differences across versions).
- **`v8std_search`** on `v8std-local` to verify the documented behaviour of the platform mechanism you suspect (or `its_help` → `fetch_its` on `1c-code-check-mcp` if that server is used instead).
- **`ask_1c_ai`** as a hint generator (treat as a draft — do not let an AI hint replace the falsifying experiment).

Produce at least 2 hypotheses. In the full loop a single hypothesis is anchoring bias — challenge it with a competing one even when it feels right (a bug that genuinely admits only one explanation belongs on the fast path, not here).

### Phase 3 — Experiment

### Phase 4 — Fix

## Anti-patterns

## Process flow

## Companion rules
