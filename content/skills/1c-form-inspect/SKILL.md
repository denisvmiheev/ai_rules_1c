---
name: 1c-form-inspect
description: "Read 1C managed forms — similar forms, element trees, bindings, commands, handlers, rendered layout, on-disk format spec — via edt-mcp and the docs MCP. Use before generating or changing a form; mutations go through 1c-metadata-manage."
argument-hint: "<Kind.Name> [form name]"
allowed-tools: mcp__edt-mcp__list_projects, mcp__edt-mcp__get_objects_by_tags, mcp__edt-mcp__get_tags, mcp__edt-mcp__get_metadata_objects, mcp__edt-mcp__get_metadata_details, mcp__edt-mcp__get_form_layout_snapshot, mcp__edt-mcp__get_form_screenshot, mcp__edt-mcp__get_template_screenshot, mcp__edt-mcp__get_server_status, mcp__1C-docs-mcp__formatspec
---

# 1c-form-inspect — read forms before touching them

Read-only. Creating or editing a form is the `1c-metadata-manage` skill (`docs/form-manage.md`; on this project the typed `edt-mcp` API); design rules — `content/rules/forms.md`. Hand-writing a form while the skill is available is a defect.

## Tools

Every call takes `projectName` (`list_projects`); take exact parameter names from the live schema.

| Need | Call | Notes |
|---|---|---|
| Similar existing forms as examples | `get_objects_by_tags` (+ `get_tags`), or `get_metadata_objects` filtered to forms | Comparable forms as layout examples |
| Element tree, attributes, commands, handlers | `get_metadata_details` on a **Form** FQN (`Document.SalesOrder.Form.DocumentForm`) | Structure as EDT models it |
| What the form actually renders | `get_form_layout_snapshot` | YAML snapshot of the calculated layout — cheap and diffable between states |
| A visual check | `get_form_screenshot` / `get_template_screenshot` | Genuine visual questions only; needs EDT started with `-DnativeFormBufferedLayoutRender=true` — a blank result is a missing flag (`get_server_status`), not a pass |
| On-disk XML format (Designer dumps, hand analysis) | `formatspec` (docs) | `name="1c-form-spec"` or `query="реквизиты формы"`; for `.mdo` / `.form` the typed API replaces the XML specs |

## Ordinary forms

Ordinary (non-managed) forms are binary in Designer dumps (`Forms/<Имя>/Ext/Form.bin`): unpack and rebuild them through `content/skills/v8unpack-cf/SKILL.md → Ordinary forms`, never edit `Form.bin` directly.

## After the change

`revalidate_objects` on the form's owner FQN → `get_project_errors` scoped to it; `get_form_layout_snapshot` when layout is the point of the change — `1c-validate`. Typed answers map to actions by code — `content/rules/mcp-policy.md → C. Server answers → actions`.

Tool catalog: `content/skills/mcp-1c-tools/docs/edt-mcp.md`.
