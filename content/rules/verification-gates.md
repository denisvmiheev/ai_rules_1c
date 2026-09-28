---
description: Verification execution gates for BSL and metadata — evidence reuse, syntax, logic, style, impact analysis, XML validation, and the platform batch check before an infobase apply
alwaysApply: false
category: quality
---

# Verification Gates — BSL, Impact, and Metadata

**When to load this file:** before validating or declaring a BSL / metadata change done. Determine depth and promotion triggers first via `verification-policy.md`.

Delivery-only soft gates and the final report contract live in `verification-delivery.md`.

All gates resolve `TOOL_*` through `content/rules/mcp-policy.md → Tool availability`. Evaluate task/depth triggers before tool presence: a needed `required` capability remains a blocker when absent. `off`/failed tools use only the gate's documented fallback; missing metadata, impact, live-IB or EDT evidence is never a pass. The gates below are written against `edt-mcp`'s validator model (tool mapping — `content/skills/mcp-1c-tools/docs/edt-mcp.md → Remap from the classic docker distribution`).

## Gate execution and evidence reuse

A gate is a requirement for the current artifact state, not a request to call the same tool
again. The agent that makes the final edit to an artifact owns its applicable validator run and
records the artifact path and content fingerprint (SHA-256 or Git blob ID), validator result,
run count and relevant execution context in the handoff / implementation report. A commit ID
alone does not identify uncommitted edits. Context includes the checked configuration / extension
and platform version when they affect the result.

The parent closing gate MUST reuse that evidence when the current fingerprint and relevant
context match the recorded state. Checking a fingerprint or reading the edit target is allowed;
it is not a repeated validator run.
It runs only missing or stale gates and MUST NOT repeat a validator against unchanged content
(`AGENTS.md → MCP Tool Calling → C.1`). Any later edit invalidates the affected validator
evidence; the final editor becomes the new owner. The same rule applies to `revalidate_objects`
and impact-analysis evidence.


## Hard gates — run on every full-cycle change

You MUST run all five gates in order. Each gate has an explicit pass / fail criterion and an explicit retry budget. When a required validator is not exposed in the current session, follow the graceful-degradation subsections (after Gate 3 and inside Gate 4) instead of silently skipping. **Gates 3a and 6 are conditional** — each runs only when its own trigger fires and its prerequisite (an exposed server, or a reachable platform + infobase) is present; neither ever replaces Gates 1–3.

The gate descriptions below state the `full` behaviour — the strictest level. When `VERIFICATION_DEPTH` is `standard` or `lite`, Gates 1–3 are modulated per `verification-policy.md → "Verification depth levels"`, promotion-trigger paths included: they are full-cycle and keep the `full` retry budget, but their gates follow the active depth.

> **Validator model on `edt-mcp`.** EDT owns one marker store; Gates 1–3 below are three *severity slices* of it, not three different engines. The standard sequence after an edit is `revalidate_objects(objects=[FQNs])` (refresh the markers for what you touched — it re-reads the disk first, so it also picks up edits made outside EDT) → `get_project_errors(...)` filtered per gate → `get_problem_summary` as the confirming pass. `write_module_source` additionally runs a **BSL syntax check before writing**, so a successful write is already a partial Gate 1 pass; `validate_query` checks query text on its own.

### Gate 1 — Syntax (`write_module_source` pre-write check → `revalidate_objects` → `get_project_errors severity=ERRORS`; `validate_query` for query text)

- Run on every touched `.bsl` module. No exceptions.
- When the module was written through `write_module_source`, its pre-write syntax check counts as the first half of this gate; still refresh and read markers, because a syntactically valid module can break references.
- Any new / changed query text (module code, DCS data set, dynamic list) — `validate_query` (`dcsMode=true` for СКД).
- Pass criterion: zero `ERRORS` items for the touched objects.
- Retry budget: canon — `verification-policy.md → Validator budget`. An `ERRORS` item is blocking: after fixing it, obtain a clean confirming run on the changed module. Under `full`, allow 3 calls total; under `standard`, 2. If the limit is exhausted without a clean pass on the latest state, Gate 1 fails. Gates 2 and 3 use the same policy with their own blocking severities.

### Gate 2 — Logic & performance (`get_project_errors` severity `BLOCKER` / `CRITICAL` / `MAJOR`, + `get_check_description`)

