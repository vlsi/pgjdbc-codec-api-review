# Prompt: design review Codec API для arrays, structs и user-defined types в pgjdbc

Я разрабатываю поддержку `struct` / `array` / `sqldata` / `sqlinput` / `sqloutput` / `Struct` / `Array` типов в pgjdbc.

Раньше pgjdbc поддерживал ограниченное количество массивов и слабо поддерживал custom types:

* не было полноценной поддержки binary protocol для custom types;
* плохо поддерживались user-defined composite types;
* не было нормальной поддержки `array-of-struct`, `struct-of-array` и похожих вложенных типов;
* код encoding/decoding был размазан по разным классам, из-за чего одни и те же преобразования дублировались.

Сейчас фокус на публичном Codec API: он должен стать единой моделью кодирования и декодирования PostgreSQL values. pgjdbc должен использовать этот API внутри JDBC операций, а сторонние библиотеки должны иметь возможность расширять его своими кодеками.

## Главные цели

Нужно приблизиться к полной и эффективной поддержке PostgreSQL types:

* built-in scalar types;
* arrays;
* composite/user-defined struct types;
* domains;
* enums;
* ranges;
* multiranges;
* произвольных вложенных комбинаций этих типов.

Важные сценарии:

* `array-of-struct`;
* `struct-of-array`;
* `array-of-struct-of-array`;
* `array-over-domain`;
* `domain-over-array`;
* `range-of-custom-type`;
* `multirange`;
* custom codec для внешней библиотеки, например PostGIS codec для geometry/geography типов.

JDBC adapters (`Array`, `Struct`, `SQLInput`, `SQLOutput`, `ResultSet.getObject`, `CallableStatement`) должны быть thin adapters поверх Codec API, а не отдельными реализациями парсинга и кодирования.

## Публичный Codec API / SPI

Codec API задуман как публичный API, а не как внутренний implementation detail.

Нужно заложить возможность, что сторонняя библиотека:

* реализует codec для своих PostgreSQL типов;
* добавляет codec в classpath;
* регистрирует codec через `ServiceLoader` или явную регистрацию;
* декодирует и кодирует значения через стабильные публичные абстракции;
* не зависит от внутренних классов pgjdbc вроде `PGStream`, `QueryExecutor`, `Field`, `Tuple`, `ArrayEncoding`, `ArrayDecoding`.

Проверь, достаточно ли хорошо дизайн разделяет:

* stable public SPI;
* connection-bound runtime context;
* type metadata / registry;
* internal protocol implementation;
* JDBC API adapters.

Отдельно оцени:

* как должны регистрироваться сторонние кодеки;
* как решать конфликты нескольких кодеков для одного типа;
* нужны ли приоритеты, override rules, fallback;
* какие interfaces/classes можно стабилизировать как public API;
* какие детали должны остаться internal.

## Standalone encode/decode API

В дизайн нужно заложить публичный standalone API для кодирования и декодирования значений в PostgreSQL wire representation и обратно.

API должен позволять стороннему коду:

* получить сырое значение в binary или text format без немедленного декодирования;
* декодировать raw binary/text value в Java object через зарегистрированный codec;
* закодировать Java object, например `CustomDto` или `CustomDto[]`, в binary/text payload, пригодный для отправки в PostgreSQL;
* делать это через стабильные публичные абстракции.

Различай два режима:

1. Connection-bound encode/decode.
   API работает рядом с `PGConnection` и использует catalog metadata, server settings, codec registry и type cache конкретного соединения.

2. Offline encode/decode.
   API работает без живого соединения, но вызывающий код обязан явно передать `TypeDescriptor` / `CodecRegistry` / server encoding context / нужные metadata.

Для первой версии connection-bound API можно считать обязательным. Offline API можно отложить, но архитектура не должна его блокировать.

Проверь:

* какие type metadata нужны для standalone encode/decode: OID, type name, schema, typmod, array OID, element OID, composite attributes, range subtype, domain base type;
* как API получает metadata: из живого `Connection`, из `TypeRegistry`, из явно переданного `TypeDescriptor`;
* как представлять binary/text format: `byte[]`, `ByteBuffer`, `InputStream`, reader/writer API;
* кто владеет буфером: borrowed view, immutable copy, reusable scratch buffer;
* можно ли избежать лишних copies на hot path;
* как codec сообщает capabilities: binary read, binary write, text read, text write;
* как выбирается формат, если codec поддерживает только часть операций;
* какие ошибки API возвращает при отсутствии metadata, unsupported format, unsupported Java class;
* как не утянуть в public API детали текущей реализации протокола.

## Текущая реализация и фокус ревью

Мне не нравятся старые классы `ArrayEncoding` / `ArrayDecoding`: похоже, ими неудобно поддержать `array-of-struct-of-array` и другие вложенные случаи.

Как замену я начал разрабатывать:

