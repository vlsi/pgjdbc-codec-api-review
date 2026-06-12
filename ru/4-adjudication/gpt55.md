**Краткий вердикт**

A = [gpt55.md](../3-comparison/gpt55.md), B = [opus48.md](../3-comparison/opus48.md).

A полезнее как судебное сравнение: claims лучше атомизированы, статусы `confirmed` / `trade-off` / `false` разделены чище, матрица полнее. B полезнее как план следующей архитектурной итерации: сильнее формулирует модель “один `ArrayCodec` + optional fast-path” и точнее заостряет `domain-over-array`.

Финальный verdict надёжен для claims, проверенных по коду. Для PostgreSQL catalog edge cases и performance claims надёжность средняя: серверные эксперименты, Gradle и JMH я не запускал.

**Матрица расхождений**

| Тема / claim | A | B | Совпадают | Нужно перепроверить | Финальный вердикт |
| --- | --- | --- | --- | --- | --- |
| Public SPI тянет `jdbc.PgType` / `CodecContext` | blocker confirmed | blocker confirmed | да | нет | confirmed |
| Offline API заблокирован `CodecContext` | confirmed | confirmed | да | нет | confirmed |
| Registry по простому имени без схемы | confirmed + trade-off | confirmed, сильнее про `search_path` | частично | нет | confirmed risk + trade-off |
| Нет явных registry layers / restore on unregister | confirmed | confirmed | да | нет | confirmed |
| `registerByName` / alias не инвалидируют OID cache | confirmed | confirmed как Fable-only | да | нет | confirmed |
| Decode без slice делает копии | confirmed | confirmed | да | нет | confirmed |
| Array read path всё ещё legacy | confirmed | confirmed | да | нет | confirmed |
| Text-read leaf для arrays отсутствует | confirmed | confirmed | да | нет | confirmed |
| Unified `ArrayCodec` + fast-path SPI | trade-off / recommendation | ключевая рекомендация | частично | прототип | trade-off, рекомендовано |
| Lower bounds теряются | confirmed + trade-off | confirmed + trade-off | да | политика | trade-off |
| `PgArray.getArray(Map)` binary / `getResultSet(Map)` | confirmed | confirmed | да | нет | confirmed |
| Foreign `Array` text encode через `toString()` | confirmed | confirmed | да | нет | confirmed |
| `setObject(dto)` не использует Java-class codec lookup | confirmed | B оставил менее проверенным | частично | нет | confirmed |
| Named `CallableStatement` overloads | confirmed | часть оставлена unverified | частично | нет | confirmed |
| `SQLInput.readArray()` отсутствует | confirmed | скорее в плане покрытия | частично | нет | confirmed |
| `Field` кэширует codec | confirmed | partially confirmed | частично | нет | confirmed |
| `getString()` text path мимо codec | confirmed | unverified | нет | нет | partially confirmed: text path да, binary path использует codec |
| Range subtype через `typelem=0`, multirange нет | confirmed | confirmed | да | серверные tests | confirmed |
| `domain-over-array` dispatch | partially confirmed | confirmed | частично | серверный repro | confirmed design risk |
| Dropped composite attributes не фильтруются | false | not reproducible | да по итогу | нет | false, не hallucination |
| “45/45 tests” как доказательство зрелости | unclear | unclear | да | Gradle/tests | unresolved |
| Новые missed topics | A: trailing bytes, element OID header | B: simple query mode, enum labels | нет | targeted review | полезные, но secondary |

**Финальные confirmed issues**