- Run on every touched module, scoped with `objects=[FQNs]`. Always after Gate 1 passes — reading checks on syntax-broken code wastes the pass.
- These severities carry EDT's substantive checks: query defects (`ql-*`, e.g. `ql-temp-table-index`), transaction handling (`begin-transaction`), platform-standard violations. Use `get_check_description(checkId)` to read what a finding actually means before fixing or dismissing it — the check id and its short UID both come from the `get_project_errors` rows.
- Pass criterion: no `BLOCKER` / `CRITICAL` item; a `MAJOR` item introduced by your change must be fixed or explicitly justified.
- Pre-existing findings outside your scope — leave alone (Surgical Changes) and, when relevant, list them as follow-ups.
- **Coverage gap.** There is no AI per-method analysis on `edt-mcp` (no AI reasoning pass). EDT checks are rule-based, so this gate does not catch reasoning-level defects on its own — pair it with the internal review checklist from `dev-standards-code-style.md §8` on any transactional / public-API / metadata-bearing change.

### Gate 3 — Style & standards (`get_project_errors` severity `MINOR` / `TRIVIAL` → `get_problem_summary`)

- Run on every touched module after Gate 2 passes; close with `get_problem_summary` as the confirming totals pass.
- Pass criterion: no severity totals worse than the pre-change baseline for the touched objects.
- `MINOR` / `TRIVIAL` items: same triage rule as Gate 2 — fix what your change introduced, leave pre-existing noise.
- For specific warnings that are intentional and justified: add a `//BSLLS:<rule>` suppression with a 1-line explanation, per `standards(name="dev-standards-code-style") → "Formatting"`. Blanket suppressions without justification are forbidden.
- `get_check_description` requires a check-descriptions folder configured in EDT's MCP preferences; if it returns a configuration error, `get_server_status` confirms the flag — record the finding without the explanation rather than dropping it.
- For a finding whose rationale is unclear from `get_check_description` alone, or when a review explicitly needs the ITS citation: `v8std_explain_diagnostics` on `v8std-local` (see `content/skills/mcp-1c-tools/docs/v8std.md`) resolves the check id to its standard clause.

### Graceful degradation for Gates 1–3 — when a validator is not exposed

Apply `TOOL_EDT` (`content/rules/mcp-policy.md → Tool availability`) at the active depth. An unavailable `required` validator blocks its gate. For `auto` absence/failure or `off`, use the compensation below and report the actual reason; policy never produces a passing verdict:

1. Record: *"Gate N unverified — `<tool>` (`<server>`): `<disabled by policy / not exposed / failed / unsuitable scope>`."*
2. Compensate with what is available. **The platform itself is the first fallback** when `PLATFORM_PATH` and `INFOBASE_PATH` are configured **and the checked configuration contains the current artifact**. Establish the source-to-IB match per `content/rules/designer-batch-checks.md → Bind the check to the current artifact` before running its check ladder. `/CheckModules` supplies syntax / context evidence for the loaded modules; `/CheckConfig` supplies structural evidence, not a replacement for business-logic or ITS review. Record the artifact fingerprint, target, platform / modes, process exit code, log path and `/DumpResult` code. A pass against an older configuration never verifies local changes. If a matching dev/test state cannot be established through an authorized workflow, record that limitation and use manual syntax review (paired keywords, directives, parameter lists) for Gate 1 and the internal checklist for Gates 2–3.

   **Minimum internal review checklist.** These independent checks remain available when a validator is missing. They do not reproduce a routed standard or prove compliance with standards that were not retrieved through MCP; report that evidence gap and follow `content/rules/help-corpus-retrieval.md` for any required standard.

   - **Quick-fix** — correctness and edge cases of the changed fragment; plus locks / transactions when the edit sits near transactional code. That is enough — do not run the full checklist on a 10-line fix.
   - **Full-cycle** — the full list: style, readability, correctness, edge cases, security, concurrency / locks / transactions, BSL-LS compliance.
   - Always consider whether an external transaction already exists (e.g. an object-write transaction) before opening a new one.
   - Findings follow `verification-policy.md → Validator budget` (a blocking defect needs a clean confirming run on the changed state; style noise does not start another loop; budget exhausted = unverified).
3. `auto`/`off` allow delivery with these limitations; `required` leaves the dependent verification incomplete. A transactional / metadata / public-API change without Gate 2 needs a follow-up validation run.

Skipping a gate without recording it under Risks is a defect — the same rule as Gate 4's graceful degradation below.

### Gate 3a — Live-IB smoke check (conditional; `edt-mcp` offline checks and a debug session)

Gate 3a supplies narrowly scoped evidence from a dev/test infobase. Distinguish **query parsing / metadata resolution** from **result correctness**: these are separate checks. `validate_query` (`edt-mcp`, offline) resolves the query against the project's metadata — it proves parsing and metadata resolution only, not rows, parameter values or RLS. Result correctness needs a live evaluation, which `edt-mcp` reaches only through a debug session (`debug_launch` → `evaluate_expression`): heavier than a lightweight smoke check, so it runs only to close a concrete correctness question. UI policy, verification depth and orchestration do not disable it.

**Triggers — run when all of the following hold:**

