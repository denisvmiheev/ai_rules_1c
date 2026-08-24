---
description: Case book of common 1C platform pitfalls and proven fix templates
alwaysApply: false
category: quality
---

# Platform Cases & Fix Templates

Each case follows the schema: **problem → symptom → correct template → reference to ITS standard or short rationale**.

---

<!-- help-mcp-router -->

## Where this standard lives

**The normative text of this file is not inlined here.** It is one document of the `1c-standards` collection on the Help MCP server (`1C-docs-mcp`):

```
standards(name="platform-solutions")     # this standard, entire - the normal call
standards(query="<what you need>")  # only when unsure which rule governs
```

`standards` is the tool for this collection. `docsearch` / `docinfo` serve the platform documentation and cannot reach it, and **no tool of this server takes a `corpus` argument**. Name resolution, paging, budget, and what to do when the server is not exposed - **`content/rules/help-corpus-retrieval.md`**.

**Retrieve before you apply.** Every section below is a heading with no body: acting on a section title without the text behind it is inventing the rule, not following it. Fetch the rule once by name rather than a query per section.

Pinned source, readable directly: <https://github.com/comol/ai_rules_1c/blob/410951e74fd3e6b7a763cf49757935b9a34d3f31/content/rules/platform-solutions.md>

## Sections

Every heading this file has always had, reproduced so that existing `platform-solutions.md` section references and anchor links still resolve - the same compatibility shape `dev-standards-core.md` uses. Each is a retrieval target, not a summary.

## 1. Checking values for emptiness

## 2. Long-running operations

## 3. Temporary storage: correct functions

## 4. Transactions in event handlers

**Problem.** Nested transactions in `ПередЗаписью` / `ПриЗаписи` of an object lead to a double `НачалоТранзакции` and unexpected rollback behavior.

**Symptom.** "Транзакция не активна" on rollback, partially saved data on inner errors, cross-session locks.

**Template.** In object-write event handlers **do not open your own transaction** — the platform has already opened one. If an outer transaction is required, open it in the calling code; inside the handler, use `ТранзакцияАктивна()` only for diagnostic checks if needed.

```bsl
// In the calling code that invokes Записать().
НачалоТранзакции();

Попытка
    Документ.Записать(РежимЗаписиДокумента.Проведение);
    ЗафиксироватьТранзакцию();
Исключение
    ОтменитьТранзакцию();
    ВызватьИсключение;
КонецПопытки;
```

**Standard.** ITS: "Использование транзакций" — [std783](https://v8std.ru/std/783/) "Транзакции: правила использования".

---

## 5. Copying objects and tabular sections

## 6. Time on the server

**Problem.** Using `ТекущаяДата()` on the server gives different values across cluster nodes and ignores the session time zone.

**Symptom.** Documents with "future" or "past" dates when users work in different time zones; mismatched logs and postings.

**Template.**

```bsl
// On the server - do NOT use
Текущая = ТекущаяДата();

// On the server
Текущая = ТекущаяДатаСеанса();

// On the client - ТекущаяДата() is acceptable, but obtaining it from the server is preferred
```

**Standard.** ITS: "Использование функций для работы с датой и временем" — [std643](https://v8std.ru/std/643/) "Работа в разных часовых поясах".

---

## 7. Searching in collections: choosing by complexity

## 8. External components on the thin client

**Problem.** Synchronous attach of an external component blocks the UI and does not work in the web client.

**Template.** Use the asynchronous form via `НачатьПодключениеВнешнейКомпоненты` + `ОписаниеОповещения`:

```bsl
&НаКлиенте
Процедура ПодключитьСканер()
    Оповещение = Новый ОписаниеОповещения("ПослеПодключенияСканера", ЭтотОбъект);
    НачатьПодключениеВнешнейКомпоненты(Оповещение, "ОбщийМакет.СканерКомпонента", "Сканер", ТипВнешнейКомпоненты.Native);
КонецПроцедуры

&НаКлиенте
Процедура ПослеПодключенияСканера(Подключено, ДополнительныеПараметры) Экспорт

    Если Не Подключено Тогда
        ОбщегоНазначенияКлиент.СообщитьПользователю(
            НСтр("ru = 'Не удалось подключить сканер.'"));
        Возврат;
    КонецЕсли;

    // ...
КонецПроцедуры
```

**Standard.** ITS: "Внешние компоненты, асинхронный интерфейс" — [std700](https://v8std.ru/std/700/) "Установка внешних компонент и расширений платформы".

---

## 9. Managed locks and deadlock prevention

**Problem.** Missing or incorrect managed locks cause read-write conflicts during posting; inconsistent lock ordering across documents leads to deadlocks.

**Symptom.** Sporadic «Конфликт блокировок при выполнении транзакции», deadlocks under load (especially in document posting), reading uncommitted data, balance inconsistencies after parallel posting.

**Template — set a managed lock before reading register balances during posting:**

```bsl
Процедура ОбработкаПроведения(Отказ, РежимПроведения)

    Блокировка = Новый БлокировкаДанных;

    ЭлементБлокировки = Блокировка.Добавить("РегистрНакопления.ТоварыНаСкладах");
    ЭлементБлокировки.Режим = РежимБлокировкиДанных.Исключительный;
    ЭлементБлокировки.ИсточникДанных = Товары;
    ЭлементБлокировки.ИспользоватьИзИсточникаДанных("Номенклатура", "Номенклатура");
    ЭлементБлокировки.ИспользоватьИзИсточникаДанных("Склад",       "Склад");

    Блокировка.Заблокировать();

    // ... read balances and write movements within the same transaction ...

КонецПроцедуры
```

Headlines of the rules to follow: lock **before** reading; use `Исключительный` for writes and `Разделяемый` only for consistent reads; pass `ИсточникДанных` instead of iterating; lock the register, not the document; never call user dialogs / long-running operations under a transaction; set configuration lock mode to `Управляемый`.

**Authoritative deep-dive — `locks-and-transactions.md`.** That file covers the full theory: transaction boundaries, implicit vs explicit transactions, lock-ordering contract for the whole project, mass-operation patterns, status-log pattern, and the technological-log diagnostics (`TLOCK` / `TDEADLOCK`). When designing a new posting path or debugging a deadlock — read it first.

**Standard.** ITS: "Управление блокировкой данных в транзакции" — [std460](https://v8std.ru/std/460/) "Использование управляемого режима блокировки"; "Особенности проведения документов" — [std603](https://v8std.ru/std/603/) "Требования к проведению документов".

---

## 10. Background jobs from an external data processor (БСП)

## Extending the case book
