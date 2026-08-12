---
description: Verification execution gates for BSL and metadata — evidence reuse, syntax, logic, style, impact analysis, and XML validation
alwaysApply: false
category: quality
---

# Verification Gates — BSL, Impact, and Metadata

**When to load this file:** before validating or declaring a BSL / metadata change done. Determine depth and promotion triggers first via `verification-policy.md`.

Delivery-only soft gates and the final report contract live in `verification-delivery.md`.

> **Default tool mapping — `edt-mcp`.** This fork's default project server is `edt-mcp` (1C:EDT plugin), not upstream's classic docker distribution. The gates below are written against `edt-mcp`'s validator model; a project running the classic distribution instead maps `syntaxcheck` → Gate 1, `check_1c_code` → Gate 2, `review_1c_code` → Gate 3, `verify_xml` → Gate 5, and keeps Gate 3a / Gate 4's graph-metadata tools as originally documented. See `content/skills/mcp-1c-tools/docs/edt-mcp.md` for the full remap.

## Gate execution and evidence reuse

A gate is a requirement for the current artifact state, not a request to call the same tool
again. The agent that makes the final edit to an artifact owns its applicable validator run and
records the artifact path, validator result and run count in the handoff / implementation report.

The parent closing gate MUST reuse that evidence when it was produced after the latest edit.
It runs only missing or stale gates and MUST NOT repeat a validator against unchanged content
(`AGENTS.md → MCP Tool Calling → C.2`). Any later edit invalidates the affected validator
evidence; the final editor becomes the new owner. The same rule applies to `revalidate_objects` (or
`verify_xml` on the classic distribution) and impact-analysis evidence.


## Hard gates — run on every full-cycle change

You MUST run all five gates in order. Each gate has an explicit pass / fail criterion and an explicit retry budget. When a required validator is not exposed in the current session, follow the graceful-degradation subsections (after Gate 3 and inside Gate 4) instead of silently skipping. **Gate 3a is conditional** — it runs only when its own trigger fires and its server is exposed; it never replaces Gates 1–3.

The gate descriptions below state the `full` behaviour — the strictest level, and the one a promotion-trigger path always gets. The project default is `standard`: same three validators, one mandatory confirmation after a blocking fix instead of an open-ended retry budget. When `VERIFICATION_DEPTH` is `standard` or `lite`, Gates 1–3 are modulated per `verification-policy.md → "Verification depth levels"` — but a full-cycle change on any promotion-trigger path always runs the complete chain regardless of the level (the safety floor).

> **Validator model on `edt-mcp`.** EDT owns one marker store; Gates 1–3 below are three *severity slices* of it, not three different engines. The standard sequence after an edit is `revalidate_objects(objects=[FQNs])` (refresh the markers for what you touched — it re-reads the disk first, so it also picks up edits made outside EDT) → `get_project_errors(...)` filtered per gate → `get_problem_summary` as the confirming pass. `write_module_source` additionally runs a **BSL syntax check before writing**, so a successful write is already a partial Gate 1 pass; `validate_query` checks query text on its own.

### Gate 1 — Syntax (`write_module_source` pre-write check → `revalidate_objects` → `get_project_errors severity=ERRORS`; `validate_query` for query text)

- Run on every touched `.bsl` module. No exceptions.
- When the module was written through `write_module_source`, its pre-write syntax check counts as the first half of this gate; still refresh and read markers, because a syntactically valid module can break references.
- Any new / changed query text (module code, DCS data set, dynamic list) — `validate_query` (`dcsMode=true` for СКД).
- Pass criterion: zero `ERRORS` items for the touched objects.
- Retry budget: canon — `AGENTS.md → MCP Tool Calling → B.1`. An `ERRORS` item is blocking: after fixing it, obtain a clean confirming run on the changed module. Under `full`, allow 3 calls total; under `standard`, 2. If the limit is exhausted without a clean pass on the latest state, Gate 1 fails. Gates 2 and 3 use the same policy with their own blocking severities.

### Gate 2 — Logic & performance (`get_project_errors` severity `BLOCKER` / `CRITICAL` / `MAJOR`, + `get_check_description`)

- Run on every touched module, scoped with `objects=[FQNs]`. Always after Gate 1 passes — reading checks on syntax-broken code wastes the pass.
- These severities carry EDT's substantive checks: query defects (`ql-*`, e.g. `ql-temp-table-index`), transaction handling (`begin-transaction`), platform-standard violations. Use `get_check_description(checkId)` to read what a finding actually means before fixing or dismissing it — the check id and its short UID both come from the `get_project_errors` rows.
- Pass criterion: no `BLOCKER` / `CRITICAL` item; a `MAJOR` item introduced by your change must be fixed or explicitly justified.
- Pre-existing findings outside your scope — leave alone (Surgical Changes) and, when relevant, list them as follow-ups.
- **Coverage gap.** There is no AI per-method analysis on `edt-mcp` (no `check_1c_code`-style AI pass). EDT checks are rule-based, so this gate does not catch reasoning-level defects on its own — pair it with the internal review checklist from `dev-standards-code-style.md §8` on any transactional / public-API / metadata-bearing change.

