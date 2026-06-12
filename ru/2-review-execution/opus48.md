## Вердикт

**Дизайн жизнеспособен, но требует точечных изменений до масштабирования на остальные типы.**

Ядро правильное. Композиция контейнеров через единую точку `ctx.getCodecs().getByOid()` работает: `CompositeCodec`, `RangeCodec`, `DomainCodec` не переизобретают резолвинг дочерних кодеков, а делегируют его реестру. Это и есть нужная абстракция, и тесты уже доказывают сложные сценарии — array-of-struct, struct-of-array, 4 уровня вложенности, binary+text parity, SQLData, кодек через ServiceLoader, нормализация идентификаторов (45/45). Это зрелая ветка, а не набросок.

Но три вещи стоит починить **до** того, как добавлять `int2/int8/float4/float8/bool/uuid/temporal` и **до** стабилизации API, иначе дублирование и протечки внутренних типов застынут в публичном контракте:

1. публичный SPI протекает внутренними типами (`org.postgresql.core.*`, `org.postgresql.jdbc.*`);
2. у decode нет zero-copy-слайса, хотя у encode уже есть streaming;
3. в array-слое сосуществуют три пути, и новый walker не стоит на горячем пути чтения.

Сейчас весь SPI помечен `@Experimental` — это правильный и самый дешёвый момент для таких изменений.

---

## Что уже сделано хорошо

Чтобы критика читалась в контексте:

- **Централизованный резолвинг дочерних кодеков.** Один шлюз `CodecRegistry.getByOid()` → `resolveByTyptype()`. Контейнеры остаются stateless-синглтонами, берущими `PgType` в момент вызова.
- **Streaming-encode с back-patching.** `StreamingBinaryCodec(..., OutputStream)` + `BackpatchingBinarySink` убирают per-element `byte[]` при вложенном кодировании. Это честная оптимизация.
- **Защита от глубины.** `CodecDepth` закрывает DoS на рекурсивных типах.
- **Нормализация идентификаторов через OID.** `IdentifierNormalizingTypeMap` решает фидбэк Lukas Eder про qualified/quoted имена в `Map<String,Class<?>>` элегантно — резолвит и ключ запроса, и ключи пользовательской мапы через `regtype` и сравнивает по OID.
- **Primitive-специализация на чтении скаляра.** `decodeAsInt/Long/...` снимают boxing на горячем scalar read.

---

## 1. Основные архитектурные риски

### R1. Три параллельных array-пути, и новый не на горячем пути

Сейчас одновременно живут:

1. legacy `ArrayEncoding` / `ArrayDecoding` (напрямую из `PgArray`);
2. `ArrayCodec` — обёртка, которая для нативных типов зовёт legacy, а новый `MultiDimArray*` использует только для `Object[]` из composite/custom-элементов;
3. `ArrayLeafStreamingCodec` — новый специализированный путь, зарегистрирован только для `_int4`.

Критично: `ArrayLeafStreamingCodec.decodeBinary` и `ArrayCodec.decodeBinary` возвращают **ленивый `PgArray`** ([ArrayCodec.java:78](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayCodec.java)), а `PgArray.getArray()` декодирует через legacy `ArrayDecoding`. Новый `MultiDimArrayBinary.decode` срабатывает только в `decodeBinaryAs(int[].class/Integer[].class)`. То есть на обычном `getArray()` / `getObject()` новый walker **не исполняется** — выигрыша нет, а удалить `ArrayEncoding`/`ArrayDecoding` нельзя, потому что новый путь сам на них опирается.

Вывод: цель «избавиться от `ArrayEncoding`/`ArrayDecoding`» пока недостижима by design. Нужно выбрать одну модель и поставить её на путь чтения.

### R2. Две конкурирующие абстракции array-кодека

`ArrayCodec` (универсальный, композирует element-кодек по OID) против `ArrayLeafStreamingCodec` (отдельный кодек на каждый element-OID, с hand-written `ArrayLeafCodec`). Обе реализуют `StreamingBinaryCodec, StreamingTextCodec`, обе ходят в `MultiDimArray*`. Непонятно, кто канонический. Если масштабировать через `ArrayLeafStreamingCodec`, получите по отдельному кодеку и по отдельному leaf-классу на `int2/int8/float4/float8/bool/...` — ровно то размножение кода, которого вы опасаетесь. См. предложение P4.

