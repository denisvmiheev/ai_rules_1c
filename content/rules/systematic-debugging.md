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

> **Retrieve through MCP only.** Call `standards(name="systematic-debugging")` on `1C-docs-mcp` before applying this standard. Retrieval, paging, and unavailable-server policy: `content/rules/help-corpus-retrieval.md`. Headings below are retrieval targets, not summaries.

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

- **`get_event_log`** (`edt-mcp`) — reads the infobase event log **offline** (no running session): timestamp, event, affected metadata object, data presentation, full description; filter by period, user, severity, event / comment / metadata substring, session. Run it immediately after the user's repro attempt to confirm an error landed in the log and on what object. It returns infobase PII — quote only what the diagnosis needs. When the log files are unreachable, fall back to the Configurator's `ОтчетПоЖурналуРегистрации`.
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
- **`v8std_search`** on `v8std-local` to verify the documented behaviour of the platform mechanism you suspect (`v8std_get_page` for the full article).
- **`ask_workmate`** as a hint generator (treat as a draft — do not let an AI hint replace the falsifying experiment).

Produce at least 2 hypotheses. In the full loop a single hypothesis is anchoring bias — challenge it with a competing one even when it feels right (a bug that genuinely admits only one explanation belongs on the fast path, not here).

### Phase 3 — Experiment

### Phase 4 — Fix

## Anti-patterns

## Process flow

## Companion rules
