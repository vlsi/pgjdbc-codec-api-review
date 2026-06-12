**1. Краткий вердикт**

Самый полезный ответ: `fable5.md`. Он покрывает больше исходной рубрики: public SPI, registry, arrays, text/binary parity, JDBC gaps, migration, performance и тесты. Большая часть его high/blocker claims подтверждается кодом.

Самый доказательный ответ: тоже Fable, с небольшим отрывом от Opus. У Fable больше проверяемых ссылок на конкретные классы. Opus полезнее в двух точках: `domain-over-array` из-за порядка `resolveByTyptype()` и идея свернуть array-дизайн к одному `ArrayCodec` плюс fast-path SPI.

GPT 5.5 — самый краткий и в целом точный executive summary, но он пропускает много edge cases и почти не доказывает claims.

Модели в основном согласны по пяти проблемам: public SPI протекает internal-типами; decode path не имеет slice/view API; array migration ещё зависит от `ArrayEncoding`/`ArrayDecoding`; registry/type identity недостаточно слоистые; range/multirange metadata не готова.

Расходятся они главным образом в оценке зрелости array path: Fable осторожнее и точнее по legacy-read пути, Opus сильнее предлагает каноническую модель, GPT остаётся на уровне правильных, но широких рекомендаций.

**2. Методика**

Сравнивались:

- [design-review-prompt.md](../1-review-prompt-creation/design-review-prompt.md)
- [fable5.md](../2-review-execution/fable5.md)
- [opus48.md](../2-review-execution/opus48.md)
- [gpt55.md](../2-review-execution/gpt55.md)

Проверялись текущие файлы рабочего дерева: codec SPI, `CodecContext`, `CodecRegistry`, `TypeInfoCache`, `PgType`, `IdentifierNormalizingTypeMap`, `ArrayCodec`, `ArrayLeafStreamingCodec`, `MultiDimArray*`, `PgArray`, `ArrayDecoding`, `CompositeCodec`, `PgStruct`, `PgSQLInput*`, `PgSQLOutput*`, `RangeCodec`, `DomainCodec`, `EnumCodec`, `PgResultSet`, `Field`, `PgPreparedStatement`, `PgCallableStatement`.

Использовал `fff` для поиска и `ast-grep` для структурной проверки ключевых методов, в частности `CodecRegistry.resolveByTyptype()` и `PgArray.getArrayImpl(...)`.

Не запускал Gradle, PostgreSQL и JMH. Поэтому claims про «45/45 tests», фактическую binary/text parity на сервере и performance numbers отмечены как непроверенные.

**3. Матрица claims**

