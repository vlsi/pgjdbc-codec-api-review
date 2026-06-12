**Вердикт**

Дизайн жизнеспособен, но масштабировать его на остальные типы пока рано. Направление правильное: scalar codec становится источником правды, `MultiDimArray*` отделяет форму массива от leaf-кодирования, composite codec уже строится поверх field codecs. Но перед расширением на `int8`, `text`, `uuid`, temporal, ranges и сторонние codecs нужно поправить границы публичного API, registry/type identity и decode hot path.

**Главные риски**

1. Public API пока тянет internal/JDBC слой.
   `BinaryCodec` принимает `PgType` и `CodecContext` из `org.postgresql.jdbc` [BinaryCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/BinaryCodec.java#L60), а `CodecContext` отдаёт `BaseConnection` и `TypeInfo` [CodecContext.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java#L275). Для stable SPI лучше ввести public `TypeDescriptor`, `CodecSession`/`EncodingContext` и не раскрывать `BaseConnection`, `TypeInfoCache`, `TimestampUtils`.

2. Type identity недостаточна.
   Registry хранит codecs по простому имени `String` [CodecRegistry.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L83), а OID lookup берёт только `pgType.getTypeName().getName()` [CodecRegistry.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L410). Это ломает сценарии с одинаковым `typname` в разных schemas. Нужен ключ: OID для connection-bound runtime, плюс schema-qualified name для регистрации и diagnostics.

3. Registry overlay смешан с built-ins.
   `registerCustomCodec` перезаписывает запись в той же map, а `unregisterCustomCodec` удаляет имя без восстановления built-in/SPI codec [CodecRegistry.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L350). Нужны отдельные слои: user override, explicit OID, SPI, built-in, fallback. Конфликты SPI сейчас фактически last-wins и молча.

4. Range/multirange metadata ещё не готова.
   `RangeCodec` пытается взять subtype из `type.getTypelem()` [RangeCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L99), но сам код признаёт, что subtype живёт в `pg_range` и пока не загружается [RangeCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L275). Без `pg_range`/`pg_multirange` descriptors нельзя закрыть `range-of-custom-type` и multirange.

5. Arrays: новая архитектура хороша для encode, но read path ещё старый.
   `MultiDimArrayBinary` пишет header и делегирует leaf loop, это удачная форма [MultiDimArrayBinary.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java#L114). Но lower bound всегда `1` [MultiDimArrayBinary.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java#L132), а `PgArray` всё ещё не поддерживает map для binary arrays [PgArray.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgArray.java#L210). Для полной замены `ArrayDecoding` нужна value-модель с dimensions, lower bounds, element descriptor и type-map aware binary decode.

6. Decode без копий пока не заложен.
   Generic array decode делает `Arrays.copyOfRange` на каждый элемент [GenericArrayLeafCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java#L116). Composite/range делают похожие копии. Streaming encode есть, но нужен и `RawBinaryValue`/slice API для decode: `byte[] + offset + length` или `ByteBuffer` с явной ownership model.

**Оценка `Int4ArrayLeafCodec`**

Как POC он удачный: primitive `int[]` не boxing-heavy, `Integer[]` корректно несёт nulls, multi-dimensional shape вынесен в общий код. Масштабировать можно, но не копипастой по типам. Лучше сделать внутренний шаблон для fixed-width primitive leaves: `int2/int4/int8/float4/float8/bool/date/time/timestamp/uuid`, а variable-width типы (`text`, `numeric`, `bytea`, composite, range) пустить через generic/streaming path.

**Что поменять до расширения**

- Ввести public `TypeDescriptor` с kind: scalar, array, domain, enum, composite, range, multirange; включить OID, qualified name, typmod, array OID, element/base/subtype metadata, composite attrs, `typdelim`.
- Заменить `Codec.getTypeName()` как основной SPI на `CodecProvider`/`CodecFactory`: names, Java classes, capabilities, priority, replace policy. На connection-bind этапе провайдер резолвится в OID bindings.
- Разделить registry layers и явно описать conflict rules: user explicit OID > user qualified name > SPI > built-in > fallback. Duplicate SPI должен давать понятную ошибку или deterministic priority.
- Сделать raw standalone API: `RawValue(format, type, bytes/view, ownership)`, `CodecSession`, `encode(value, type, preferredFormat)`, `decode(raw, targetClass)`.
- Перевести `PgArray` read path на новый array codec, включая binary `getArray(map)`, `getResultSet(map)`, slices и lower bounds.
- Добавить `pg_range`/`pg_multirange` loading до работы над range/multirange codecs.
- Зафиксировать contract: codecs cached in `Field`, поэтому `registerCodec()` влияет на будущие queries/result sets, не на уже инициализированные fields [Field.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/core/Field.java#L196), [PgResultSet.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgResultSet.java#L3326).
- Закрыть JDBC gaps: foreign `Array` in `setArray` сейчас зависит от `toString()` [PgPreparedStatement.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgPreparedStatement.java#L1063); `CallableStatement.registerOutParameter(String, ...)` всё ещё not implemented [PgCallableStatement.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgCallableStatement.java#L620).

Тесты не запускал: это был архитектурный проход по текущему коду и repro tests.
