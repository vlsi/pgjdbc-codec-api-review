## Вердикт

**Дизайн жизнеспособен, но требует изменений перед масштабированием.** Ядро новой array-обвязки — разделение на shape walker (`MultiDimArrayBinary`/`MultiDimArrayText`) и типизированные leaf-циклы (`ArrayLeafCodec`) — правильное и тиражируется на `int8`/`float8`/`bool`/`uuid` механически. Streaming encode с back-patch длин, кэш кодеков на `Field`, примитивные специализации `decodeAsInt`/`decodeTextBytesAsInt` — всё это стоит сохранить как есть.

Но три вещи надо решить **до** тиражирования на остальные типы, потому что они меняют сигнатуры и контракты, а цена правки растёт с каждым новым кодеком:

1. граница public SPI / internal сейчас не существует — `api.codec.*` тянет за собой `BaseConnection`, `TypeInfo` и JDBC-классы;
2. decode-сторона API не имеет slice-абстракции (копия `byte[]` на каждый элемент/поле) и вообще не имеет text-decode пути для массивов — `ArrayLeafCodec.readLeaf` существует только для binary;
3. identity-модель registry — голые строки имён без схемы, SPI-кодеки глобально перекрывают built-in.

Дальше — по разделам, которые вы просили.

---

## 1. Что уже хорошо (сохранить при любой переработке)