| ID | Claim | Категория / серьёзность | Fable | Opus | GPT | Проверка по коду | Вердикт |
| --- | --- | --- | --- | --- | --- | --- | --- |
| C01 | Public SPI зависит от `org.postgresql.jdbc.*` / `core.*` | public SPI / blocker | да | да | да | `BinaryCodec` импортирует `CodecContext`/`PgType`; `CodecContext` отдаёт `BaseConnection`, `TypeInfo`, `Encoding` | confirmed |
| C02 | Offline/standalone API архитектурно заблокирован текущим `CodecContext` | standalone / high | да | да | частично | connectionless constructor package-private, registries null, `withTypeMap()` бросает | confirmed |
| C03 | Registry keyed by simple type name, schema collisions возможны | type identity / high | да | да | да | `Codec.getTypeName()` без схемы; `codecsByName: Map<String, Codec>` | confirmed + trade-off |
| C04 | Registry layers/override rules недостаточны | registry / high | да | да | да | SPI, built-in и custom попадают в одни maps; duplicate SPI last-wins | confirmed |
| C05 | `registerByName`/`registerAlias` не инвалидируют OID cache | registry / medium | да | нет | нет | методы пишут в map без `oidCache.invalidateAll()` | confirmed |
| C06 | `unregisterCustomCodec` не восстанавливает built-in/SPI codec | registry / medium | нет | нет | да | удаляет имя из `codecsByName`; отдельного слоя нет | confirmed |
| C07 | SPI default Java class может перекрыть built-in class mapping | registry / high | нет | да | нет | `registerByClass(codec.getDefaultJavaType(), codec)` после built-ins | confirmed |
| C08 | Нет explicit codec capabilities | registry / high | да | да | да | формат выводится через `instanceof BinaryCodec/TextCodec` и `binaryTransferSend()` | confirmed |
| C09 | Decode API без slice вызывает копии | performance / blocker | да | да | да | `GenericArrayLeafCodec.copyOfRange`, `CompositeCodec.new byte[]`, `RangeCodec.copyOfRange` | confirmed |
| C10 | Streaming encode/backpatch — удачная часть дизайна | performance / medium | да | да | нет | `BackpatchByteArrayOutputStream`, `reserveInt32`, container streaming paths | confirmed |
| C11 | Backpatch capability не оформлена как public API | public SPI / medium | нет | да | нет | `BackpatchingBinarySink` package-private; public API ссылается на jdbc codec class | partially confirmed |
| C12 | Leaf/walker decomposition для arrays жизнеспособна | array / medium | да | да | да | `MultiDimArrayBinary/Text` отделяют shape от leaf loop | design trade-off |
| C13 | Новый array read path ещё сосуществует с legacy | migration / high | да | да | да | `decodeBinary()` возвращает `PgArray`; `PgArray` читает через `ArrayDecoding` | confirmed |
| C14 | В новой array архитектуре нет text read leaf | array / high | да | да | нет | `ArrayLeafCodec` имеет binary read/write и text write; `MultiDimArrayText` encode-only | confirmed |
| C15 | `Int4ArrayLeafCodec` дублирует scalar int4 logic | maintainability / medium | нет | да | нет | leaf сам пишет/читает int4 и зовёт `Int4Codec.toInt`, но не общий scalar encode/decode | confirmed |
| C16 | Общего primitive encode fast path в public SPI нет | performance / medium | частично | да | нет | decode specializations есть; encode specializations нет, кроме private leaf POC | confirmed |
| C17 | `computeDimensions()` ломает runtime-nested `Object[]{Object[]{...}}` | array / medium | да | нет | нет | размерность считается по class, не по элементам | confirmed |
| C18 | Lower bounds теряются / encode всегда пишет `1` | array / medium | да | да | да | `MultiDimArrayBinary` пишет `1`; decode читает и игнорирует lower bound | confirmed + trade-off |
| C19 | `PgArray.getArray(Map)` не работает для binary arrays | JDBC / high | да | спорно | да | binary branch throws `notImplemented` with map | confirmed |
| C20 | `PgArray.getResultSet(Map)` не реализован | JDBC / high | да | спорно | нет | `getResultSetImpl(..., Map)` throws `notImplemented` | confirmed |
| C21 | `setArray` для foreign `Array` использует `toString()` | JDBC / high | да | нет | да | `PgPreparedStatement.setArray` comments admit limitation | confirmed |
| C22 | `SQLInput.readArray()` отсутствует | JDBC / high | да | нет | нет | `PgSQLInput.readArray()` throws not implemented | confirmed |
| C23 | `SQLOutput.writeArray()` есть | JDBC / low | нет | нет | нет | `PgSQLOutput.writeArray()` delegates to codec | confirmed, useful correction |
| C24 | Composite binary decode copies field bytes | performance / medium | да | нет | нет | `CompositeCodec.decodeBinaryFields()` allocates per field | confirmed |
| C25 | Composite text decode tolerates field-count mismatch silently | correctness / high | да | нет | нет | uses `Math.min(rawFields.length, expected)` | confirmed |
| C26 | Dropped attributes not filtered | composite / high | нет | да | нет | `TypeInfoCache.loadCompositeFields()` has `NOT a.attisdropped` | false / hallucinated |
| C27 | Range subtype metadata missing; binary range broken | ranges / blocker | да | частично | да | `RangeCodec` uses `type.getTypelem()`; text path comments `pg_range` not loaded | confirmed |
| C28 | Multirange unsupported | multirange / high | да | да | да | `resolveByTyptype()` has no `typtype == 'm'` branch | confirmed |
| C29 | Domain identity/typmod is lost through base codec delegation | domains / medium | да | частично | частично | `DomainCodec` passes base `PgType` to base codec | confirmed |
| C30 | `domain-over-array` likely resolves as array before domain | domains/arrays / high | нет | да | нет | code checks `isArray()` before `isDomain()`; PG catalog behavior not experimentally checked | partially confirmed |
| C31 | Enum decode does not map to Java enum through type map | enum / medium | да | нет | нет | `decode*As` accepts only `String`/`Object`; encode accepts `Enum` | confirmed |
| C32 | `getString()` text path bypasses codec | scalar / medium | да | нет | нет | text branch decodes connection encoding directly | confirmed |
| C33 | `CodecContext` allocated repeatedly | performance / low | да | нет | нет | `PgConnection.getCodecContext()` returns new instance each call | confirmed |
| C34 | `Field` caches codec once | registry / medium | да | нет | да | `Field.initializeCodec()` no-ops if already initialized | confirmed |
| C35 | `IdentifierNormalizingTypeMap` handles qualified/quoted `Map` keys | JDBC / high | да | да | нет | resolves lookup and user keys through `regtype` OID | confirmed |
| C36 | Named `CallableStatement.registerOutParameter(String, …)` not implemented | JDBC / medium | нет | частично | да | string overloads throw `notImplemented` | confirmed |
| C37 | Positional Callable OUT exists and goes through `ResultSet.getObject` | JDBC / medium | да | да | нет | `executeWithFlags()` fills `callResult[j] = rs.getObject(...)` | confirmed |
| C38 | `setObject(dto)` does not use Java-class codec lookup | JDBC/SPI / high | нет | да | нет | `findCodecFor` exists, but untyped `setObject` does not call it | confirmed |
| C39 | Test count / “45/45” proves maturity | tests / low | нет | да | нет | tests not run in this comparison | unclear |
| C40 | Simple-name registration is “wrong” rather than a trade-off | type identity / medium | implied | implied | implied | code confirms ambiguity, but unqualified names are useful with `search_path` | design trade-off |

