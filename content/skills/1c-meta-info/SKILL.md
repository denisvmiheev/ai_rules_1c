---
name: 1c-meta-info
description: "Facts about 1C metadata objects — passport, typed attributes, tabular sections, forms, roles, subsystems, objects by kind, name or synonym — from edt-mcp. Use before code or metadata that depends on an object's structure, and to find objects by synonym."
argument-hint: "<Kind.Name | description> [aspect]"
allowed-tools: mcp__edt-mcp__list_projects, mcp__edt-mcp__get_metadata_objects, mcp__edt-mcp__get_metadata_details, mcp__edt-mcp__get_objects_by_tags, mcp__edt-mcp__get_tags, mcp__edt-mcp__get_configuration_properties, mcp__edt-mcp__list_subsystems, mcp__edt-mcp__get_subsystem_content
---

# 1c-meta-info — facts about a metadata object

Facts only; a verdict about an object needs the validators of `1c-validate`. Search discipline and freshness — `content/rules/mcp-first-search.md`.

## Address

Metadata is addressed by FQN: `Catalog.Products`, `Document.SalesOrder`, `AccumulationRegister.Stock`, `CommonModule.WorkWithCounterparties`; a member appends its kind: `Catalog.Products.Attribute.Weight`, `Catalog.Products.Form.ItemForm`. English and Russian type tokens are accepted (`Справочник.Контрагенты`). Every call takes `projectName` (`list_projects`).

## Tools

| Need | Call | Key parameters |
|---|---|---|
| Passport of a known object: properties, attributes, tabular sections, forms, modules | `get_metadata_details` | FQN(s); a **Form** FQN renders the form structure, a **Role** FQN the rights matrix + RLS; `assignable: true` returns the property schema `modify_metadata` can set |
| What objects exist by kind / name / synonym | `get_metadata_objects` | `metadataType`, `nameFilter` (name or synonym fragment) |
| Comparable existing objects / forms as examples | `get_objects_by_tags` (+ `get_tags`) | tag names |
| Platform compatibility mode, configuration name | `get_configuration_properties` | — |
| Subsystem tree and content | `list_subsystems` / `get_subsystem_content` | subsystem FQN |

## Rules

- Retrieve the smallest projection that answers: names via `get_metadata_objects` for identification, one object's details for a field list; never expand every hit into full details without need.
- An empty result proves absence only after `resync_to_disk` / `revalidate_objects` when files were edited outside EDT (`content/rules/edt-workflow.md → Model vs disk`).
- Synonym search has no semantic lane: try the Russian synonym fragment, then the name, then `search_in_code` for the identifier.
- Extensions are separate EDT projects: pass the contour's `projectName`; an object adopted into an extension shows its base object plus the extension additions — the effective picture needs both.
- Typed answers map to actions by code — `content/rules/mcp-policy.md → C. Server answers → actions`.

Tool catalog: `content/skills/mcp-1c-tools/docs/edt-mcp.md`.