- **Декомпозиция leaf/walker** ([MultiDimArrayBinary.java:30-36](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java#L30)): walker владеет header/dimensions/rectangular-валидацией и ходит по внешним измерениям через reflection (стоимость ограничена произведением внешних измерений), leaf владеет горячим типизированным циклом. `Int4ArrayLeafCodec` — это ~150 строк на тип; `int2/int8/float4/float8/bool` добавляются без дублирования формата.
- **`BackpatchingBinarySink` + `reserveInt32`/`setInt32At`** — правильное решение для length-prefixed вложенного encode без промежуточных `byte[]`. `GenericArrayLeafCodec.writeLeaf` ([GenericArrayLeafCodec.java:84-88](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java#L84)) и `CompositeCodec` (строки 765-769) уже используют его для streaming-элементов.
- **Кэширование кодека на `Field`** (`Field.initializeCodec`, один раз на колонку) и пре-резолв per-field кодеков в `PgSQLInput*`/`PgSQLOutput*` при конструировании.
- **`getInt()` не боксит**: binary → `decodeAsInt(byte[])`, text → `decodeTextBytesAsInt` с ASCII fast path ([Int4Codec.java:107-117](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/Int4Codec.java#L107)).
- **Эпоха инвалидации type cache** на DDL и `SET search_path`, ленивые composite fields с фильтром `attisdropped` на сервере, резолв имён через `?::regtype` — qualified/quoted/case-folding делает сервер, а не клиентский парсер. Это закрывает большую часть замечания Lukas Eder про `Map<String, Class<?>>`: `IdentifierNormalizingTypeMap` сводит ключи к OID через regtype, так что `"myschema.mytype"` и `"\"MyType\""` матчатся.
- **`CodecDepth`** (лимит 64) против рекурсивных типов, **`FallbackCodec`** — registry никогда не возвращает null.

---

## 2. Основные архитектурные риски

### Р1. Public SPI сцеплён с internal намертво (блокирует и стабилизацию, и offline-режим)

- `org.postgresql.api.codec.Codec/BinaryCodec/TextCodec` ссылаются на `org.postgresql.jdbc.CodecContext` и `org.postgresql.jdbc.PgType` — типы из jdbc-пакета, который исторически internal.
- `CodecContext.getConnection()` возвращает **`BaseConnection`** ([CodecContext.java:275-278](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java#L275)), `getTypeInfo()` — **`core.TypeInfo`**, который теперь сам импортирует jdbc-классы (циклическая зависимость core → jdbc). Любой сторонний codec, которому нужен lazy `PgArray`, обязан звать `ctx.getConnection()` — то есть зависит от всего internal ядра.
- `ArrayLeafStreamingCodec.decodeBinary` возвращает `new PgArray(ctx.getConnection(), …)` ([ArrayLeafStreamingCodec.java:92-95](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayLeafStreamingCodec.java#L92)) — JDBC-объект как результат wire-декода.
- Default-методы `BinaryCodec.decodeAsBoolean` зовут `org.postgresql.jdbc.BooleanTypeUtil` ([BinaryCodec.java:170-174](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/BinaryCodec.java#L170)).
- `PgType` — конкретный класс с пятью публичными конструкторами и `withFields`; как стабильный контракт его не зафиксировать.

Следствие для offline encode/decode: режим «без живого соединения» сейчас **архитектурно заблокирован**, а не просто отложен. Connectionless-конструктор `CodecContext` package-private и собирает полуживой объект (`getTypeInfo()`/`getCodecs()` бросают), `withTypeMap` на нём отказывает. Вы просили «offline можно отложить, но не блокировать» — текущий контракт это требование не выполняет.

### Р2. Decode без slice: копия на каждый элемент и каждое поле

`decodeBinary(byte[] data, …)` принимает только полный буфер. Поэтому:

- `GenericArrayLeafCodec.readLeaf` делает `Arrays.copyOfRange` **на каждый элемент** ([GenericArrayLeafCodec.java:116](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java#L116));
- `CompositeCodec.decodeBinaryFields` — `new byte[length]` **на каждое поле** (CompositeCodec.java:156-157);
- `RangeCodec` — копия на каждую границу.

Для `array-of-struct-of-array` одни и те же байты копируются на каждом уровне вложенности: O(payload × depth) аллокаций и memcpy. Encode-сторона эту проблему решила streaming-синком; decode-сторона симметричного решения не имеет. Это **самое дорогое для later-fix изменение**, потому что трогает сигнатуру каждого decode-метода каждого кодека — менять надо до написания остальных ~30 кодеков, а не после.

### Р3. Text decode массивов в новой архитектуре отсутствует как класс

`ArrayLeafCodec` имеет `writeLeaf` (binary), `readLeaf` (binary), `appendLeaf` (text encode) — но **нет text read**. `MultiDimArrayText` умеет только encode. Весь парсинг текстовых литералов массивов по-прежнему живёт в `ArrayDecoding.buildArrayList`/`readStringArray` (через `PgArray`), включая quoting, escaping, `typdelim` и `[l:u]=`-префиксы. Цель «полностью избавиться от `ArrayEncoding`/`ArrayDecoding`» недостижима без четвёртого leaf-примитива (text reader) и общего токенизатора текстового литерала. Это не правка, а недостающая четверть дизайна — и её стоит спроектировать сейчас, чтобы убедиться, что интерфейс `ArrayLeafCodec` её выдержит (скорее всего понадобится курсор/токенизатор вместо `String` целиком, и он же переиспользуется composite-парсером).

### Р4. Identity-модель registry: голые имена, глобальный SPI, нет приоритетов

- `Codec.getTypeName()` — строка без схемы; `codecsByName` ключуется по `pgType.getTypeName().getName()` ([CodecRegistry.java:410](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L410)). Сторонний codec с именем `point` перекрывает built-in для **всех** схем — вы уже наступили на это в тестах (коммит 099675898 «stop ServicePointCodec from shadowing built-in point globally»). Для PostGIS-сценария это сработает ровно до тех пор, пока у пользователя нет своего типа `geometry` в другой схеме.
- SPI-кодеки загружаются в static (`spiCodecs`) один раз на classloader и при создании каждого registry перекрывают built-in простым `put` ([CodecRegistry.java:268-273](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L268)). Конфликт двух SPI-кодеков на одно имя решается порядком classpath — недетерминированно и молча. Исключения при загрузке глотаются без логирования (комментарий «In production, this should use…» это признаёт).
- Спецрешение V2 «SPI scope per-Driver» фактически не выполнено: scope — static per-classloader.
- `registerByName`/`registerAlias` публичны, но не инвалидируют `oidCache` (в отличие от `registerCustomCodec`) — после registry может отдавать устаревший резолв.
- Одноаргументные `getBinaryCodec(int oid)`/`getTextCodec(int oid)` зовут `getByOid(oid, null)` и для холодного кэша возвращают `FallbackCodec` даже для известных типов. `RangeCodec` использует именно их (RangeCodec.java:101, 171), выбрасывая уже резолвнутый `PgType` — латентный баг.

### Р5. Двоепутье на горячих путях чтения

Сейчас три системы декодирования сосуществуют:

| Путь | Что используется |
|---|---|
| `getObject(i, int[].class)` | новый `ArrayLeafStreamingCodec` → `MultiDimArrayBinary` |
| `getArray(i).getArray()` | старый `ArrayDecoding` с hardcoded per-OID декодерами (не кодеки!) |
| `getDate/getTime/getTimestamp`, text-`getString`, text-`getBigDecimal` | hardcoded switch / inline-парсеры мимо кодеков |

Переходно это нормально (вы сами так и планировали), но именно здесь живут parity-риски: например, `decodeBinaryAs(int[].class)` работает, а `decodeTextAs(int[].class)` упадёт, потому что text-путь идёт через `PgArray.getArray()`, который возвращает boxed `Integer[]` ([ArrayLeafStreamingCodec.java:119-138](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayLeafStreamingCodec.java#L119)). Binary/text-parity нужно закрепить тестами до конвертации следующих типов, иначе каждый перенесённый тип будет тихо менять поведение одного из путей.

---

## 3. Missing abstractions

1. **Slice/view для decode** — `decode(byte[] data, int offset, int length, …)` или лёгкий `BinaryView`. Без него п. Р2 не лечится. (Альтернатива `ByteBuffer` — приемлемо, но дороже по дисциплине ownership.)
2. **Text-токенизатор + `LeafTextReader`** — недостающий read-аналог `appendLeaf` (п. Р3). Один курсорный парсер должен обслуживать массивы, composites и ranges: сейчас у composite свой парсер (`parseCompositeText`), у массивов — `ArrayDecoding`, у range — `PGRange.parse`. Три реализации quoting/escaping — это три набора edge-case-багов.
3. **Range subtype в метаданных.** `PgType` не знает `rngsubtype` — `pg_range` не читается вообще, а `typelem` у range-типов равен 0. Text-путь обходит это «не парсим границы», binary-путь падает (см. Р6-2). Нужен либо `PgRangeType` с `subtypeOid` (и тогда же `PgMultirangeType` с `rngtypid`), либо общее поле. Multirange сейчас отсутствует полностью — `resolveByTyptype` не имеет ветки `'m'`, тип уходит в `FallbackCodec` (для первой версии терпимо, но это в списке ваших целей).
4. **Capabilities кодека.** Сейчас «binary поддержан» определяется `instanceof BinaryCodec`, а «binary не смог» — выбросом исключения изнутри. Нет способа спросить заранее: codec умеет binary-read, но не binary-write? `GeometricCodec` для circle/line — text-only, и выбор формата параметра решается глобальным `binaryTransferSend(oid)`-set, никак не связанным с возможностями кодека. При появлении сторонних кодеков формат надо выбирать по пересечению (server-supported × codec-supported × настройки) — заложите `supportsBinaryRead()/Write()` или enum-set capabilities.
5. **Scoped-регистрация вместо `getTypeName()`.** Регистрация должна описывать, к чему codec применим: OID / (schema, name) / предикат по `PgType` — плюс источник (builtin/SPI/connection) для разрешения конфликтов. Сейчас идентичность кодека — одна строка.
6. **Decode-streaming (visitor) отсутствует.** Encode-стриминг есть, decode — только материализация. Для больших массивов/composite это половина выигрыша. Можно отложить, но сигнатуры slice-уровня (п. 1) — фундамент и для него.

---

## 4. Где public API протекает в internal (конкретно)

| Утечка | Место |
|---|---|
| `BaseConnection` из `CodecContext.getConnection()` | [CodecContext.java:275](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java#L275) |
| `core.TypeInfo` из `getTypeInfo()`; сам `TypeInfo` теперь импортирует `CodecRegistry`/`JavaTypeRegistry`/`PgType` (core→jdbc цикл) | [TypeInfo.java:8-11](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/core/TypeInfo.java#L8) |
| `PgArray` (JDBC-класс) как результат `decodeBinary` массивных кодеков | [ArrayCodec.java:78-82](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayCodec.java#L78) |
| `BooleanTypeUtil` в default-методах публичного интерфейса | [BinaryCodec.java:172](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/BinaryCodec.java#L172) |
| `PgType` с публичными конструкторами и `Oid.BOX`-логикой в них | [PgType.java:52-56](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgType.java#L52) |
| JDBC-полиси внутри wire-контекста: `prefersJavaTimeFor*`, `convertBooleanToNumeric` в `CodecContext` | [CodecContext.java:52-61](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java#L52) |

Последний пункт — не «убрать», а разделить: wire-контекст (charset, server params, `TimestampUtils`-аналог) против JDBC read policy. Сторонним кодекам нужен первый; второй — деталь адаптеров. Заодно: в контексте нет server version и `integer_datetimes`-подобных параметров — для standalone encode/decode они понадобятся.

---

## 5. Performance risks

1. **Per-element копии при decode** — п. Р2, главный пункт.
2. **`PgConnection.getCodecContext()` аллоцирует новый контекст на каждый вызов** (PgConnection:2101-2106), а `ArrayDecoding.MappedTypeObjectArrayDecoder` зовёт его **на каждый элемент массива** (ArrayDecoding:453). Контекст immutable — кэшируйте на соединении, пересоздавайте при смене typemap.
3. **Скрытые encode→decode round-trips**:
   - `createArrayOf(...).getArray(map/index/count)` — Java-массив сериализуется в text-литерал и парсится обратно (PgArray:206);
   - `PgStruct.getAttributes(Map)` для вложенных structs — encode в text → `decodeTextAs` (PgStruct:134-141); тот же паттерн в `PgCallableStatement` (518-524) и OUT-`getDate/getTime/getTimestamp` через `result.toString()` (PgCS:549-571);
   - `PgArray.toString()` у binary-backed массива — полный decode + text encode.
   Каждый из них — и CPU, и потенциальные lossy-преобразования (float форматирование, timestamp-парсинг).
4. **Двойная rectangular-валидация при encode**: `estimateInitialCapacityFor` уже вызывает `computeDimensionLengths` (полный рекурсивный обход), затем `encode` вызывает его снова ([MultiDimArrayBinary.java:124,162](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java#L124)).
5. **Megamorphic dispatch на `getInt`-пути.** Старый путь был `switch(oid)` + static; новый — interface-вызов `codec.decodeAsInt` через call-site, общий для всех колонок. Обычно JIT справляется (bimorphic на колонку), но это надо мерить, а не верить. Benchmarks-модуль уже есть (`benchmarks/src/jmh`) — добавьте: `ProcessResultSet`-стиль getInt/getString по text/binary, decode `int4[]` (1K/100K элементов, `int[]` vs `Integer[]`), composite decode, и сравнение с master. Плюс `-prof gc` на alloc rate.
6. `TYPE_ALIASES` — static, process-wide, растёт без ограничения от пользовательских строк (TypeInfoCache:563-565). Негативные lookup'ы (неизвестный OID/имя) не кэшируются — повторный промах оплачивается до двух запросов каждый раз.

Где boxing **неизбежен по JDBC**, а где нет: `Array.getArray()` обязан вернуть `Object` (исторически boxed `Integer[]` — менять нельзя), но `getObject(i, int[].class)`, `ResultSet.getInt` по элементам через `Array.getResultSet`, и leaf-пути — primitive. Текущий дизайн это правильно разделяет.

---

## 6. Correctness risks

1. **`ArrayCodec.encodeText` для произвольного `java.sql.Array` делает `value.toString()`** ([ArrayCodec.java:197-200](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayCodec.java#L197)). Для `PgArray` это корректный литерал, для сторонней реализации `Array` — `com.foo.MyArray@1a2b3c` уйдёт в SQL без ошибки. То же в `setArray` (PgPS:1063-1067). Binary-ветка рядом честно делает `getArray()` — text-ветка должна так же.
2. **Binary range decode сломан для любых range-типов**: subtype берётся из `typelem`, который у range равен 0 → `getPgTypeByOid(0)` → `PSQLException` (RangeCodec:99-101 без guard, в отличие от text-пути с задокументированным fallback). Сейчас это скрыто тем, что range-OIDs не входят в default binary transfer set; первый, кто включит binary для range, получит ошибку на ровном месте.
3. **`MultiDimArraySupport.computeDimensions` считает размерность по классу** ([MultiDimArraySupport.java:28-36](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArraySupport.java#L28)). `Object[]{ Object[]{...}, ... }` (вложенность видна только в runtime) получит dims=1, и leaf-writer упадёт с «unsupported leaf» или закодирует мусорный элемент. Старый `ArrayEncoding` определял размерность по первому элементу рекурсивно. Это регрессия для `createArrayOf("int4", new Object[]{new Object[]{1,2}})`-стиля вызовов — проверьте тестом и решите сознательно (поддержать или дать ясную ошибку).
4. **Lower bounds**: decode молча отбрасывает их и в новом (MultiDimArrayBinary:195), и в старом пути; encode всегда пишет 1. Это совместимо со старым поведением, но раз уж цель — «полная поддержка PostgreSQL types», зафиксируйте решение в javadoc/spec: `getArray()` нормализует к 1-based, `getArray(index,count)` интерпретирует index как 1-based независимо от серверных bounds.
5. **`CompositeCodec` text decode молча терпит расхождение количества полей** (`Math.min(rawFields.length, expected)`, CC:571-573): лишние wire-поля отбрасываются, недостающие — null. После `ALTER TYPE ... ADD ATTRIBUTE` в другой сессии вы получите тихо искажённые данные вместо ошибки. Binary-путь, наоборот, вообще не сверяется с каталогом (это плюс для anonymous records, но удостоверьтесь, что mismatch с metadata где-то детектится).
6. **Парсер composite-текста не учитывает вложенные скобки вне кавычек** (CC:427-432 сканирует до запятой). Сервер всегда квотит вложенные значения, так что для серверного вывода это ок, но `PgStruct.getValue()`-литералы, собранные клиентом, и пользовательский ввод могут быть беззащитны — стоит fuzz-тест против `row(...)::text` сервера.
7. **`DomainCodec` теряет domain identity и typmod**: base codec получает `PgType` базового типа (DC:74-86), `typtypmod` домена никуда не передаётся. Для `numeric`-доменов с precision это пока не влияет (encode не применяет typmod), но решение «domain прозрачно разворачивается» стоит зафиксировать как контракт — включая `getObject`: пользователь домена получает Java-тип базового типа, а `DISTINCT`-идентичность видна только в metadata.
8. **`EnumCodec` не поддерживает typeMap → Java enum** (бросает для любого класса кроме `String`) — задокументировано, но проверьте, что ошибка внятная, потому что это первое, что попробует пользователь.
9. `Int4Codec.decodeAsInt(String)` делает `data.trim()` — расхождение со старым строгим путём? Старый `PgResultSet.toInt` тоже трогал пробелы, но проверьте money/локали на parity.
10. **`getString()` для text-колонок минует codec полностью** (RS:2421-2423) — после переноса всех типов на кодеки это станет последним hardcoded-путём; учтите его в плане миграции, иначе custom text codec не увидит `getString`.

---

## 7. JDBC compatibility gaps

Закрыто хорошо: `getObject(int, Map)` с qualified/quoted ключами (regtype-нормализация), `createArrayOf`/`createStruct` через server-side резолв, `CallableStatement` OUT через codec-декод результата, `addDataType`/`PGobject` продолжают работать, updateable ResultSet идёт через кодеки.

Остаётся:

| Кейс | Состояние |
|---|---|
| `Array.getArray(Map)` на binary-backed массиве | `Driver.notImplemented` (PgArray:212-213) |
| `Array.getResultSet(Map)` | `notImplemented` (PgArray:477-479) |
| `SQLInput.readArray()` | not implemented (PgSQLInput:306-323) — а это ровно ваш сценарий `struct-of-array` через SQLData; `readBlob/readClob/readRef/readRowId` тоже |
| `setArray(сторонний Array)` | кодирует `toString()` без валидации (см. Р6-1) |
| `setObject(List)` | «Can't infer the SQL type» — допустимо, но сообщение могло бы подсказывать `createArrayOf` |
| `getObject(i, T[].class)` для text-результатов | падает там, где binary работает (parity, см. Р5) |
| Multirange | `FallbackCodec` → `PGobject` со строкой; binary получит `PGUnknownBinary` |
| `Struct` OUT-параметры в `CallableStatement` | работают через encode-text→decode round-trip — корректно, но хрупко |
| Мёртвый код | недостижимые ветки `SQLData`/`Struct` в `setObject` (PgPS:877-882) |

Принцип «или поддержать, или внятная ошибка» в целом соблюдён — формулировки ошибок конкретные (тип, поле, OID). Добавьте к ним совет («Use SQLData implementation», «register a codec via …») и debug-логирование выбора кодека (`FINEST`: oid → codec class + источник регистрации) — это закроет ваш пункт про «как объяснять, какой codec был выбран».

---

## 8. Конкретные предложения (в порядке выполнения)

**До расширения на другие типы (меняют контракты):**

1. **Зафиксировать публичный периметр.** Перенести/продублировать в `org.postgresql.api.codec`: `PgType` → интерфейс `PgTypeMeta` (oid, name как (schema,name), typtype, typmod, elementOid, arrayOid, baseTypeOid, subtypeOid, fields), `CodecContext` → интерфейс с wire-частью (charset, server params, registry lookup, type lookup) без `getConnection()`. JDBC-полиси (prefersJavaTime*, convertBooleanToNumeric) — во внутренний наследник/композицию. Lazy-`PgArray`-кодеки могут жить в internal-слое адаптеров, а не в SPI.
2. **Добавить slice в decode-сигнатуры сейчас**: `decodeBinary(byte[] data, int off, int len, PgType, ctx)` с default-делегацией на полный буфер (совместимо), контейнерные кодеки переводятся на slice-вызовы. Ownership задокументировать: borrowed view, валиден только в течение вызова, копируй если сохраняешь.
3. **Спроектировать text read leaf**: общий курсорный токенизатор literal-формата (массив/composite/range) + `readLeafText` в `ArrayLeafCodec`. Без этого «убрать ArrayDecoding» не случится.
4. **Registry: scoped-регистрация и приоритеты.** Ключ — OID или (schema|null, name); порядок: explicit connection > connection-name > SPI > builtin; конфликт двух SPI на один ключ — warning в лог + детерминированный выбор (например, по классу-имени). `registerByName` либо приватизировать, либо инвалидировать `oidCache`. SPI грузить с явным classloader и логировать ошибки.
5. **`PgRangeType`/загрузка `pg_range`** (+ multirange хотя бы в metadata), починить binary range (передавать резолвнутый `PgType` в `getBinaryCodec(oid, pgType)`).

**Параллельно/после (не меняют контрактов):**

6. Кэшировать `CodecContext` на соединении; убрать per-element `getCodecContext()` в `MappedTypeObjectArrayDecoder`.
7. Убрать двойную rectangular-валидацию; кэшировать `GenericArrayLeafCodec` по element-OID в registry.
8. Починить `encodeText(Array)`/`setArray` для сторонних `Array` (через `getArray()`); решить судьбу `computeDimensions` для `Object[]`-nested.
9. Parity-harness: для каждого типа property-тест «random value → text-encode → server roundtrip → binary-decode == text-decode == исходное», плюс lower-bounds/null/empty/ragged/0-dim матрица. Это входной билет на удаление `ArrayEncoding`/`ArrayDecoding`.
10. JMH: `getInt`/`getString` text+binary, `int4[]` decode (`int[]`/`Integer[]`/`getArray()`), composite decode/encode, против master. Контролировать alloc rate (`-prof gc`).
11. Удалить пустой `TypeInfoCacheLoader`, мёртвые ветки в `setObject`, ограничить рост `TYPE_ALIASES` (bounded или per-connection).

**Путь миграции старых классов** — ваш план «оставить, постепенно перевести, удалить» работает; промежуточный шаг «сделать `ArrayEncoding` thin wrapper над кодеками» я бы пропустил: дешевле переключать registry с `ArrayCodec.INSTANCE` на `ArrayLeafStreamingCodec(name, leaf)` по мере готовности leaf-кодеков (как уже сделано для `_int4`), а `ArrayDecoding` умрёт вместе с text-reader'ом и переводом `PgArray` на кодеки. Observable-поведение, которое надо сохранить: boxed-типы из `getArray()`, 1-based индексация slice, `getBaseTypeName()`-формат, тексты ошибок там, где на них завязаны тесты (`ArrayEncoding` produces «the same error message it always has» — вы это уже учитываете в [ArrayCodec.java:126-131](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayCodec.java#L126)).

---

## 9. Вопросы, которые стоит решить до расширения

1. Какой минимальный slice/view-контракт decode вы готовы зафиксировать в public API: `(byte[], off, len)` или `ByteBuffer`? (Влияет на все кодеки, решается один раз.)
2. Identity кодека при регистрации: достаточно ли пары (schema-optional name | OID) + источник, или нужен предикат по `PgType` (для PostGIS, где OID per-database)?
3. Контракт доменов: «прозрачно разворачиваем, identity только в metadata» — да/нет? (Сейчас де-факто да; зафиксировать.)
4. `Array.getArray()` после миграции: остаётся boxed `Integer[]` навсегда (совместимость), primitive — только через `getObject(i, int[].class)`?
5. Судьба `getString`/`getDate`/`getTimestamp` hardcoded-путей: переводить на кодеки в этой ветке или объявить вне scope?
6. Multirange: metadata-only в первой версии (Fallback на чтение) или полный codec?
7. SPI: оставить static per-classloader (и честно задокументировать) или сделать честный per-Driver/per-DataSource scope, как записано в решениях V2?
8. Нужен ли `CodecContext` snapshot серверных параметров (server_version, integer_datetimes, DateStyle) уже сейчас — ради будущего offline-режима?

Если по пунктам 1-4 раздела 8 решения совпадут с моими предложениями — дальше имеет смысл спуститься на уровень code findings по конкретным файлам (у меня уже размечены кандидаты: `RangeCodec`, `CompositeCodec` text-парсер, `MultiDimArraySupport.computeDimensions`, registry-инварианты).
