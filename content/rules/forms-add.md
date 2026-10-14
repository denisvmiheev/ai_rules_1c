---
description: Generating or significantly altering a managed 1C form (`Form.xml` + `Form.Module.bsl`), including form-presentation rules — programmatic modification of typical forms, element placement, fill checking, form commands. Load from `forms.md` for any form-creation or form-presentation task.
alwaysApply: false
category: forms
---

# Adding or Modifying a Managed Form

This file owns the **rules**, not the MCP sequence. The pre-edit and post-edit MCP playbooks live in:

- `tooling-playbooks.md → Form Analysis and Generation` — full ordered list of MCP calls (`get_objects_by_tags`/`get_tags` → `get_form_layout_snapshot` → `get_metadata_objects` → `create_metadata` / `modify_metadata` on form FQNs → `revalidate_objects` → `get_project_errors`).

Do not duplicate that sequence here.

## Rules specific to creating / modifying a form

- **The `1c-metadata-manage` skill (form-manage section) is the mandatory execution path** for creating or structurally modifying `Form.xml` — hard gate per `AGENTS.md → Skills and Subagents`; the skill drives the toolchain (BOM, encoding, UID generation, ordering of `ChildObjects`). Hand-editing is allowed only within the narrow exceptions of `content/skills/1c-metadata-manage/SKILL.md → Hard rule` (unambiguous one-line fix; skill not available — stated once). This path is for forms of the project's own objects only — a **typical form** is never mutated declaratively (*Programmatic Modification of Typical Forms — hard rule* below).
- **XSD validation is mandatory** after any XML edit — `verify_xml` against the schema returned by `get_xsd_schema(object_type="Форма")`. A form that parses in your editor is not a form that loads in Designer.
- **Form-element naming.** Elements added to a typical form must carry the `{PREFIX}` prefix from `.dev.env`. Elements inside a newly created form (object already prefixed) do **not** repeat the prefix on every element — see `dev-standards-change-markers.md → "Metadata Naming"`.
- **Common pitfalls** are catalogued in `metadata-xml-workarounds.md` — read it before hand-editing the XML.
- **Region structure of the form module** — `module-structure.md → Form Module` (5 mandatory regions).

## Form-Presentation Rules

### Programmatic Modification of Typical Forms — hard rule

A **typical form** is any managed form of the vendor configuration — a form of a typical object or a typical common form, i.e. every form the project did not create itself under `{PREFIX}`. Typical forms are changed **only programmatically, through the `РедакторФорм` common-module library**. Never change their `Form.xml` / `Form.form`: not by hand, not with the `1c-metadata-manage` tools, not with EDT-MCP `create_metadata` / `modify_metadata` on form members (items, form attributes, commands, event handlers), not in the EDT / Designer form editor. A declarative change of a typical form is a **defect** with the same standing as a hand edit: it collides with every vendor update of that form, while code in the form module survives the update merge.

- **Where.** A server procedure `{PREFIX}ПодготовитьФорму()` in the company region of the form module, called once at the end of `ПриСозданииНаСервере` between change markers (`dev-standards-change-markers.md §3`). It starts with the re-entry guard `Если РедакторФорм.ФормаПодготовлена(ЭтотОбъект) Тогда Возврат; КонецЕсли;`.
- **How.** Build the element context with `РедакторФорм.НовыйКонтекстЭлемента(ЭтотОбъект, <Родитель>, <ПередЭлементом>)`, fill `Контекст.Свойства` (`Заголовок`, `ТолькоПросмотр`, …), then call the matching constructor: `НовыйРеквизитОбъектаФормы` for an object attribute (field kind follows the value type), `НоваяГруппаФормы` / `НоваяГруппаОбычная`, `НоваяТаблицаФормы`, `НоваяКомандаИКнопкаФормы`, `НоваяДекорацияНадпись`, …. The library differs between projects — read its live API (`get_module_structure` on `CommonModules/РедакторФорм/Module.bsl`, `responseFormat=detailed`) and one existing call site before writing code.
- **Metadata stays metadata.** A new attribute of a typical *object* is still created through the `1c-metadata-manage` skill / EDT-MCP; only its appearance on the typical form is code.
- **No library — stop.** If the project has no `РедакторФорм` common module, do not fall back to a form edit or to ad-hoc `Элементы.Добавить`: raise `CONFUSION` and let the user decide.
- **Out of scope.** Forms of objects the project created (`{PREFIX}` objects) are designed declaratively as usual (see *New Forms* below); adopted forms in an extension follow `extension-patterns.md`.

```bsl
&НаСервере
Процедура ПриСозданииНаСервере(Отказ, СтандартнаяОбработка)
	// ... typical code ...
	// {COMMENT_OPEN}
	{PREFIX}ПодготовитьФорму();
	// {COMMENT_CLOSE}
КонецПроцедуры

#Область {COMPANY}

// {COMMENT_OPEN}
&НаСервере
Процедура {PREFIX}ПодготовитьФорму()

	Если РедакторФорм.ФормаПодготовлена(ЭтотОбъект) Тогда
		Возврат;
	КонецЕсли;

	Контекст = РедакторФорм.НовыйКонтекстЭлемента(ЭтотОбъект, Элементы.ГруппаВыплата, Элементы.СпособВыплаты);
	Контекст.Свойства.Вставить("Заголовок", НСтр("ru = 'Характер выплаты'"));
	РедакторФорм.НовыйРеквизитОбъектаФормы(Контекст, "{PREFIX}ХарактерВыплаты");

КонецПроцедуры
// {COMMENT_CLOSE}

#КонецОбласти
```

### Placement of Added Elements

- If the form has tabs — add elements to a separate tab (e.g. "Additional" or with `{PREFIX}`).
- If no tabs — create a group without title for added elements.
- Typical form element names — with `{PREFIX}` prefix.

### New Forms (Non-Typical Objects)

- Separate header attributes and tabular sections into distinct tabs: "Main" (header), then one tab per tabular section.
- Fill "Header Data Path" property for pages with tabular sections.
- Reference fields — maximum width 27 characters.
- Multiline comment fields — width 79, height 3.

### Fill Checking

- Use "Fill check" property on form attributes.
- Before writing / posting, call `ПроверитьЗаполнение()`:

```bsl
Если Не ПроверитьЗаполнение() Тогда
	Возврат;
КонецЕсли;
```

### Form Commands

- When creating commands that modify data — enable "Modifies stored data" flag.

## Companion rules

| If the change also includes… | Also load |
|---|---|
| Event handlers (`ПриОткрытии`, `ПередЗаписью`, …), form-module logic, reserved names | `form-module.md` |
| Client-side async code (`Асинх` / `Ждать`) | `async-methods.md` |

This list is curated by the router file `forms.md`; load only the items you actually touch.
