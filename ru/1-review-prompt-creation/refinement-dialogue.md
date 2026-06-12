# Уточнения, которые сфокусировали prompt

Это сжатая стенограмма процесса, в котором исходная постановка превратилась в финальный [`design-review-prompt.md`](design-review-prompt.md).

Важная часть процесса: задача не сводилась к одношаговому «улучши prompt». Уточнения выявили скрытые развилки: какой тип ревью нужен, является ли Codec API публичным SPI, какие PostgreSQL types входят в scope и нужно ли закладывать standalone encode/decode API.

## Первый слой уточнений

После исходной постановки были предложены вопросы:

* Codec API должен быть публичным API для пользователей и расширений или внутренней деталью pgjdbc?
* В scope «все типы» входят domains, enums, ranges, multiranges или только built-in scalar types, composites и arrays?
* Какой результат нужен: архитектурный design review, список code findings по файлам или roadmap удаления `ArrayEncoding` / `ArrayDecoding`?
* Нужно ли рассматривать standalone API для кодирования и декодирования PostgreSQL wire representation?

## Ответы и последствия

Codec API был зафиксирован как публичный API.

Это добавило в prompt отдельный фокус на публичный SPI:

* сторонние библиотеки, например PostGIS, могут реализовать собственные кодеки;
* регистрация должна быть возможна через `ServiceLoader`, явную регистрацию или похожий механизм;
* public API не должен требовать зависимости от внутренних классов pgjdbc;
* нужно заранее продумать конфликты кодеков, приоритеты, fallback и границы stable/internal API.

Scope был расширен до всех PostgreSQL types:

* built-in scalar types;
* arrays;
* composites;
* domains;
* enums;
* ranges;
* multiranges;
* вложенные комбинации этих типов.

Формат результата был сфокусирован на design review:

* сначала архитектурная жизнеспособность;
* затем риски, missing abstractions и изменения дизайна;
* code findings только после того, как понятно, что архитектура приемлема;
* roadmap удаления `ArrayEncoding` / `ArrayDecoding` не является главной темой, потому что он зависит от готовности новой codec-based реализации.

## Standalone encode/decode API

Отдельным уточнением было решено заложить публичный API для прямого кодирования и декодирования PostgreSQL values:

* получить сырое binary/text value без немедленного декодирования;
* декодировать raw binary/text payload через зарегистрированный codec;
* закодировать Java object, например `CustomDto` или `CustomDto[]`, в binary/text representation;
* использовать это через стабильные публичные абстракции, а не через `PGStream`, `QueryExecutor`, `Field`, `Tuple`, `ArrayEncoding` или `ArrayDecoding`.

После этого prompt явно разделил два режима:

1. Connection-bound encode/decode.
   Этот режим использует metadata, registry, server settings и type cache конкретного соединения. Для первой версии он считается обязательным.

2. Offline encode/decode.
   Этот режим работает без живого соединения, но требует явно переданных type descriptors и server context. Его можно отложить, но архитектура не должна его блокировать.

## Что изменилось по сравнению с исходной постановкой

Исходная задача была в основном про конкретную реализацию array support: `Int4ArrayLeafCodec`, `GenericArrayLeafCodec`, `MultiDimArrayBinary`, `MultiDimArrayText` и возможность заменить `ArrayEncoding` / `ArrayDecoding`.

Финальный prompt стал шире и точнее:

* `Int4ArrayLeafCodec` рассматривается как проверка архитектурной линии, а не как самостоятельная цель;
* arrays, structs, domains, enums, ranges и multiranges анализируются как части одной codec-системы;
* JDBC APIs описаны как adapters поверх Codec API;
* public SPI и standalone encode/decode стали частью design constraints;
* performance/correctness/usability вопросы вынесены в отдельные проверяемые блоки;
* результат ревью сфокусирован на архитектурных рисках, а не на случайном наборе замечаний по коду.