**4. Подтверждённые общие выводы**

Все или почти все модели правильно нашли, что public SPI нельзя стабилизировать в текущем виде. `org.postgresql.api.codec.BinaryCodec` и `TextCodec` принимают `PgType` и `CodecContext` из `org.postgresql.jdbc`, а `CodecContext` отдаёт `BaseConnection`, `TypeInfo`, `CodecRegistry`, `JavaTypeRegistry` и `Encoding`. Это blocker для публичного SPI и для offline API.

Все три модели правильно указывают на отсутствие decode slice/view API. Это уже создаёт копии в arrays, composites и ranges, а позже будет дороже исправляться: менять придётся сигнатуры всех container/scalar decode paths.

Все модели сходятся, что array migration не завершена. Новый `Int4ArrayLeafCodec` полезен, но обычный `PgArray.getArray()` всё ещё идёт через `ArrayDecoding`, а text read leaf в новой модели отсутствует.

Все модели правы по range/multirange: `RangeCodec` не имеет subtype metadata из `pg_range`, binary path сейчас берёт `typelem`, а multirange не имеет registry dispatch.

Fable и Opus правильно считают `IdentifierNormalizingTypeMap` сильной частью текущего кода: feedback про qualified/quoted identifiers для `ResultSet.getObject(int, Map)` в основном закрыт через `regtype`/OID normalization.

**5. Уникальные полезные findings**

Fable:

- `registerByName` и `registerAlias` не инвалидируют OID cache, в отличие от `registerCustomCodec`.
- `ArrayCodec.encodeText(Array)` и `PgPreparedStatement.setArray(foreign Array)` зависят от `Array.toString()`.
- `CompositeCodec.decodeTextAsStruct()` молча терпит mismatch числа полей.
- `getString()` для text values остаётся hardcoded path вне codec model.
- `PgConnection.getCodecContext()` создаёт новый context, а legacy `ArrayDecoding` может делать это per element для mapped custom types.