* `GenericArrayLeafCodec`;
* `MultiDimArrayBinary`;
* `MultiDimArrayText`;
* `MultiDimArraySupport`;
* `Int4ArrayLeafCodec`;
* связанную обвязку multi-dimensional array support.

Сейчас я пробую дизайн на `int4` / `Int4ArrayLeafCodec`, прежде чем расширять его на `int8`, `int2`, `float4`, `float8`, `bool`, `text`, `uuid`, temporal types и другие типы.

Проанализируй, подходит ли получившийся дизайн `Int4ArrayLeafCodec` и multi-dimensional array обвязки для масштабирования на остальные типы.

Не ограничивайся вопросом "`int4[]` работает или нет". Интересует, можно ли на этой архитектуре построить всю систему:

* scalar codecs;
* array codecs;
* composite codecs;
* domain codecs;
* enum codecs;
* range codecs;
* multirange codecs;
* public codec registry;
* JDBC adapters.

## Известный внешний feedback

Предварительную версию уже смотрел Lukas Eder. Он нашёл такие недочёты:

* lack of `Array` support;
* lack of `CallableStatement` support;
* no support for qualified or quoted identifiers in `Map<String, Class<?>>` arguments, for example `ResultSet.getObject(int, Map)`.

Qualification support для type names нужен. Quoted identifiers, возможно, optional, но лучше оценить стоимость и последствия.

## Что именно нужно проверить

Сделай design review публичного Codec API, а не line-by-line code review.

Проверь:

* достаточно ли хорошо разделены public SPI и internal implementation;
* масштабируется ли дизайн `Int4ArrayLeafCodec` / `MultiDimArray*` на другие scalar и container types;
* можно ли container codecs строить композиционно поверх element / field / bound codecs;
* где дизайн создаёт лишний boxing, allocation или copying;
* какие JDBC API cases останутся неподдержанными;
* какие PostgreSQL correctness edge cases не покрыты;
* как должны работать codec lookup, type metadata cache, OID/schema/qualified-name resolution;
* как обрабатывать конфликтующие сторонние кодеки;
* что стоит изменить до расширения реализации на остальные типы.

Не спускайся в мелкий code review, пока не станет понятно, что архитектура жизнеспособна. Если дизайн неприемлем, сначала предложи архитектурные изменения.

## Архитектурные темы для анализа

### Type identity и metadata

Проверь:

* как идентифицируется PostgreSQL type: OID, schema-qualified name, quoted name, array OID, element OID;
* как работают одинаковые type names в разных schemas;
* как учитывается `search_path`;
* как обрабатываются quoted identifiers;
* как представлены domains, enums, ranges, multiranges, composites, arrays;
* сохраняется ли domain identity или domain прозрачно разворачивается в base type;
* как обрабатываются domains over arrays и arrays over domains;
* как type cache переживает `CREATE TYPE`, `DROP TYPE`, `ALTER TYPE`, смену `search_path`.

### Codec registry и lookup

Проверь:

* lookup по OID;
* lookup по qualified type name;
* lookup по Java class;
* lookup по element/field/bound type;
* fallback chain;
* приоритет пользовательских кодеков над built-in кодеками;
* поведение при неоднозначном mapping;
* поведение при unsupported type или unsupported Java class;
* thread-safety registry и codec instances;
* connection-specific state vs shared immutable codec state.

### Scalar codecs

Проверь, можно ли scalar codec использовать как единственный источник правды для:

* JDBC scalar read/write;
* array element read/write;
* composite field read/write;
* range bound read/write;
* standalone encode/decode.

Например:

* `int4` binary/text encoding не должен дублироваться в array/composite/range коде;
* `timestamptz` bytes-to-`OffsetDateTime` должен жить в одном месте;
* UUID/text/numeric/date-time codecs должны переиспользоваться всеми container codecs.

### Array codecs

Проверь:

* null array vs empty array vs array with null elements;
* multidimensional arrays;
* lower bounds;
* non-1 lower bounds;
* rectangular vs ragged arrays;
* binary array header;
* text array parsing/escaping/quoting;
* `typdelim`;
* primitive arrays: `int[]`, `long[]`, `short[]`, `boolean[]`, `float[]`, `double[]`;
* boxed arrays: `Integer[]`, `Long[]`, etc.;
* object arrays: `Object[]`, custom DTO arrays, arrays of composite objects;
* slices: `Array.getArray(long index, int count)`;
* `Array.getResultSet`;
* `Array.getArray(Map<String, Class<?>>)`;
* `createArrayOf`;
* `setArray`;
* `setObject` with arrays;
* binary/text parity.

Особенно важно оценить, где JDBC API вынуждает boxing, а где pgjdbc может сохранить primitive fast path.

### Composite / Struct codecs

Проверь:

* binary composite format;
* text composite parsing/escaping/quoting;
* null fields;
* dropped attributes;
* attribute order;
* schema-qualified type names;
* quoted type/field names;
* `Struct`;
* `SQLData`;
* `SQLInput`;
* `SQLOutput`;
* `ResultSet.getObject(int, Map<String, Class<?>>)`;
* `Connection.createStruct`;
* `CallableStatement` OUT/INOUT parameters;
* arrays of composite values;
* composites containing arrays, domains, enums, ranges, multiranges.

