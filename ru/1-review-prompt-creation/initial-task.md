# Исходная постановка задачи

Это исходная постановка, с которой началось составление prompt для architecture review Codec API. На этом этапе задача уже содержала инженерный контекст, но ещё не фиксировала тип ревью, границы публичного API и полный scope PostgreSQL types.

```text
Я разрабатываю поддержку struct/array/sqldata/sqlinput/sqloutput/Struct/Array типов в pgjdbc.
Раньше поддерживались ограниченное количество массивов, и очень плохо поддерживались custom types.
Не было поддержки бинарного протокола для custom types, не было поддержки array-of-struct и т.п.

Основные цели:

* поддержка произвольных форматов, произвольных структур
* поддержка бинарного и текстовых write formats для всех типов данных, включая user-defined-structs
* эффективная работа для примитивов (например, int[] по возможности должно избегать аллокации Integer)
* избежание дублирования кода. Например, кодирование byte[] (timestamptz) -> OffsetDateTime должно выполняться в каком-то одном классе, и переиспользоваться, а не дублироваться в разных частях кода. Аналогично, кодирование и декодирование int4 должно быть один раз и переиспользоваться в остальных
* хочу поддерживать максимально странные запросы пользователей. Если JDBC API не запрещает вызовы какого-то типа, то pgjdbc должно их поддерживать

Сейчас фокус на struct, array, user-defined types -- Codec API.

Я уже показывал предварительную версию пользователям, и Lukas Eder нашёл такие недочёты:

* Lack of Array support
* Lack of CallableStatement support
* No support for qualified or quoted identifiers in Map<String, Class<?>> arguments (e.g. ResultSet.getObject(int, Map)). Qualification support is necessary, quoted identifiers probably optional?

Я их проанализировал и начал работать над поддержкой массивов.
Мне не нравятся старые классы ArrayEncoding / ArrayDecoding (похоже, ими неудобно поддержать array-of-struct-of-array), и как замену я разработал GenericArrayLeafCodec, MultiDimArrayBinary, MultiDimArrayText, MultiDimArraySupport, Int4ArrayLeafCodec

Проанализируй подходит ли получившийся дизайн Int4ArrayLeafCodec и т.п. обвязки multi-dim-array, и можно ли масштабировать его на другие типы массивов (int8, int2, ...) -- хотел сначала попробовать на одном и уже потом расширять.

Какие ещё архитектурные изменения стоит сделать, чтобы приблизиться к цели "полной и эффективной (low latency, high throughput, low cpu) поддержке arrays, structs, custom types".

Если хочешь запустить какие-то тесты (чтобы, например, понять поддерживается или нет тот или иной вариант использования API) можешь запустить postgresql-сервер через docker (см. docker/postgres-server/docker-compose.yml)
```

