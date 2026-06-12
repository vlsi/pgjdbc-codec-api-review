# Сравнение LLM-ревью архитектуры для Codec API в pgjdbc

Этот репозиторий — эксперимент по сравнению architecture review, которые несколько LLM-агентов выполнили для публичного Codec API в pgjdbc.

Цель эксперимента — показать не только фактические выводы моделей, но и сам процесс сравнения:

* как формулировалась исходная задача для architecture review;
* какие проблемы нашли разные модели;
* какие выводы совпали, разошлись или оказались неподтверждёнными;
* как исходную инженерную постановку сфокусировали в prompt для design review;
* как несколько больших LLM-ответов превратить в проверяемую матрицу claims;
* как выглядит финальный adjudication pass, когда два независимых сравнения почти сошлись.

Все запросы к моделям — и первичные ревью, и сравнения, и финальное сравнение — выполнялись с максимальным reasoning effort. Использованы Fable 5, GPT 5.5 и Opus 4.8.

Ревью выполнялось по состоянию кода в форке `vlsi/pgjdbc`, зафиксированному тегом [`codec-api-review-2026-06-12`](https://github.com/vlsi/pgjdbc/tree/codec-api-review-2026-06-12) (коммит `4b2df19`).

## Что здесь лежит

Это русская версия; английская — в [корне репозитория](../README.md). Файлы разбиты по этапам конвейера — от исходной задачи до финального сравнения.

### Исходная задача

* [`1-review-prompt-creation/design-review-prompt.md`](1-review-prompt-creation/design-review-prompt.md) — prompt для первичного architecture review Codec API: arrays, structs, user-defined types, standalone encode/decode, JDBC adapters, registry, metadata, performance и миграция от `ArrayEncoding` / `ArrayDecoding`.
* [`1-review-prompt-creation/initial-task.md`](1-review-prompt-creation/initial-task.md) — исходная постановка задачи до уточнений.
* [`1-review-prompt-creation/refinement-dialogue.md`](1-review-prompt-creation/refinement-dialogue.md) — сжатая стенограмма уточнений, после которых исходная постановка стала финальным prompt.

Исходная постановка уже содержала много технического контекста, но ещё не фиксировала важные развилки: нужен ли code review или design review, публичный ли это SPI, какие PostgreSQL types входят в scope и нужно ли закладывать standalone encode/decode API.

Полезным оказался не сам факт, что LLM «улучшила prompt», а итерация, которая выявила скрытые цели исследования. После уточнений prompt стал не просьбой посмотреть `Int4ArrayLeafCodec`, а заданием на architecture review публичной codec-системы для всех PostgreSQL types.

### Первичные architecture reviews

* [`2-review-execution/fable5.md`](2-review-execution/fable5.md) — результат Fable 5.
* [`2-review-execution/gpt55.md`](2-review-execution/gpt55.md) — результат GPT 5.5.
* [`2-review-execution/opus48.md`](2-review-execution/opus48.md) — результат Opus 4.8.

Эти файлы отвечают на один и тот же исходный prompt. Их полезно читать как независимые попытки найти архитектурные риски в одном коде.

### Судебная процедура сравнения

* [`3-comparison/comparison-prompt.md`](3-comparison/comparison-prompt.md) — prompt для сравнения первичных ревью.

Этот prompt задаёт процедуру: разложить ответы на атомарные claims, проверить существенные утверждения по коду, отделить facts от opinions, найти hallucinations, собрать матрицу совпадений и предложить практический next-step plan.

### Результаты сравнения первичных ревью

* [`3-comparison/gpt55.md`](3-comparison/gpt55.md) — сравнение, выполненное GPT 5.5.
* [`3-comparison/opus48.md`](3-comparison/opus48.md) — сравнение, выполненное Opus 4.8.

Оба результата сравнивают одни и те же первичные ревью, но независимо. Это полезно: видно, насколько устойчива сама процедура сравнения.

### Финальное сравнение сравнений

* [`4-adjudication/adjudication-prompt.md`](4-adjudication/adjudication-prompt.md) — prompt для финального adjudication pass.
* [`4-adjudication/gpt55.md`](4-adjudication/gpt55.md) — финальное сравнение двух судебных сравнений, выполненное GPT 5.5.
* [`4-adjudication/opus48.md`](4-adjudication/opus48.md) — финальное сравнение двух судебных сравнений, выполненное Opus 4.8.

Финальные adjudication-результаты почти совпали. Это хороший сигнал: существенные claims и практические выводы оказались устойчивыми к смене модели на этапе сравнения.

## Как читать

Если нужно быстро понять результат:

1. Начните с [`4-adjudication/gpt55.md`](4-adjudication/gpt55.md) или [`4-adjudication/opus48.md`](4-adjudication/opus48.md).
2. Откройте [`3-comparison/gpt55.md`](3-comparison/gpt55.md) и [`3-comparison/opus48.md`](3-comparison/opus48.md), если хотите увидеть, как получены финальные claims.
3. Вернитесь к первичным ревью, если интересует, какая модель первой заметила конкретную проблему.
4. Откройте [`3-comparison/comparison-prompt.md`](3-comparison/comparison-prompt.md), если интересна сама методика сравнения.
5. Откройте [`1-review-prompt-creation/design-review-prompt.md`](1-review-prompt-creation/design-review-prompt.md), если нужен полный контекст исходной инженерной задачи.

Если интересна методология, а не pgjdbc:

1. Прочитайте [`1-review-prompt-creation/initial-task.md`](1-review-prompt-creation/initial-task.md), чтобы увидеть исходную постановку.
2. Посмотрите [`1-review-prompt-creation/refinement-dialogue.md`](1-review-prompt-creation/refinement-dialogue.md), чтобы понять, какие цели уточнили до запуска ревью.
3. Прочитайте финальный [`1-review-prompt-creation/design-review-prompt.md`](1-review-prompt-creation/design-review-prompt.md).
4. Посмотрите один первичный review.
5. Посмотрите prompt судебной процедуры.
6. Сравните два результата судебной процедуры.
7. Посмотрите финальный adjudication prompt и один финальный результат.

## Методика

Эксперимент устроен в несколько этапов.

0. Сначала исходная инженерная постановка уточняется до prompt для design review. Здесь важно не просто переписать текст, а выявить скрытые решения: тип ревью, границы public API, scope типов, standalone encode/decode и критерии результата.
1. Затем несколько моделей независимо выполняют architecture review одного и того же кода.
2. Другие модели не делают review заново, а сравнивают результаты: извлекают claims, проверяют их по коду и маркируют статус.
3. Потом идёт финальное сравнение двух сравнений: оно проверяет, где сошлись уже сами adjudication-результаты.

Ключевая идея — не доверять уверенной формулировке без evidence.

Каждый существенный claim попадает в один из статусов:

* `confirmed` — подтверждается кодом или спецификацией;
* `partially confirmed` — в целом верно, но формулировка шире фактов;
* `unclear` — данных недостаточно;
* `false / hallucinated` — противоречит коду или спецификации;
* `design trade-off` — не баг, а выбор между несколькими разумными вариантами;
* `opinion` — рекомендация без строгого критерия.

Такой процесс помогает отделить:

* реальные архитектурные риски;
* спорные design trade-offs;
* неподтверждённые claims;
* галлюцинации;
* полезные, но не срочные рекомендации.

## Что показал эксперимент

Наиболее устойчивые выводы:

* public Codec SPI пока протекает внутренними типами pgjdbc;
* registry и lookup rules требуют более явной модели type identity, override и fallback;
* array path ещё не стал единым codec-based hot path;
* range и multirange metadata требуют отдельной модели, а не эвристик через `typelem`;
* JDBC compatibility gaps важны не меньше, чем внутренняя codec-архитектура;
* primitive fast path нужно проектировать явно, иначе общая container-модель станет слишком boxing-heavy;
* часть claims моделей оказалась не багами, а design trade-offs.

Самым полезным оказался не один «лучший» ответ, а пересечение независимых результатов плюс список расхождений, которые удалось проверить по коду.

## Как воспроизвести процесс

1. Дайте нескольким моделям [`1-review-prompt-creation/design-review-prompt.md`](1-review-prompt-creation/design-review-prompt.md) и доступ к тому же коду.
2. Сохраните их ответы как отдельные Markdown-файлы.
3. Дайте другой модели [`3-comparison/comparison-prompt.md`](3-comparison/comparison-prompt.md), исходный prompt и все первичные ответы.
4. Повторите шаг 3 другой моделью.
5. Дайте третьей модели [`4-adjudication/adjudication-prompt.md`](4-adjudication/adjudication-prompt.md) и два результата сравнения.
6. Проверьте вручную короткий список unresolved и high-severity claims.

Для нового проекта достаточно заменить исходный architecture review prompt и первичные ответы моделей. Судебная процедура почти не зависит от pgjdbc.

Если хочется воспроизвести не только comparison pipeline, но и подготовку prompt, начните с сырой постановки задачи и отдельно зафиксируйте ответы на вопросы:

* какой тип ревью нужен;
* что считается public API;
* какие сущности входят в scope;
* какие performance/correctness/usability цели важны;
* какие результаты считаются полезными после ревью.

## Disclaimer

Это исследовательский артефакт, а не официальный документ pgjdbc.

LLM-ответы могут содержать ошибки. Важные claims нужно проверять по коду, тестам, документации JDBC и поведению PostgreSQL.

Ценность эксперимента не в том, что одна модель «права», а в том, что независимые ответы можно привести к проверяемой форме: claims, evidence, status, unresolved questions и next steps.

## Лицензия

Материалы доступны по лицензии [CC BY 4.0](../LICENSE) — можно делиться и адаптировать с указанием авторства, в том числе в коммерческих целях.
