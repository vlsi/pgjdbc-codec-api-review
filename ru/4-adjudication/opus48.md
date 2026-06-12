Я прочитал все входные файлы: исходный prompt, три первичных ревью (Fable 5, GPT 5.5, Opus 4.8), prompt судебной процедуры и оба результата судебного сравнения. Прежде чем собирать финальный вердикт, разрешу ключевые расхождения проверкой по коду.

Главные расхождения между двумя сравнениями, которые стоит разрешить кодом:
1. **domain-over-array**: comparison_opus48 ставит `confirmed`, comparison_gpt55 — `partially confirmed` («PG catalog behavior not checked»).
2. **attisdropped**: оба согласны, что это ложный claim, но цитируют разные строки (859 vs 864) — подтвержу фильтр.
3. **binary range** severity: gpt55 = `blocker`, opus48 = `high (latent)`.
4. уникальные claim'ы про element OID в `MultiDimArrayBinary.decode` и trailing bytes.

Проверю код.

Подтверждено по коду:
- `resolveByTyptype` ([CodecRegistry.java:484-506](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L484)) проверяет `isArray()` **первым**, до `isComposite()`/`isDomain()`/`isEnum()`/range.
- `isArray()` = `typcategory == 'A'` ([PgType.java:390](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgType.java#L390)); `isDomain()` = `typtype == 'd'` ([PgType.java:372](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgType.java#L372)).
- Фильтр `attisdropped` существует ([TypeInfoCache.java:864](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/TypeInfoCache.java#L864)) — claim Opus о dropped-attributes действительно ложный. gpt55 цитировал строку 859, фактически фильтр на 864 (запрос начинается на 859).

Критический момент расхождения domain-over-array (`confirmed` vs `partially confirmed`) зависит от того, наследует ли домен `typcategory` базового типа в PostgreSQL. Это решает, нужна ли правка precedence до масштабирования. Проверю эмпирически — посмотрю, поднят ли PostgreSQL.

Подтверждено: binary range берёт subtype из `type.getTypelem()` ([RangeCodec.java:99](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L99)), а у range-типа `typelem == 0` → element-codec не резолвится. Баг реален (latent, т.к. range не в default binary set).

PostgreSQL не поднят. Расхождение `confirmed` vs `partially confirmed` по domain-over-array целиком зависит от того, наследует ли домен `typcategory='A'` от базового массива — оба сравнения расходятся именно здесь, и hedge gpt55 методологически корректен. Это самое важное расхождение и оно определяет «решение перед масштабированием», поэтому разрешу эмпирически.

Полный pgjdbc-харнесс избыточен для проверки одного факта каталога. Подниму лёгкий одноразовый postgres и запрошу `pg_type`.

Эксперимент дал authoritative-ответ. Удалю контейнер.

Все ключевые расхождения разрешены. Собираю финальный adjudication.

---

# Финальный adjudication: сравнение двух судебных сравнений

## 1. Краткий вердикт

**Два судебных сравнения совпадают в ~90% выводов и не противоречат друг другу ни по одному подтверждённому багу.** Это само по себе сильный сигнал надёжности: независимые прогоны сошлись на одной карте рисков.

**Где согласны (полный консенсус, подтверждено кодом обоими):**
- public SPI протекает internal-типами (`api.codec.*` → `jdbc.PgType`/`CodecContext` → `BaseConnection`/`core.TypeInfo`);
- у decode нет zero-copy-слайса (копия на каждый элемент/поле/границу);
- новый `MultiDimArray*`-walker не стоит на горячем пути чтения; text-read массивов отсутствует → `ArrayEncoding`/`ArrayDecoding` пока неустранимы;
- registry ключуется голым именем без схемы; слои override/SPI/built-in/fallback не разделены;
- range subtype не загружается из `pg_range`; multirange не поддержан;
- оба согласны: лучший план в сумме — Fable, лучшая одиночная архитектурная развязка — Opus P4 (один `ArrayCodec` + опциональный fast-path leaf), GPT — корректный, но тонкий скелет;
- оба пометили «45/45 tests» как непроверенное и НЕ нашли галлюцинаций ни в одном первичном ревью (кроме одного ложного claim самого Opus-original — см. §4).

**Где расходятся (всего три существенных пункта, все разрешены):**

| # | Расхождение | A (gpt55) | B (opus48) | Разрешение |
|---|---|---|---|---|
| 1 | Статус domain-over-array | `partially confirmed` (PG-каталог не проверял) | `confirmed` | **B прав.** Эксперимент: домен наследует `typcategory` базового типа |
| 2 | Severity binary range | `blocker` | `high (latent)` | **B точнее.** range не в default binary set → latent |
| 3 | Severity decode-slice | `blocker` | `high` | Семантика. Финал: `high`, но **contract-gating** |

**Какое сравнение полезнее.** Для задачи adjudication — **comparison_opus48 чуть полезнее**: он (а) дал самую точную диагностику domain-over-array, которую эксперимент подтвердил дословно; (б) дисциплинированно отделил trade-offs от багов (явно вынес lower-bounds=1, квотирование чисел, domain-unwrap как осознанные компромиссы, а не дефекты); (в) явно разрешил противоречие Fable↔Opus по `attisdropped` в пользу Fable. **comparison_gpt55 сильнее в атомарности**: его матрица из 40 claims тоньше дробит registry-проблемы (отдельно `registerByName`-no-invalidate, `unregister`-no-restore, SPI-class-shadow) и он единственный поднял два code-confirmable пропуска (element OID, trailing-bytes validation).

**Надёжность финального вердикта: высокая** по confirmed issues (двойная сходимость + я перепроверил спорный по коду и эксперименту), **средняя** по severity-ранжированию (это вопрос политики «что считать блокером»), и несколько перф-claims остаются `unresolved` без JMH/построчного чтения (§5).

**Качество процедуры (оба).** Оба извлекли атомарные claims, проверили существенные по коду через `fff`/`ast-grep`, отделили факты от мнений, отличили галлюцинацию (`attisdropped`) от unresolved (`45/45`), не подменили сравнение новым architecture review и дали практичный next-step. comparison_opus48 аккуратнее метит `не проверено`; comparison_gpt55 атомарнее. Обе пригодны.

---

## 2. Матрица расхождений

`A` = comparison_gpt55, `B` = comparison_opus48. «Совпадают?» — про вывод, не про формулировку.

| Тема / claim | A | B | Совпадают? | Перепроверял? | Финальный вердикт |
|---|---|---|---|---|---|
| SPI протекает internal-типами | confirmed/blocker | confirmed/blocker | да | да (код) | **confirmed, blocker** |
| Offline заблокирован (`CodecContext` final, ctor package-private, `withTypeMap` бросает) | confirmed | confirmed | да | нет | **confirmed, high** |
| Decode без slice → копии | confirmed/**blocker** | confirmed/**high** | факт да, severity нет | да (код) | **confirmed, high (contract-gating)** |
| Registry: голое имя, schema-коллизия | confirmed+trade-off | confirmed | да | да (код) | **confirmed + trade-off на форму регистрации** |
| Registry: слои/override/unregister | confirmed (атомарно) | confirmed (сгруппир.) | да | да (код) | **confirmed, high** |
| **domain-over-array precedence** | **partially confirmed** | **confirmed** | **частично** | **да (эксперимент)** | **confirmed, high** |
| Binary range subtype из `typelem=0` | confirmed/**blocker** | confirmed/**high latent** | факт да, severity нет | да (код) | **confirmed, high (latent)** |
| Multirange не поддержан | confirmed | confirmed | да | да (код) | **confirmed, medium** |
| New walker не на горячем пути | confirmed | confirmed | да | да (код) | **confirmed, high** |
| Нет text-read leaf массивов | confirmed | confirmed | да | да (код) | **confirmed, high** |
| `attisdropped` НЕ фильтруется (claim Opus-original) | **false** | **false (не воспроизв.)** | да | да (код) | **false / hallucinated** |
| `unregister`/`reset` теряет built-in | confirmed (unique GPT) | confirmed (credits GPT) | да | нет | **confirmed, medium** |
| `registerByName` не инвалидирует `oidCache` | confirmed (unique Fable) | confirmed (unique Fable) | да | нет | **confirmed, low** |
| SPI `registerByClass(defaultJavaType)` затеняет String | confirmed | confirmed | да | нет | **confirmed, medium** |
| Lower bounds теряются (encode=1) | confirmed+trade-off | confirmed (trade-off) | да | да (код) | **confirmed, trade-off** |
| `computeDimensions` rank по статическому классу → dims=1 | confirmed (**регрессия**) | confirmed (факт да, «регрессия» не пров.) | частично | частично | **факт confirmed; «регрессия vs ArrayEncoding» unresolved** |
| `BackpatchingBinarySink` package-private | partially / overstated | confirmed с нюансом | да (оба добавляют нюанс) | да (код) | **confirmed boundary-leak; нюанс: scalar-streaming через `OutputStream` работает** |
| `getArray(Map)`/`getResultSet(Map)` binary → `notImplemented`; Opus-original звал «covered» | confirmed (opus overstated) | confirmed (opus overstated) | да | да (код) | **confirmed; Opus-original overstated** |
| `encodeText`/`setArray` для стороннего `Array` через `toString()` | confirmed | — (не поднял) | confirmed-by-one | нет | **confirmed, medium** |
| element OID отбрасывается в `MultiDimArrayBinary.decode` | raised (missed-by-all) | — | confirmed-by-one | частично | **confirmed-by-one, low/medium (важно для standalone/malformed)** |
| simple vs extended query mode пропущен моделями | raised (поверхностно) | raised (сильно) | да | нет | **confirmed: пропуск первичных моделей** |
| «45/45 tests» = зрелость | unclear | unclear | да | нет | **unresolved (не запускалось)** |
| Plan: Fable best / Opus P4 / GPT skeleton | да | да | да | — | **согласовано** |

---

## 3. Финальные confirmed issues

Подтверждены кодом (а domain-over-array — и экспериментом). Severity — консолидированная.

**Blocker (цементируется в публичном контракте, чинить под `@Experimental`):**
1. **SPI протекает internal-типами.** `api.codec.BinaryCodec/TextCodec` принимают `org.postgresql.jdbc.PgType` и `CodecContext`; `CodecContext` отдаёт `BaseConnection`, `core.TypeInfo`, `Encoding`. Источник: `BinaryCodec.java`, `CodecContext.java:275,285`. Любой сторонний codec тянет internal-ядро.

**High:**
2. **Decode без slice** → `Arrays.copyOfRange`/`new byte[]` на каждый элемент/поле/границу ([GenericArrayLeafCodec.java:116](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java#L116), [RangeCodec.java:120](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L120)). Contract-gating: трогает сигнатуру каждого decode-метода → решать до написания ~30 кодеков. Severity-расхождение (blocker vs high) разрешаю в `high, но contract-gating`: чинится backward-совместимо через default-делегацию на полный буфер, поэтому не «блокер реализации», но «блокер стабилизации сигнатуры».
3. **domain-over-array мис-роутится в `ArrayCodec`.** **Разрешено экспериментом** (PostgreSQL 17):

   | typname | typtype | typcategory | typelem | base_category |
   |---|---|---|---|---|
   | `d_arr` (domain AS int4[]) | `d` | **`A`** | **0** | A |
   | `d_comp` (domain AS comp) | `d` | `C` | 0 | C |
   | `d_int` (domain AS int4) | `d` | `N` | 0 | N |
   | `d_rng` (domain AS int4range) | `d` | `R` | 0 | R |

   Домен наследует `typcategory` базового типа. В `resolveByTyptype` ([CodecRegistry.java:486](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L486)) `isArray()` (`typcategory=='A'`) проверяется **первым** → `d_arr` уходит в `ArrayCodec`, где `getTypelem()==0` ломает резолв element-codec. **Только domain-over-array затронут**: domain-over-composite/scalar/range роутятся верно, потому что их проверки используют `typtype`, а typcategory-проверка одна — `isArray()` — и она первая. Фикс Opus точен: считать массивом только `typtype=='b' && typcategory=='A'`. **comparison_opus48 прав, hedge comparison_gpt55 снят.**
4. **Новый walker не на горячем пути.** `getArray()`/`getObject()` возвращают `PgArray` → legacy `ArrayDecoding`; `MultiDimArrayBinary.decode` исполняется лишь в `decodeBinaryAs(int[].class/Integer[].class)`. Цель «убрать `ArrayEncoding`/`ArrayDecoding`» сейчас недостижима by design.
5. **Нет text-read leaf** массивов в новой архитектуре → весь text-парсинг в legacy `ArrayDecoding`. Усилено пропущенной обоими темой: **simple query mode всегда text** → без text-read leaf simple-query целиком завязан на legacy.
6. **Binary range subtype из `typelem`** ([RangeCodec.java:99](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L99)): у range `typelem==0`, `pg_range` не загружается. Severity: **high (latent)** — range не в default binary transfer set; разрешаю blocker-vs-high в пользу `high/latent` (comparison_opus48). Первый, кто включит binary для range, получит ошибку.
7. **Registry identity и слои**: голое имя без схемы ([CodecRegistry.java:84,410](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L410)); SPI/built-in/custom в одних map; `registerByClass(getDefaultJavaType)` может затенить String; коллизии SPI — last-wins молча; ошибки загрузки глотаются.

**Medium:**
8. Multirange (`typtype='m'`) не в `resolveByTyptype` → молчаливый Fallback.
9. `unregister`/`reset` не восстанавливают перекрытый built-in (баг reset-сценария пула).
10. `encodeText`/`setArray` стороннего `Array` через `toString()` → мусор в SQL без ошибки.
11. `computeDimensions` считает rank по статическому классу → runtime-вложенный `Object[]` получает dims=1 (факт confirmed; «регрессия vs ArrayEncoding» — unresolved, §5).
12. `getArray(Map)`/`getResultSet(Map)` на binary → `notImplemented` ([PgArray.java:213,478](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgArray.java#L213)).
13. `getCodecContext()` аллоцирует новый контекст на каждый вызов, зовётся per-element в `ArrayDecoding`.

**Low:** `registerByName` не инвалидирует `oidCache`; single-arg `getBinaryCodec(int)` → Fallback на холодном кэше; enum→Java enum не декодируется; двойная rectangular-валидация на encode.

---

## 4. Финальные false / hallucinated claims

1. **`attisdropped` не фильтруется (claim Opus-original review, CR6).** **Ложный.** Фильтр существует: [TypeInfoCache.java:864](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/TypeInfoCache.java#L864) (`AND NOT a.attisdropped`, запрос с 859). Плюс PostgreSQL `record_send` исключает dropped-колонки из binary-формата, `row(...)::text` — тоже. Рассинхрона по позиции нет. **Оба сравнения сошлись на false** (comparison_opus48 явно разрешил противоречие Fable↔Opus в пользу Fable). Opus-original честно пометил его «обязательно проверить», но подал как вероятный баг — это переоценка, не выдумка.
2. **Opus-original: `getArray(Map)`/`getResultSet`/`getArray(Map)` «covered».** Overstated: срезы и `getResultSet()` есть, но binary `getArray(Map)` и `getResultSet(Map)` бросают `notImplemented`. Оба сравнения это поймали.
3. **Opus-original (через comparison): «third-party streaming codec *always* gets slow path».** Overstated, не ложь: `BackpatchByteArrayOutputStream` публичен, scalar element-codec стримит через `OutputStream`; container-level backpatch недоступен. Оба сравнения добавили один и тот же нюанс.

Чистых галлюцинаций (придуманное поведение/несуществующий код) в самих сравнениях **не обнаружено** — все file:line, что я перепроверял, попадают в реальный код.

---

## 5. Unresolved questions

| Вопрос | Почему не разрешён | Какая проверка нужна |
|---|---|---|
| «Регрессия vs `ArrayEncoding`» для runtime-nested `Object[]` (C25/C17) | dims=1 подтверждён, но поведение старого `ArrayEncoding` для `new Object[]{new Object[]{...}}` не сверялось | прогнать `createArrayOf("int4", Object[]{Object[]{...}})` на старом и новом пути, сравнить |
| Перф-claims Fable: encode→decode round-trips в `PgStruct`/`PgCallableStatement`, рост `TYPE_ALIASES`, megamorphic dispatch на `getInt`, `getString` мимо codec | оба сравнения пометили «не проверено» построчно | целевой JMH + построчное чтение `PgStruct.getAttributes`/`PgCS` OUT |
| «45/45 tests» как зрелость | тесты не запускались ни в одном сравнении | `./gradlew --quiet test` на codec-наборе |
| element OID validation / trailing-bytes (поднял только gpt55) | структурно `RangeCodec` проверяет truncation, но не trailing после upper bound; `MultiDimArrayBinary.decode` читает element OID из header и отбрасывает | code-level: добавить mismatch/trailing-assert и проверить на malformed payload |
| Реальная binary/text parity на сервере | ни одно сравнение не гоняло parity на PostgreSQL | parity-harness (план Fable) |

---

## 6. Design trade-offs (не баги — у PostgreSQL/JDBC нет единственного решения)

1. **Регистрация по имени без схемы.** Факт: коллизия одинаковых `typname` в разных схемах реальна (коммит `099675898` гасит её в тестах). Но это **trade-off, а не баг**: OID точен, но не переносим между БД для user-типов; unqualified-имя удобно и следует `search_path`; schema-qualified точно, но требует явной привязки `myschema.t → Codec`. **Рекомендация:** runtime-резолв по OID; регистрация — три формы (OID / schema-qualified / unqualified-alias) с явными conflict-rules. Оба сравнения правильно НЕ назвали это чистым багом.
2. **Lower bounds=1.** encode всегда пишет 1, decode игнорирует non-1 bounds. Совместимо со старым JDBC-поведением pgjdbc. **Зафиксировать контрактом в javadoc**, не «чинить».
3. **Domain unwrap.** `DomainCodec` делегирует в base-codec, теряя domain identity и typmod. Для `getObject` разворот ок; для имён в `Struct`/`SQLData` и для domain-over-array — нет. **Рекомендация:** сохранять `TypeDescriptor` домена (identity в metadata), переиспользовать base-codec для значения. Сознательное решение, а не потеря.
4. **Квотирование чисел в text-массиве.** generic квотит, `Int4ArrayLeafCodec` — нет. PostgreSQL принимает оба → функционально ок, но другие байты на проводе. **Проверить, что тесты не ассертят точные литералы.**
5. **`Array.getArray()` остаётся boxed (`Integer[]`).** По JDBC исторически — менять нельзя; primitive — только через `getObject(int[].class)`. Текущий дизайн разделяет верно.

---

## 7. Проблемы, пропущенные обоими сравнениями

Добавляю только подтверждённое кодом/спецификацией и нужное для решений §8.

1. **Связка «simple query mode ⇒ всегда text ⇒ зависит от legacy text-read».** comparison_gpt55 упомянул simple/extended вскользь, comparison_opus48 — сильнее, но **ни одно не связало это с приоритетом text-read leaf как gate на удаление `ArrayDecoding`**. Факт: в simple-протоколе binary-codecs не задействуются вовсе. Это повышает приоритет text-read leaf (Decision 4) с «следующая итерация» до «обязательно до удаления legacy».
2. **element OID из binary header отбрасывается** в `MultiDimArrayBinary.decode` (поднял только gpt55, подтверждаю структуру). Для trusted ResultSet терпимо, для standalone/raw decode и malformed payload — нужна валидация mismatch. Прямо относится к рубрике «malformed binary payload errors» из исходного prompt.
3. **typmod на уровне field vs type.** Codec API работает с `PgType`, а typmod (precision `numeric`, длина `varchar`) живёт на `PgField`/в домене (`typtypmod`). Для offline encode и domain-доменов это понадобится — ни одно сравнение не развело type-level и field-level metadata.

---

## 8. Решения перед масштабированием (с int4 на остальные типы)

1. **Граница public API.** Ввести `api.codec.TypeDescriptor` (read-only: oid, (schema,name), typtype, typcategory, typmod, elementOid, arrayOid, baseTypeOid, subtypeOid, fields, typdelim) и `CodecContext` как **интерфейс** с wire-частью, без `getConnection()`/`BaseConnection`/`core.TypeInfo`. *Блокирует:* после сторонних зависимостей контракт не переделать. *Сейчас:* делать под `@Experimental`. *Отложить:* состав descriptor для multirange.
2. **Сигнатура decode-слайса.** `(byte[] data, int off, int len)` vs `ByteBuffer` vs курсор. *Предпочтительно:* `(byte[], off, len)` + default-делегация на полный буфер (совместимо), ownership = borrowed на время вызова. *Блокирует:* трогает каждый decode-метод. *Отложить:* decode-streaming visitor (slice — его фундамент).
3. **Одна array-модель (Opus P4).** Универсальный `ArrayCodec`, композирующий element-codecs, + **опциональный** fast-path leaf-интерфейс на scalar-codec для `int[]/long[]/...`. *Блокирует:* иначе по отдельному codec+leaf на int2/int8/float4/float8/bool — то самое размножение. *Отложить:* fast-path для variable-width (text/numeric — сразу generic/streaming).
4. **Text-read leaf + общий токенизатор** (массив/composite/range). *Поднят до «обязательно до удаления legacy»* из-за связки с simple-query-mode (§7.1). *Сейчас:* зафиксировать интерфейс, чтобы `ArrayLeafCodec` его выдержал.
5. **Identity и конфликты registry.** Ключ OID (runtime) + (schema-optional name) для регистрации; слои `explicit OID > connection-name > SPI > built-in > fallback`; чинить `unregister`/`reset`; инвалидировать `oidCache` при name-регистрации; детерминированный выбор + логирование при конфликте SPI. *Отложить:* предикатную регистрацию (PostGIS, OID per-DB) — заложить место.
6. **Precedence классификации (подтверждён экспериментом).** Считать массивом только `typtype=='b' && typcategory=='A'`, либо проверять `typtype` раньше `typcategory`. Маленькая правка, без неё domain-over-array уезжает в `ArrayCodec`. *Сейчас:* делать + regression-тест на domain-over-array/scalar/composite/enum/range.
7. **Range/multirange metadata.** Загрузить `pg_range` (subtype) + `PgRangeType.subtypeOid`; передавать резолвнутый `PgType` в `getBinaryCodec(oid, pgType)`. *Сейчас:* `pg_range`; multirange — metadata-only с явной ошибкой в v1.
8. **Format capabilities.** Заменить `instanceof BinaryCodec`/«бросил исключение» на явные `supportsBinaryRead/Write`, `textRead/Write`; формат выбирать по пересечению server × codec × настройки. *Блокирует:* для сторонних binary-only/text-only кодеков.
9. **Контракт домена и typmod** (trade-off §6.3): разворачивать в base или сохранять identity для имён/domain-over-array; пробрасывать ли `typtypmod`. Зафиксировать как контракт.

---

## 9. Recommended next iteration

В порядке выполнения, минимизируя contract-риск:

1. **Periметр + precedence (дёшево, разблокирует остальное).** `api.codec.TypeDescriptor` + `CodecContext`-интерфейс (Decision 1); фикс `resolveByTyptype` на `typtype=='b' && typcategory=='A'` (Decision 6) + regression-тест domain-over-array/scalar/composite/enum/range. **Эксперимент уже показал, что тест поймает баг.**
2. **Зафиксировать decode-слайс** с default-делегацией на полный буфер (Decision 2), перевести `GenericArrayLeafCodec`/`CompositeCodec`/`RangeCodec` — убрать per-element `copyOfRange` до того, как кодеков станет 30.
3. **Прототип Opus P4 на `int8` или `bool`** (Decision 3): доказать, что fast-path как опциональный интерфейс на scalar-codec убирает дублирование. Чистый прототип = шаблон для остальных fixed-width.
4. **Parity-harness как gate** (план Fable): random value → server roundtrip → `text-decode == binary-decode == исходное`; матрица null/empty/ragged/lower-bounds/0-dim. Без него каждый перенесённый тип тихо меняет одно из поведений.
5. **Параллельно — точечные подтверждённые баги, не меняющие контракт:** `unregister`/`reset` built-in, single-arg `getBinaryCodec` в `RangeCodec`, `encodeText`/`setArray` стороннего `Array` через `getArray()`, кэш `CodecContext` на соединении, `registerByName`→инвалидация `oidCache`.

Text-read leaf (Decision 4) и `pg_range` (Decision 7) — следующая итерация, но интерфейс text-leaf набросать сейчас. Когда договоримся по 1–3 — можно спускаться на code-level findings (`RangeCodec` binary, text-парсер `CompositeCodec`, `MultiDimArraySupport.computeDimensions`, инварианты registry).

---

**Что я сделал сверх чтения:** перепроверил по коду `resolveByTyptype`/`PgType`/`TypeInfoCache`/`RangeCodec` и **поднял PostgreSQL 17 для эксперимента**, разрешившего главное расхождение (domain наследует `typcategory='A'` → domain-over-array confirmed). Я НЕ переделывал первичное architecture review и не пересравнивал первичные ответы с нуля — только адъюдицировал два сравнения.