- `blocker`: public SPI протекает internal API. [BinaryCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/BinaryCodec.java#L9) импортирует `CodecContext` и `PgType`; [CodecContext.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java#L41) `final` и отдаёт `BaseConnection` / `TypeInfo`.
- `high`: registry identity и override semantics не готовы для public API. Один `codecsByName`, simple name lookup и destructive unregister видны в [CodecRegistry.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L280).
- `high`: decode API без slice/view уже создаёт копии в array/range/composite paths, например [GenericArrayLeafCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java#L116) и [RangeCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L120).
- `high`: array migration не завершена. `ArrayCodec.decodeBinary()` возвращает `PgArray`, а typed decode уходит в `ArrayDecoding`.
- `high`: JDBC gaps остаются: binary `PgArray.getArray(Map)` и `getResultSet(Map)` бросают `notImplemented`; `SQLInput.readArray()` тоже.
- `high`: range metadata не загружена из `pg_range`; binary range берёт subtype из `typelem`, который для range не является subtype.
- `medium/high`: `resolveByTyptype()` проверяет `isArray()` раньше `isDomain()`. При домене, наследующем array category, это уводит dispatch в `ArrayCodec`.

**Финальные false / hallucinated claims**

- “Dropped attributes in composites are not filtered”: false. `TypeInfoCache.loadCompositeFields()` добавляет `AND NOT a.attisdropped`.
- “`getArray(Map)` / `getResultSet(Map)` уже покрыты”: false для non-empty map в текущем коде.
- “Backpatching полностью недоступен сторонним codec”: overstated. Проблема public boundary реальна, но это не абсолютный запрет на streaming encode.

**Unresolved questions**

- Нужен серверный repro для `domain-over-array`: проверить реальные `typtype`, `typcategory`, `typelem`, `typbasetype` и encode/decode path.
- Нужен parity harness для binary/text arrays, composites, ranges, domains и enum labels.
- Нужны JMH/alloc профили: claims про round-trip, boxing и per-element context allocation без чисел остаются risk-based.
- Нужно решить policy, а не “починить баг”: lower bounds, domain identity, typmod propagation, offline API scope.

**Design trade-offs**

- Custom codec registration: OID точен в соединении, schema-qualified name переносимее, unqualified name удобен, но зависит от `search_path`. Практично поддержать все формы с явным priority.
- Domain handling: base-codec reuse прост и совместим; сохранение domain identity нужно для metadata/JDBC fidelity. Рекомендация: descriptor сохраняет domain, value codec делегирует base type.
- Primitive arrays: JDBC `Array.getArray()` часто ведёт к boxed/object path; fast path нужен для `getObject(int[].class)` и internal containers.
- Binary-only/text-only codecs: нужен explicit capabilities contract вместо `instanceof` и исключений изнутри.

**Проблемы, пропущенные обоими сравнениями**

Не добавляю новых как final findings: это уже было бы новое architecture review. Для текущего verdict достаточно missed topics, которые подняли A или B: simple query mode, trailing bytes, binary array header element OID, buffer ownership, `PGobject` / `addDataType` compatibility.

**Решения перед масштабированием**

1. Public `TypeDescriptor` и public `CodecContext` interface без `BaseConnection`.
2. Decode slice / `RawValue` / ownership model.
3. Registry layers и conflict rules: user OID, qualified name, unqualified alias, SPI, built-in, fallback.
4. Один `ArrayCodec` с optional scalar fast-path, без размножения leaf-классов.
5. Общий text tokenizer для arrays/composites/ranges.
6. Type dispatch precedence: `typtype` раньше `typcategory`.
7. `pg_range` / multirange metadata model.
8. JDBC coverage floor: `Array(Map)`, `SQLInput.readArray`, named Callable, `setObject` Java-class lookup.
9. Explicit binary/text read/write capabilities.
10. Migration gate: parity tests плюс JMH до удаления `ArrayEncoding` / `ArrayDecoding`.

**Recommended next iteration**

Сначала зафиксировать API sketch для `TypeDescriptor`, codec context, decode slice и capabilities. Затем починить dispatch precedence и registry restore/cache semantics. После этого прототипировать unified array model на `int8` или `bool`, добавить parity tests для `domain-over-array`, binary/text arrays, `Array.getArray(Map)` и range subtype.