### Gate 3 — Style & standards (`get_project_errors` severity `MINOR` / `TRIVIAL` → `get_problem_summary`)

- Run on every touched module after Gate 2 passes; close with `get_problem_summary` as the confirming totals pass.
- Pass criterion: no severity totals worse than the pre-change baseline for the touched objects.
- `MINOR` / `TRIVIAL` items: same triage rule as Gate 2 — fix what your change introduced, leave pre-existing noise.
- For specific warnings that are intentional and justified: add a `//BSLLS:<rule>` suppression with a 1-line explanation, per `dev-standards-code-style.md → "Formatting"`. Blanket suppressions without justification are forbidden.
- `get_check_description` requires a check-descriptions folder configured in EDT's MCP preferences; if it returns a configuration error, `get_server_status` confirms the flag — record the finding without the explanation rather than dropping it.
- For a finding whose rationale is unclear from `get_check_description` alone, or when a review explicitly needs the ITS citation: `v8std_explain_diagnostics` on `v8std-local` (see `content/skills/mcp-1c-tools/docs/v8std.md`) resolves the check id to its standard clause. On a project running the classic distribution's `1c-code-check-mcp` instead, `its_help` → `fetch_its` is the alternate path.

### Graceful degradation for Gates 1–3 — when a validator is not exposed

Gates 1–3 are mandatory only when the corresponding validator is exposed in the current session (`AGENTS.md → MCP Tool Calling → A.1`: a server counts as available only when its tools are visible in the tool schema). When a validator is missing, do **not** silently skip its gate:

1. Record the fact in the delivery summary under **Risks** as a fixed line: *"Gate N skipped — `<tool>` (`<server>`) not exposed in this session."*
2. Compensate with what is available: for Gate 1 — a careful manual syntax review of every touched module (paired keywords, directives, parameter lists); for Gates 2–3 — the internal review checklist from `dev-standards-code-style.md §8` (style, readability, correctness, edge cases, security, concurrency / locks / transactions).
3. Delivery is not blocked, but a transactional / metadata / public-API change that went through without Gate 2 must be flagged as needing a follow-up validation run in a session where the server is exposed.

Skipping a gate without recording it under Risks is a defect — the same rule as Gate 4's graceful degradation below.

### Gate 3a — Live-IB smoke check (conditional, classic distribution's `1c-data-mcp` only)

