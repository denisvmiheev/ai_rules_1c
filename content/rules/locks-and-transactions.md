---
description: Managed locks, transaction boundaries, lock ordering, deadlock prevention, shared / exclusive lock modes, monitoring via the technological log. Load when designing posting / multi-document operations, debugging lock conflicts, or extending an existing transactional path.
alwaysApply: false
category: quality
---

# Locks and Transactions — Design Rules

The 1C platform offers two locking subsystems (automatic / managed) and an implicit-transaction model around object writes. Most production lock incidents come from mixing the two, opening unintended transactions, or holding locks across user dialogs. This file is the canonical home for those rules.

> **Scope.** This file owns the design rules. The narrow case "set a lock before reading balances during posting" lives as a worked example in `platform-solutions.md §9 → "Managed locks and deadlock prevention"` — that section now points back here for the general theory.

<!-- help-mcp-router -->

## Where this standard lives

**The normative text of this file is not inlined here.** It is one document of the `1c-standards` collection on the Help MCP server (`1C-docs-mcp`):

```
standards(name="locks-and-transactions")     # this standard, entire - the normal call
standards(query="<what you need>")  # only when unsure which rule governs
```

`standards` is the tool for this collection. `docsearch` / `docinfo` serve the platform documentation and cannot reach it, and **no tool of this server takes a `corpus` argument**. Name resolution, paging, budget, and what to do when the server is not exposed - **`content/rules/help-corpus-retrieval.md`**.

**Retrieve before you apply.** Every section below is a heading with no body: acting on a section title without the text behind it is inventing the rule, not following it. Fetch the rule once by name rather than a query per section.

Pinned source, readable directly: <https://github.com/comol/ai_rules_1c/blob/410951e74fd3e6b7a763cf49757935b9a34d3f31/content/rules/locks-and-transactions.md>

## Sections

Every heading this file has always had, reproduced so that existing `locks-and-transactions.md` section references and anchor links still resolve - the same compatibility shape `dev-standards-core.md` uses. Each is a retrieval target, not a summary.

## 1. Lock mode of the configuration

Set the configuration property **«Режим управления блокировкой данных»** to `Управляемый` (Managed) for the whole configuration. Reasons:

- Mixed mode hides bugs — code written under automatic locking suddenly conflicts when the same data path is reached under managed mode.
- Managed mode is the only mode where lock granularity is under the developer's control (per-table / per-key, not per-row-page).
- Modern БСП subsystems assume managed mode.

Migrating an existing project from automatic to managed is a separate, multi-step task — out of scope here; use ITS "Перевод конфигурации на управляемый режим блокировки" — [std460](https://v8std.ru/std/460/) "Использование управляемого режима блокировки" (`v8std-local`, see `.claude/skills/mcp-1c-tools/docs/v8std.md`).

## 2. Transaction boundaries

### Implicit transactions

### Explicit transactions in calling code

### Forbidden inside transactions

## 3. Managed-lock primitives

### Modes

## 4. Lock ordering — the deadlock contract

## 5. Locking patterns

### Pattern: posting a document that touches several registers

### Pattern: mass operation across many documents

### Pattern: status update outside posting

## 6. Diagnosing lock conflicts and deadlocks

### Symptoms

### Diagnostic tools

## 7. Companion rules