Opus:

- `resolveByTyptype()` проверяет array до domain. Для `domain-over-array` это вероятный correctness bug.
- `registerByClass(codec.getDefaultJavaType(), codec)` для SPI может перекрыть built-in `String`/`Object`.
- `setObject(dto)` не использует `CodecRegistry.findCodecFor()`.
- План “один универсальный `ArrayCodec` + optional fast-path на scalar codec” лучше масштабируется, чем отдельный public-ish codec на каждый element OID.

GPT:

- Чётко формулирует проблему registry layers: `unregisterCustomCodec()` удаляет custom entry, но не восстанавливает скрытый built-in/SPI codec.
- Правильно выделяет contract для `Field` codec cache: registry changes не должны обещать влияние на уже инициализированные result-set fields.
- Хорошо формулирует будущий standalone shape через `RawValue(format, type, bytes/view, ownership)`.

**6. Сомнительные или ложные claims**

Ложный claim Opus: dropped attributes в composites якобы не фильтруются. В текущем коде [TypeInfoCache.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/TypeInfoCache.java#L859) загружает поля с `AND NOT a.attisdropped`.

Сомнительный claim Opus: `getArray(Map)`, `getResultSet`, `getArray(Map)` “covered”. Срезы и обычный `getResultSet()` есть, но [PgArray.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgArray.java#L213) бросает `notImplemented` для binary `getArray(Map)`, а [PgArray.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgArray.java#L478) бросает `notImplemented` для `getResultSet(Map)`.

Overstated claim Opus: third-party streaming codec “always” gets slow path because `BackpatchingBinarySink` package-private. Проблема реальна как API boundary leak, но не абсолютная: `BackpatchByteArrayOutputStream` public, и сторонний код технически может на него завязаться. Просто это будет зависимость от internal `org.postgresql.jdbc.codec`.

Частично подтверждённый claim Fable/Opus: simple-name codec registration “ошибочна”. Факт: schema collision возможен. Но это не чистый баг, а trade-off: unqualified name удобен и следует `search_path`, OID точен connection-bound, schema-qualified name точен для пользователя, но требует явной привязки. Практическая рекомендация: поддержать все три формы, но сделать conflict rules явными.

Непроверенный claim Opus: “45/45 tests”. Я тесты не запускал, поэтому это остаётся неподтверждённым.

**7. Пропущенные темы**

Проблемы, пропущенные всеми моделями:

- `MultiDimArrayBinary.decode()` игнорирует element OID из binary header. Код читает OID и отбрасывает его, полагаясь на caller-known leaf. Для обычного trusted ResultSet это терпимо, но для standalone/raw decode и malformed payload стоит валидировать mismatch.
- Binary decoders почти не проверяют trailing bytes. `CompositeCodec.decodeBinaryFields()` не проверяет, что `ByteBuffer` исчерпан после заявленного числа полей; `RangeCodec.decodeBinary()` не проверяет trailing bytes после upper bound. Это относится к исходной рубрике “malformed binary payload errors”.
- `PgSQLInputBinary`/`PgSQLInputText` pre-cache codecs через `castNonNull(ctx.getCodecs().get*Codec(...))`. Если type dispatch вернул codec без нужного format, ошибка будет не такой понятной, как у container encode paths. Это ещё один симптом отсутствия capabilities.

Темы из исходной задачи, покрытые поверхностно:

- explicit ownership model для raw binary/text buffers: borrowed view, copy, reusable scratch.
- exact `simple query mode` vs extended query mode policy для custom codecs.
- compatibility contract для `PGobject`/`Connection.addDataType` при появлении Codec SPI.
- typmod propagation: `PgField` хранит typmod, но codec API фактически работает с `PgType`, а не с field-level typmod.
- concrete migration test matrix для удаления `ArrayEncoding`/`ArrayDecoding`.

**8. Сравнение планов**

Лучший общий план: Fable, потому что он ставит contract-changing задачи до тиражирования кодеков: public boundary, decode slice, text array reader, scoped registry, range metadata. Это лучше снижает риск зацементировать неудачный SPI.

Лучшее отдельное архитектурное предложение: Opus P4 — один универсальный `ArrayCodec`, который композиционно использует element codecs, плюс optional fast-path interface на scalar codec. Это лучше отвечает на вопрос “как быстро и масштабируемо добавить `int2/int8/float/bool/text/uuid/...` без копипасты leaf codecs”.

План GPT полезен как короткая дорожная карта, но ему не хватает проверочных тестов, order-of-work и деталей миграции старого array path.

Что взять:

- из Fable: public boundary, decode slice, text tokenizer/read leaf, range metadata, parity/JMH план;
- из Opus: unified array model + scalar fast-path, fix `resolveByTyptype` precedence, SPI conflict handling;
- из GPT: registry layers и `RawValue`/standalone API как публичная форма.

Что отбросить или отложить:

- не пытаться сразу переписать весь driver на codecs;
- не делать отдельный public array codec на каждый element OID;
- не фиксировать `PgType`/`CodecContext` как stable API;
- не требовать full multirange implementation до metadata layer, но не оставлять silent fallback без понятного статуса.

**9. Решения перед масштабированием**

| Решение | Почему блокирует масштабирование | Варианты | Предпочтительно сейчас | Можно отложить |
| --- | --- | --- | --- | --- |
| Public SPI boundary | После сторонних codecs ломать API будет дорого | оставить `PgType/CodecContext`; ввести `TypeDescriptor`/public context | ввести public interfaces, internal adapters | полноценный offline runtime |
| Type identity | simple name ломает schema collisions | OID; unqualified name; schema-qualified; predicate | OID for connection-bound + schema-qualified registration + unqualified aliases | advanced predicates |
| Registry layers | иначе override/unregister/cache semantics неясны | one map; layered maps; priority numbers | explicit user OID > user qualified > SPI > built-in > fallback | dynamic rebinding existing fields |
| Decode buffer API | без него будут копии во всех container codecs | `byte[]`; `(byte[], off, len)`; `ByteBuffer`; `RawValue` | borrowed `(byte[], off, len)` plus future `RawValue` | streaming visitor decode |
| Array architecture | иначе размножатся leaf codecs | per-OID leaf codec; universal array codec; hybrid | universal `ArrayCodec` + optional scalar fast-path | all primitive specializations |
| Text parsing | без text read нельзя удалить `ArrayDecoding` | keep legacy; shared tokenizer | shared cursor/tokenizer for array/composite/range | full legacy deletion |
| Domain semantics | affects domain-over-array and metadata | unwrap always; preserve descriptor + delegate base codec | preserve domain descriptor, reuse base codec for value | client-side domain constraints |
| Range/multirange metadata | range-of-custom-type невозможен без subtype | text-only fallback; load `pg_range`; full codec | load `pg_range`/multirange metadata first | full multirange encode/decode |
| JDBC coverage boundary | public API must be predictable | support all; clear unsupported errors | fix `SQLInput.readArray`, binary `Array.getArray(Map)`, foreign `Array`, named Callable | LOB/Ref/RowId |
| Format capabilities | binary/text choice сейчас implicit | `instanceof`; capability enum | explicit read/write capabilities per format | advanced negotiation policy |

**10. Рекомендуемый следующий шаг**

Ближайшая итерация:

1. Зафиксировать маленький API sketch: `TypeDescriptor`, public `CodecContext`, decode slice, format capabilities, registry layers.
2. Прототипировать это на `int4[] + composite[]`: убрать per-element copy в generic array/composite decode и сохранить текущий `int[]` fast path.
3. Добавить regression tests для `domain-over-array`, binary `Array.getArray(Map)`, foreign `Array.setArray`, binary range explicit failure/metadata, и `SQLInput.readArray`.
4. После этого переносить следующий scalar (`int8` или `bool`) не как копию `Int4ArrayLeafCodec`, а через выбранный fast-path interface.