Gates 1–3 are static: they confirm that the code parses, follows standards, and has no detectable logic smell. They cannot confirm that a **query actually resolves against this configuration's metadata**. Gate 3a closes exactly that gap, and only that gap — it is not part of this fork's `edt-mcp` default (see the coverage-gaps note in `content/skills/mcp-1c-tools/docs/edt-mcp.md`: `edt-mcp`'s closest equivalent is a heavier-weight `debug_launch` → `evaluate_expression` session, not a lightweight smoke check), so it only fires on a project that also installed the classic `1c-data-mcp`.

**Triggers — run when all of the following hold:**

1. The change authored or modified 1C **query text** (module code, DCS scheme, dynamic list) **or** a self-contained BSL function with no side effects whose result the static validators cannot confirm.
2. `1c-data-mcp` is exposed in the current session (`validatequery` / `vcexecutecode` visible in the tool schema).
3. The connected infobase is a development / test base. **On a production infobase this gate is not run** — record the skip and move on.

**Execution:**

- **Query text → `validatequery`.** Pass criterion: `"нет ошибок"`. This parses the query and resolves parameters; it does **not** execute it, does not verify that tables / fields exist, and does not evaluate RLS.
- **Pure function → `vcexecutecode`** with a **read-only** fragment. Pass criterion: `"ошибок нет"`, or the expected value returned via `Результат`.
- **Mutations are out of scope for this gate.** No `Записать()` / `Удалить()` / `НачалоТранзакции` / register movements — the read-only discipline and consent rules apply unchanged. If confirming the change requires a mutation, that is a task for `1c-tester` against a test base, not for this gate.
- **Budget:** one call per artifact. Re-run only after the artifact changed — the no-change-repeat rule (`AGENTS.md → MCP Tool Calling → C.2`) applies.

**Failure is blocking for the artifact,** the same as a Gate 1 `ERRORS` finding: fix the query / fragment and re-run once against the changed state.

**When a trigger fired but the gate could not run** (server not exposed, or a production IB), record one line in the delivery summary under **Risks**: *"Gate 3a not run — `<reason>`; query text validated statically only."* Delivery is not blocked. This gate never substitutes for Gates 1–3 and never justifies lowering them.

### Gate 4 — Impact analysis (only when public surface changed)

Skip this gate **only** when the change is fully internal:

- a private procedure of a non-export common module;
- a procedure of a form module that has no `Экспорт`;
- a comment / docstring / `//BSLLS:` suppression edit.

In every other case run impact analysis:

- For every changed export procedure / function, use **`get_method_call_hierarchy`** to find callers, and its callees when the routine's own dependencies may have changed.
- For a changed metadata or module object, use **`find_references`** to find dependents and, where relevant, its own dependency tree.
- For metadata changes (new attribute, renamed object, removed attribute): **`find_references`** to list every metadata reference that needs to be reviewed (`edt-mcp` has no separate graph-database usage-search tool — see `content/skills/mcp-1c-tools/docs/edt-mcp.md → Coverage gaps`). A project running the classic distribution's `1c-graph-metadata-mcp` instead uses `trace_call_chain(direction="callers"|"callees")` / `trace_impact(direction="downstream"|"upstream")` / `find_objects_using_object` / `find_usages_of_object`, with `get_method_call_hierarchy` / `graph_dependencies` (`1c-code-metadata-mcp`) as fallback.

Pass criterion: every caller / dependent listed by impact analysis was either not affected by the change, or explicitly handled in the plan, or explicitly noted as a follow-up risk in the delivery summary. Silent breakage of downstream code is a defect.

**Graceful degradation — when no applicable impact-analysis tool is exposed.** The applicable tools are `get_method_call_hierarchy` for routine changes and `find_references` for object / metadata changes (or their classic-distribution equivalents above). If none of them are exposed, do **not** silently skip the gate. Instead:

1. State the fact explicitly in the Delivery summary under **Risks** as a fixed line: *"Impact analysis not run — no MCP impact-analysis tool exposed in this session; downstream callers and metadata references were not enumerated."*
2. For metadata changes, perform a best-effort manual review based on what the agent already knows about the change (which forms / modules / queries touch the affected object) — list those callers as candidates that still need review, marked as such.
3. Do not promote a quick-fix to "verified" if a metadata or public-API change went through without impact analysis. If the change is risky and the user cannot accept the residual risk, hand off to a session that has the MCP exposed.

Skipping the gate without recording it under Risks is a defect.

### Gate 5 — Metadata validation (only when metadata / forms / layouts were changed)

Skip this gate **only** when no metadata object was touched.

There is **no XSD / raw-XML validator** on `edt-mcp` (unlike the classic distribution's `verify_xml`). Metadata correctness is enforced by the typed mutation API plus EDT's own validation:

- **`revalidate_objects(objects=[FQNs])`** for every changed object — it refreshes from disk first, so it also covers `.mdo` / `.form` files edited by hand or by an external tool.
- **`get_project_errors(objects=[FQNs])`** — pass criterion: zero `ERRORS` / `BLOCKER` items on the changed objects. `md-reference-intergrity` "lost reference" findings block `update_database` and are fixed with `resync_to_disk` (report first; `cleanDanglingReferences=true` only when the dangling entries are confirmed garbage).
- **Execution-path check.** Metadata mutations (new objects, attributes, tabular sections, forms, layouts, roles, DCS) must have gone through the `1c-metadata-manage` skill / `1c-metadata-manager` subagent — hard gate per `AGENTS.md → Skills and Subagents`, which on `edt-mcp` means `create_metadata` / `modify_metadata` / `delete_metadata` / `rename_metadata_object` / `adopt_metadata_object`. If hand edits of `.mdo` / `.form` were used, this gate passes only when the exception is one of those documented in `SKILL.md → Hard rule` **and** is stated in the delivery summary (`Metadata tooling: hand-edit — <exception>`); additionally cross-check `metadata-xml-workarounds.md` for the recurring traps. Hand-edited metadata without a stated documented exception is a gate failure — the same class as a skipped validator.
- **Destructive operations are two-phase.** `delete_metadata` and `rename_metadata_object` must be run once without `confirm` (preview of every change point / every reference that would be cleaned), reviewed, and only then with `confirm=true`. A `force=true` delete that leaves dangling references requires an explicit note in the delivery summary.
- **For form changes:** confirm the result with `get_metadata_details` on the form FQN (structure) and, when layout is the point of the change, `get_form_layout_snapshot` / `get_form_screenshot` (both need EDT started with `-DnativeFormBufferedLayoutRender=true`; a blank result is a missing JVM flag, not a pass).

A project running the classic distribution's `1c-code-metadata-mcp` instead uses `verify_xml` against the appropriate XSD (`get_xsd_schema` first if unsure of the type) as the primary check; for `Form.xml` edits also confirm the form opens in Configurator without warnings.