### R3. Порядок резолвинга ломает domain-over-array

`resolveByTyptype()` проверяет `isArray()` (это `typcategory == 'A'`, [PgType.java:390](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgType.java)) **раньше** `isDomain()` ([CodecRegistry.java:486](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java)). Домен наследует `typcategory` базового типа, поэтому `CREATE DOMAIN d AS int[]` даёт `typtype='d'`, `typcategory='A'` → попадает в `ArrayCodec`, а не в `DomainCodec`. Дальше `ArrayCodec.streamBinaryArrayViaCodec` берёт `arrayType.getTypelem()`, но у домена `typelem == 0` → element-кодек не находится. `typcategory` — более слабый сигнал, чем `typtype`, и должен проверяться последним.

### R4. Multirange не поддержан вообще

`typtype='m'` отсутствует в `resolveByTyptype()`, имена `*multirange` не зарегистрированы → `FallbackCodec`. В целях это есть, в коде — нет. Молчаливый fallback хуже явной ошибки.

---

## 2. Missing abstractions

- **`api.codec.TypeDescriptor`** — публичный read-only-интерфейс метаданных. Сейчас весь SPI завязан на конкретный `org.postgresql.jdbc.PgType`.
- **Decode-слайс** — симметрия к `StreamingBinaryCodec`. Нужна сигнатура вида `decodeBinary(byte[] data, int offset, int length, ...)` либо `ByteBuffer`/курсор, чтобы контейнеры не делали `copyOfRange` на каждый элемент.
- **`CodecContext` как интерфейс.** Сейчас это `final`-класс, конструируемый только из `BaseConnection`. Это блокирует offline-режим (см. L3).
- **Публичный resolver дочерних кодеков.** Чтобы сторонний контейнерный кодек резолвил вложенные типы, не трогая `CodecRegistry` и `TypeInfo` (оба фактически internal).
- **Явные capability-флаги** (`supportsBinaryRead/Write`, `textRead/Write`). Сейчас способность выводится через `instanceof BinaryCodec/TextCodec` и «это streaming?» — неявно и не самодокументируемо.
- **Модель конфликтов SPI** — приоритеты/детект дублей (см. CG в gaps).

---

## 3. Где публичный API протекает внутренней реализацией

Это самый важный раздел, потому что чинить его после появления сторонних зависимостей дороже всего.

- **SPI-интерфейсы из `api.codec` ссылаются на `org.postgresql.jdbc.PgType` и `org.postgresql.jdbc.CodecContext`** — пакет, где лежит вся внутренняя кухня. Пакет `api.codec` чистый по имени, но не по ссылкам.
- **`CodecContext` (публичный, но в `org.postgresql.jdbc`) отдаёт через геттеры:** `BaseConnection`, `TypeInfo`, `Encoding` (всё `org.postgresql.core`), плюс `CodecRegistry`, `JavaTypeRegistry`, `TimestampUtils`. Сторонний кодек, которому нужны метаданные или дочерний кодек, неизбежно импортирует `org.postgresql.core.*` — ровно те «внутренности» (рядом с `Field`, `Tuple`, `QueryExecutor`), которые вы хотите спрятать.
- **`PgType` конструируется из `ObjectName` и `List<PgField>`** — оба internal. Для offline/standalone это нерабочая граница.
- **`BackpatchingBinarySink` — package-private** ([BackpatchingBinarySink.java:18](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/BackpatchingBinarySink.java)). Сторонний `StreamingBinaryCodec` его не видит и не может сделать `instanceof`, поэтому всегда получает медленный `byte[]`-путь. Фактически streaming-выигрыш закрыт для third-party и доступен только built-in-кодекам в том же пакете.

---

## 4. Performance risks