### Domains, enums, ranges, multiranges

Проверь:

* domain identity vs base type codec reuse;
* domain constraints and whether client-side code should know about them;
* enum labels with unusual characters;
* enum binary/text behavior;
* range empty value;
* infinite bounds;
* inclusive/exclusive bounds;
* canonicalization assumptions;
* range subtype codec reuse;
* custom range subtype;
* multirange binary/text format;
* arrays of ranges and ranges inside composites.

### JDBC API coverage

Проверь, не остаются ли вне дизайна:

* `ResultSet.getObject(int)`;
* `ResultSet.getObject(int, Class<T>)`;
* `ResultSet.getObject(int, Map<String, Class<?>>)`;
* `PreparedStatement.setObject`;
* `PreparedStatement.setArray`;
* `CallableStatement` IN/OUT/INOUT;
* `Connection.createArrayOf`;
* `Connection.createStruct`;
* `Array.getArray`;
* `Array.getResultSet`;
* `Struct.getAttributes`;
* `SQLData`;
* `SQLInput`;
* `SQLOutput`;
* `PGobject`;
* `Connection.addDataType`;
* simple query mode vs extended query mode;
* binary transfer enable/disable settings.

Если JDBC API не запрещает какой-то вызов, желательно, чтобы pgjdbc его поддерживал или выдавал понятную ошибку с объяснением ограничения.

### Performance

Особенно важны:

* low latency;
* high throughput;
* low CPU;
* низкое количество allocations;
* минимум boxing для primitive arrays;
* отсутствие лишних intermediate `Object[]`;
* отсутствие лишних `String`/`byte[]` copies;
* возможность streaming/visitor-style traversal для больших arrays/composites;
* reuse buffers там, где это безопасно;
* понятная ownership model для borrowed/copy buffers.

Проверь:

* где текущий дизайн неизбежно boxing-heavy;
* где можно добавить primitive specialization без дублирования логики;
* как не размножить код для `int2`, `int4`, `int8`, `float4`, `float8`, `bool`;
* как измерять regressions;
* какие JMH benchmarks или integration benchmarks стоит добавить;
* какие performance counters/alloc profiles стоит проверить.

### Error handling и usability

Проверь:

* понятность ошибок при unsupported type;
* понятность ошибок при ambiguous type name;
* понятность ошибок при missing codec;
* понятность ошибок при malformed array/composite/range literal;
* что пользователь увидит при unsupported binary/text format;
* как объяснять, какой codec был выбран и почему;
* нужна ли debug/tracing возможность для codec lookup.

## Compatibility и миграция

Моя конечная цель -- полностью избавиться от `ArrayEncoding` / `ArrayDecoding`.

Порядок удаления не важен: сначала можно оставить старый код, постепенно перевести остальные типы и затем удалить старые классы.

Проверь:

* какие observable behaviors старого `ArrayEncoding` / `ArrayDecoding` нужно сохранить;
* какие несовместимости допустимы;
* какие тесты должны быть перед удалением старого пути;
* какие старые extension points (`PGobject`, `Connection.addDataType`) должны продолжить работать;
* можно ли сделать старые классы thin wrappers поверх нового Codec API на переходный период.

## Тесты и эксперименты

Если нужно проверить поведение pgjdbc или PostgreSQL, можно запускать тесты и PostgreSQL server.

PostgreSQL server можно поднять через Docker:

```bash
docker compose -f docker/postgres-server/docker-compose.yml up -d
```

Если запускаешь Gradle, используй `--quiet`, чтобы не раздувать вывод.

Примеры того, что можно проверить экспериментально:

* binary/text parity для arrays/composites;
* behavior `ResultSet.getObject(int, Map<String, Class<?>>)`;
* behavior qualified/quoted identifiers;
* `CallableStatement` с custom/composite/array OUT parameters;
* `createArrayOf` и `createStruct`;
* arrays with lower bounds;
* arrays containing nulls;
* multidimensional arrays;
* domains over arrays;
* arrays over domains;
* enum/range/multirange behavior;
* performance/allocation profile для `int[]` vs `Integer[]`.

## Ожидаемый результат

Сначала дай архитектурный вердикт:

* дизайн жизнеспособен;
* дизайн жизнеспособен, но требует изменений перед масштабированием;
* дизайн не стоит масштабировать без существенной переработки.

Затем дай:

* основные архитектурные риски;
* missing abstractions;
* places where public API leaks internal implementation;
* performance risks;
* correctness risks;
* JDBC compatibility gaps;
* конкретные предложения по изменению архитектуры;
* список вопросов, которые нужно решить до расширения на остальные типы.

Если архитектура выглядит приемлемой, можно после этого перейти к code findings по конкретным файлам.