1. The change authored or modified 1C **query text** (module code, DCS scheme, dynamic list), a self-contained side-effect-free BSL function whose result the static validators cannot confirm, **or** the agent identifies a concrete behavioural result / boundary case requiring a read-only check. State the expected outcome before execution; do not add unrelated test runs.
2. The required capability is exposed in this session and allowed by tool policy (`validate_query`; `debug_launch` / `evaluate_expression` when a live evaluation is needed).
3. The connected infobase is a development / test base (`INFOBASE_ROLE`). **On a production infobase this gate is not run** — record the skip and move on.

**Execution:**

- **Query text → `validate_query`** (`dcsMode=true` for СКД). Pass criterion: no errors. It parses the query and resolves it against the project's metadata; it does **not** execute it, check parameter values or evaluate RLS.
- **Result correctness → expected-value comparison.** Only when a concrete correctness question is open: `get_applications` → `debug_launch` → `evaluate_expression` (read-only) against a dev / test IB, and compare returned rows / values against the stated scenario; record the infobase state and the technical user's rights. A run under the technical user does not prove RLS behaviour for other users. `evaluate_expression` without an expected-value comparison proves absence of a runtime exception only.
- **Pure function →** a **read-only** `evaluate_expression` fragment returning the expected value.
- **Mutations are out of scope for this gate.** No `Записать()` / `Удалить()` / `НачалоТранзакции` / register movements — the read-only discipline and consent rules of `content/skills/1c-live-ib/SKILL.md → Safety` apply unchanged. If confirming the change requires a mutation, that is a task for `1c-tester` against a test base, not for this gate.
- **Budget:** one call per applicable tool / artifact state; run only the checks needed to close an open question. Re-run after a relevant artifact or test-state change only — the no-change-repeat rule (`AGENTS.md → MCP Tool Calling → C.1`) applies.

**Failure is blocking for the artifact,** the same as a Gate 1 `ERRORS` finding: fix the query / fragment and re-run once against the changed state.

**When a trigger fired but the gate could not run** (server not exposed, production IB, or stale / unknown test state), record one line under **Risks**: *"Gate 3a not run — `<reason>`; `<remaining checks>` unverified."* When only parsing ran, report *"Gate 3a: parsing passed; metadata resolution / result correctness unverified"* unless separate evidence closes those questions. Delivery is not blocked by unavailable tooling, but never report an unperformed check as passed. This gate never substitutes for Gates 1–3 and never justifies lowering them.

### Gate 4 — Impact analysis (only when public surface changed)

This gate also checks callers of an internally edited export and verifies that a new metadata
addition is unwired. Such a bounded check does **not** itself promote quick-fix to full-cycle;
promotion depends on the risks it discovers (`verification-policy.md → Quick-fix gate`).

Skip this gate **only** when the change is fully internal:

- a private procedure of a non-export common module;
- a procedure of a form module that has no `Экспорт`;
- a comment / docstring / `//BSLLS:` suppression edit.

In every other case run impact analysis:

- For every changed export procedure / function, use **`get_method_call_hierarchy`** to find callers, and its callees when the routine's own dependencies may have changed.
- For a changed metadata or module object, use **`find_references`** to find dependents and, where relevant, its own dependency tree.
- For metadata changes (new attribute, renamed object, removed attribute): **`find_references`** to list every metadata reference that needs to be reviewed (`edt-mcp` has no separate graph-database usage-search tool — see `content/skills/mcp-1c-tools/docs/edt-mcp.md → Coverage gaps`). A project running the classic distribution's `1c-graph-metadata-mcp` instead uses `trace_call_chain(direction="callers"|"callees")` / `trace_impact(direction="downstream"|"upstream")` / `find_references`, with `get_method_call_hierarchy` / `find_references` (`1c-code-metadata-mcp`) as fallback.

Pass criterion: every caller / dependent listed by impact analysis was either not affected by the change, or explicitly handled in the plan, or explicitly noted as a follow-up risk in the delivery summary. Silent breakage of downstream code is a defect.

**Graceful degradation — when no applicable impact-analysis tool is exposed.** The applicable tools are `get_method_call_hierarchy` for routine changes and `find_references` for object / metadata changes. If neither is exposed, do **not** silently skip the gate. Instead:

1. State the fact explicitly in the Delivery summary under **Risks** as a fixed line: *"Impact analysis not run — no MCP impact-analysis tool exposed in this session; downstream callers and metadata references were not enumerated."*
2. For metadata changes, perform a best-effort manual review based on what the agent already knows about the change (which forms / modules / queries touch the affected object) — list those callers as candidates that still need review, marked as such.
3. Do not promote a quick-fix to "verified" if a metadata or public-API change went through without impact analysis. If the change is risky and the user cannot accept the residual risk, hand off to a session that has the MCP exposed.

Skipping the gate without recording it under Risks is a defect.