- **PR1. `GenericArrayLeafCodec.readLeaf` делает `Arrays.copyOfRange` на каждый элемент** ([GenericArrayLeafCodec.java:116](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java)). Любой неспециализированный элемент (composite, uuid, numeric, text, int8…) аллоцирует копию на decode. Копию избегает только `_int4`. Это прямое следствие отсутствия decode-слайса (L-abstraction выше).
- **PR2. Decode идёт через legacy `PgArray`/`ArrayDecoding`** даже для `_int4` на `getArray()` — типизированный leaf-выигрыш не реализуется на горячем пути (см. R1).
- **PR3. Нет primitive-специализации на encode.** Кодирование `Integer[]`/`Object[]` боксит каждый элемент; на SPI нет `encodeInt`/`encodeLong`. Специализация есть только на decode-скаляре.
- **PR4. Специализированный leaf переинлайнивает scalar-логику.** `Int4ArrayLeafCodec.writeLeaf`/`appendLeaf` дублируют int4-кодирование вместо вызова `Int4Codec`. Для int это терпимо; для `numeric`/`timestamptz` переинлайнивать нельзя, поэтому они уйдут в боксящий generic-путь. Перф-история раздваивается, и цель «не размножать логику для int2/int4/int8/float4/float8/bool» не достигается.
- **PR5.** `MultiDimArrayBinary` использует reflection на внешних измерениях — приемлемо, стоимость ограничена произведением внешних размерностей, не числом элементов.

---

## 5. Correctness risks

- **CR1.** domain-over-array (R3) и **CR2.** multirange (R4) — выше.
- **CR3. Lower bounds теряются молча.** encode всегда пишет lower bound = 1 ([MultiDimArrayBinary.java:132](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java)), decode читает и игнорирует (строка 195). Совпадает с legacy-поведением JDBC, но non-1 bounds исчезают — это надо явно задокументировать.
- **CR4. Разное квотирование в text-массиве.** `GenericArrayLeafCodec.appendLeaf` квотирует каждый non-null элемент, `Int4ArrayLeafCodec` — нет, legacy оставляет числа без кавычек. PostgreSQL принимает квотированные числа, так что функционально ок, но это другие байты на проводе. Проверьте, что тесты не ассертят точные литералы, и крайние случаи (пустая строка-элемент, элементы с разделителем/скобками).
- **CR5. У нового generic-пути нет text-read leaf.** Text-decode массивов полностью на legacy `ArrayDecoding`. Путь асимметричен: пишет text, но не читает.
- **CR6. Dropped attributes в `CompositeCodec` (`attisdropped`).** Агент не нашёл фильтрации. В binary record-формате удалённые колонки приходят как NULL со своим OID; если список полей в text и binary трактует их по-разному, decode рассинхронизируется по позиции. Это классический composite-баг — **обязательно проверить** перед расширением.
- **CR7. `registerByClass(codec.getDefaultJavaType(), codec)` для SPI** ([CodecRegistry.java:271](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java)). SPI-кодек с default-типом `String`/`Object` молча перекроет встроенное кодирование `String`. Нужна детекция коллизий.

---

## 6. JDBC compatibility gaps

- **CG1. `setObject(i, obj)` без type hint для произвольного DTO** не ищет кодек по Java-классу (`findCodecFor` есть, но не подключён к этому пути) → ошибка. Запись кастомного DTO требует явного типа.
- **CG2. CallableStatement OUT-параметры** декодируются «попутно» через ResultSet (работает), но `registerOutParameter` не выбирает кодек по типу; именованные геттеры частично не реализованы (план T2). Тесты добавлены.
- **CG3. Offline encode/decode** заблокирован дизайном `CodecContext` (L3).
- **CG4. `addDataType` и codec-реестр — разные механизмы.** Зарегистрированный пользователем `PGobject`-подкласс не используется как кодек; он живёт только в legacy-fallback `getObject`. Связь стоит задокументировать.
- Хорошие новости: слайсы `getArray(index,count)`, `getResultSet`, `getArray(Map)`, `createArrayOf`/`createStruct`, updatable ResultSet для Struct/array — уже покрыты тестами. `getObject(i, T[].class)` сознательно отложен (решение V2 #7).

---

## 7. Конкретные предложения по архитектуре (до масштабирования)

- **P1. Ввести `api.codec.TypeDescriptor` (метаданные) и `api.codec.CodecContext` (интерфейс).** Снять SPI с `org.postgresql.jdbc.PgType`/`CodecContext`, оставить внутренние реализации. Самое важное изменение: после появления third-party-зависимостей его уже не сделать, а сейчас API `@Experimental`.
- **P2. Добавить decode-слайс:** `decodeBinary(byte[] data, int offset, int length, TypeDescriptor, CodecContext)` (или `BinaryReader`-курсор). Контейнеры передают слайс вместо `copyOfRange`. Симметрично `StreamingBinaryCodec`.
- **P3. Поднять `BackpatchingBinarySink` (или эквивалент) в `api.codec`**, иначе streaming-SPI работает только для built-in.
- **P4. Свернуть к одной модели массива:** один универсальный `ArrayCodec`, композирующий element-кодеки, плюс **опциональный** SPI-интерфейс fast-path (нынешний `ArrayLeafCodec`), который scalar-кодек может реализовать для `int[]/long[]/...`. Тогда `int2/int8/float4/float8/bool` получают быстрый путь, реализовав один маленький интерфейс на уже существующем scalar-кодеке — без параллельного кодека и без переинлайненной scalar-логики. Это прямой ответ на «как не размножить код».
- **P5. Добавить primitive-encode** в тот же fast-path leaf, чтобы `int[]` кодировался без boxing.
- **P6. Починить precedence в `resolveByTyptype`:** typtype-проверки (composite/domain/enum/range/multirange) до typcategory-проверки массива; либо детектировать массив только при `typtype=='b' && typcategory=='A'`.
- **P7. Добавить multirange-кодек** (`typtype='m'`), композирующий range-кодек, либо явную ошибку «не поддержано, потому что…».
- **P8. Определить политику конфликтов SPI:** детект дублей по имени, явный приоритет (`getPriority()` или упорядоченный ServiceLoader), лог при перекрытии built-in или другого SPI. Решить, можно ли SPI вообще перекрывать built-in.
- **P9. Schema-qualified-регистрация.** `codecsByName` ключуется по простому имени, поэтому одинаковые имена в разных схемах коллизируют. Минимум — задокументировать, что name-регистрация schema-agnostic, а OID-регистрация (`registerByOid`) — способ развести типы.

---

## 8. Вопросы, которые нужно решить до расширения

1. **Один array-кодек или по кодеку на element-OID?** (рекомендую один + fast-path SPI, P4).
2. **`CodecContext`/`PgType` — часть стабильного API или вводим абстракции `api.codec`?** (рекомендую второе, P1).
3. **Ownership `byte[] data` на decode:** borrowed (кодек не вправе удерживать) или owned? Задокументировать, затем добавить слайс (P2).
4. **Domain identity:** разворачивать в base (текущий `DomainCodec`) или сохранять? Для `getObject` разворот ок, для имён типов в `Struct`/SQLData и для domain-over-array — нет.
5. **Composite dropped attributes** — обработаны? (CR6, требует проверки в коде/тесте).
6. **Multirange — в скоупе v1?**
7. **Политика override и приоритетов SPI** (P8).
8. **Offline API:** зафиксировать `CodecContext` как интерфейс сейчас, чтобы не заблокировать (CG3).
9. **Text-read массивов:** новый путь когда-нибудь возьмёт парсинг на себя или делегирование `ArrayDecoding` остаётся навсегда? (CR5, R1).

---

Архитектура приемлема, поэтому логичный следующий шаг — перейти к code-level findings. Готов:

- **(A)** проверить по коду CR6 (dropped attributes в `CompositeCodec`) и CR4 (квотирование) — это самые вероятные «тихие» баги; могу поднять PostgreSQL через Docker и прогнать parity-тесты binary/text;
- **(B)** прототипировать P4 на `int8` или `bool` — показать, что fast-path как опциональный интерфейс на scalar-кодеке убирает размножение кода;
- **(C)** набросать сигнатуры `api.codec.TypeDescriptor` / `CodecContext` / decode-слайса (P1–P3), чтобы зафиксировать публичную границу до расширения.