### Gate 5 — Metadata validation (only when metadata / forms / layouts were changed)

Skip this gate **only** when no metadata object was touched.

There is **no XSD / raw-XML validator** on `edt-mcp`. Metadata correctness is enforced by the typed mutation API plus EDT's own validation:

- **`revalidate_objects(objects=[FQNs])`** for every changed object — it refreshes from disk first, so it also covers `.mdo` / `.form` files edited by hand or by an external tool.
- **`get_project_errors(objects=[FQNs])`** — pass criterion: zero `ERRORS` / `BLOCKER` items on the changed objects. `md-reference-intergrity` "lost reference" findings block `update_database` and are fixed with `resync_to_disk` (report first; `cleanDanglingReferences=true` only when the dangling entries are confirmed garbage).
- **Execution-path check.** Metadata mutations (new objects, attributes, tabular sections, forms, layouts, roles, DCS) must have gone through the `1c-metadata-manage` skill / `1c-metadata-manager` subagent — hard gate per `AGENTS.md → Skills and Subagents`, which on `edt-mcp` means `create_metadata` / `modify_metadata` / `delete_metadata` / `rename_metadata_object` / `adopt_metadata_object`. If hand edits of `.mdo` / `.form` were used, this gate passes only when the exception is one of those documented in `SKILL.md → Hard rule` **and** is stated in the delivery summary (`Metadata tooling: hand-edit — <exception>`); additionally cross-check `metadata-xml-workarounds.md` for the recurring traps. Hand-edited metadata without a stated documented exception is a gate failure — the same class as a skipped validator.
- **Destructive operations are two-phase.** `delete_metadata` and `rename_metadata_object` must be run once without `confirm` (preview of every change point / every reference that would be cleaned), reviewed, and only then with `confirm=true`. A `force=true` delete that leaves dangling references requires an explicit note in the delivery summary.
- **For form changes:** confirm the result with `get_metadata_details` on the form FQN (structure) and, when layout is the point of the change, `get_form_layout_snapshot` / `get_form_screenshot` (both need EDT started with `-DnativeFormBufferedLayoutRender=true`; a blank result is a missing JVM flag, not a pass).
- **EDT-format sources (`USE_EDT=true`, MDO tree)** — this is the default shape here: `edt-mcp`'s `revalidate_objects` → `get_project_errors` / `get_problem_summary` **is** the metadata validation for `*.mdo` / `*.form`, recorded in the delivery summary as a validator result. Canon — `content/rules/edt-workflow.md → Validation`.
- **Form load on the platform** is evidence EDT validation does not give. When the change reaches an infobase, Gate 6 (`/CheckConfig` on the loaded configuration) supplies it; otherwise record under **Risks**: *"Form load not verified on the platform — open `<form>` in the client / run `/CheckConfig` before release."* Never claim the form opens without that evidence.


### Gate 6 — Platform batch check (only when the change reaches an infobase)

Gates 1–5 read source. They cannot answer the question the platform answers: does this configuration or extension **apply** to the target infobase. Run this gate when either trigger fires:

1. **An extension is about to be applied** to an infobase — by `/deploy-and-test`, `/update1cbase`, `/restore-testbase`, `/build-release`, or a `db-ops` load. A `&Вместо` / `&ИзменениеИКонтроль` interceptor naming a method that no longer exists in the vendor original passes Gates 1–5 untouched: the source is well-formed and the XML is schema-valid. Only `/CheckCanApplyConfigurationExtensions` sees it.
2. **The main configuration is about to be loaded and applied**, and the change touched metadata or module code that the MCP validators did not cover (a whole-snapshot deploy, a large refactor, a release build).

Execution and pass criterion — `content/rules/designer-batch-checks.md`: first bind the current artifact to the checked configuration, then the ladder (`/CheckModules` → `/CheckCanApplyConfigurationExtensions` → `/CheckConfig`), stop at the first failure, and the **three-signal verdict** (process exit code + `/DumpResult` + `/Out` diagnostics). Success phrases neutralize only their own fragment, never other diagnostics on the same line. Warnings fail this gate — `-WarningsAsErrors` will reject the same content at apply time.

Evidence includes the artifact fingerprint and source-to-IB match, target configuration / extension,
platform / modes, process exit code, log path and `/DumpResult` code. Reuse it only while the
artifact and relevant target state still match; a later load or source edit invalidates it.

**When the trigger fired but the gate could not run** (no `PLATFORM_PATH` / `INFOBASE_PATH`, or the base is production and no dev/test copy is available), record one line under **Risks**: *"Gate 6 not run — `<reason>`; extension applicability was not verified against an infobase."* Delivery is not blocked, but an extension change delivered without it must be flagged as unverified against a real base. This gate never substitutes for Gates 1–5.
