# Сравнение architecture review для Codec API в pgjdbc

## 1. Краткий вердикт

**Самый полезный ответ — Fable 5.** Он даёт самую полную карту рисков, отделяет правки, меняющие контракт, от тех, что можно сделать потом, и единственный предлагает конкретный план снижения риска (parity-harness как входной билет на удаление `ArrayEncoding`/`ArrayDecoding`, список JMH-бенчмарков, порядок миграции). При этом ни одного галлюцинированного claim у него я не нашёл.

**Самый доказательный по отдельной находке — Opus 4.8.** Его уникальный R3 (порядок проверок в `resolveByTyptype`: `isArray()` раньше `isDomain()`, из-за чего domain-over-array уезжает в `ArrayCodec`) — самый точный и проверяемый баг во всех трёх ответах, подтверждается и кодом, и семантикой PostgreSQL. Его же P4 (свести к одному `ArrayCodec` + опциональный fast-path-интерфейс) — самый чёткий ответ на вопрос «как не размножить код».

**GPT 5.5 — самый компактный и без ошибок**, но беднее по покрытию: меньше file-level находок, нет плана бенчмарков и порядка миграции. Зато у него единственная находка про `unregisterCustomCodec`/`resetCustomCodecs`, которые удаляют имя, не восстанавливая перекрытый built-in, и про контракт «codec кэшируется в `Field`».

**Где сходятся все три:** граница public SPI протекает internal-типами; type identity в registry — голое имя без схемы; у decode нет zero-copy-слайса (копия на каждый элемент/поле); range/multirange metadata не готова; новый array-walker не стоит на горячем пути чтения; lower bounds теряются; offline-режим заблокирован тем, что `CodecContext` — `final` connection-bound класс.

**Где расходятся:** Opus уникально нашёл precedence-баг domain-over-array и предложил самую сильную модель «один array codec + fast-path SPI». Fable уникально разобрал перф-детали (per-call `getCodecContext`, двойная rectangular-валидация, encode→decode round-trips) и дал план тестов. GPT уникально нашёл потерю built-in при unregister/reset и контракт кэширования в `Field`. По `attisdropped` Fable и Opus прямо противоречат друг другу — проверка ниже решает в пользу Fable.

---

## 2. Методика

Сравнивались три файла: [fable5.md](../2-review-execution/fable5.md), [gpt55.md](../2-review-execution/gpt55.md), [opus48.md](../2-review-execution/opus48.md) против рубрики из [design-review-prompt.md](../1-review-prompt-creation/design-review-prompt.md).

Текущее состояние кода совпадает с тем, что смотрели модели: `GenericArrayLeafCodec`, `Int4ArrayLeafCodec`, `MultiDimArraySupport`, `ArrayLeafStreamingCodec`, `BackpatchingBinarySink` присутствуют как untracked-файлы поверх коммита `61d61d000`.

**Прочитал и проверил по коду:**
- публичный SPI: [Codec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/Codec.java), [BinaryCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/BinaryCodec.java), [CodecContext.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java);
- registry: [CodecRegistry.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java), классификация [PgType.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgType.java);
- array-путь: [ArrayCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayCodec.java), [MultiDimArrayBinary.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java), [GenericArrayLeafCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java), [Int4ArrayLeafCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/Int4ArrayLeafCodec.java), [ArrayLeafStreamingCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayLeafStreamingCodec.java), [MultiDimArraySupport.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArraySupport.java);
- container/scalar codecs: [RangeCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java), [DomainCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/DomainCodec.java), [EnumCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/EnumCodec.java), [BackpatchingBinarySink.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/BackpatchingBinarySink.java);
- JDBC-адаптеры и инфраструктура: [IdentifierNormalizingTypeMap.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/IdentifierNormalizingTypeMap.java), `PgArray`, `PgConnection.getCodecContext`, `ArrayDecoding`, фильтр `attisdropped` в `TypeInfoCache`.

**Остались без проверки кода (помечены ниже как «не проверено»):** часть перф-claims Fable про encode→decode round-trips в `PgStruct`/`PgCallableStatement`, рост `TYPE_ALIASES`, megamorphic dispatch на `getInt`, `getString()` мимо codec; claim GPT про `registerOutParameter(String,...)`; claim Opus CG1 про `setObject` DTO без type hint. Они правдоподобны и согласуются с архитектурой, но я их не открывал построчно.

**Замечание про дрейф кода:** PostgreSQL семантику record_send (dropped-колонки исключаются из binary-формата) я не запускал на сервере — это знание спецификации, на нём строится разбор Opus CR6.

---

## 3. Матрица claims

Severity: B=blocker, H=high, M=medium, L=low. Статус проверки в последней колонке.

| # | Claim (кратко) | Категория | F5 | O48 | G55 | Sev | Проверка по коду | Статус |
|---|---|---|---|---|---|---|---|---|
| C1 | SPI `api.codec.*` тянет `jdbc.PgType`/`CodecContext` | SPI boundary | ✓ | ✓ | ✓ | B | BinaryCodec.java:9-10,60 | confirmed |
| C2 | `CodecContext.getConnection()`→`BaseConnection`, `getTypeInfo()`→`core.TypeInfo` | SPI boundary | ✓ | ✓ | ✓ | B | CodecContext.java:275,285 | confirmed |
| C3 | Offline-режим заблокирован: `CodecContext` final, connectionless ctor package-private, `withTypeMap` бросает | SPI boundary | ✓ | ✓ | ✓ | H | CodecContext.java:41,155,218 | confirmed |
| C4 | `decodeAsBoolean` зовёт `jdbc.BooleanTypeUtil` из public-интерфейса | SPI boundary | ✓ | – | – | M | BinaryCodec.java:172 | confirmed |
| C5 | Registry ключует codecs голым именем; OID-резолв через `getTypeName().getName()` → коллизия одинаковых имён в разных схемах | registry | ✓ | ✓ | ✓ | H | CodecRegistry.java:84,410; коммит 099675898 | confirmed |
| C6 | `resolveByTyptype`: `isArray()` (typcategory='A') проверяется раньше `isDomain()` → domain-over-array уходит в ArrayCodec, typelem=0 ломает резолв | scalar/registry | – | ✓ | – | H | CodecRegistry.java:486→494; PgType.java:372,390 | confirmed |
| C7 | Multirange (`typtype='m'`) не поддержан, имена `*multirange` не зарегистрированы → Fallback | ranges | ✓ | ✓ | ✓ | M | CodecRegistry.java:484-506 | confirmed |
| C8 | Decode без слайса: `copyOfRange`/`new byte[]` на каждый элемент/границу/поле | performance | ✓ | ✓ | ✓ | H | GenericArrayLeafCodec.java:116; RangeCodec.java:120,139 | confirmed |
| C9 | Новый `MultiDimArray*`-walker не на горячем пути: `getArray()`/`getObject()` идут через `PgArray`→`ArrayDecoding` | array | ✓ | ✓ | ✓ | H | ArrayCodec.java:81,250; ArrayLeafStreamingCodec.java:94,113 | confirmed |
| C10 | Text-read массивов в новой архитектуре отсутствует; делегируется legacy `ArrayDecoding` | array | ✓ | ✓ | – | H | ArrayCodec.java:272-273; ArrayLeafStreamingCodec.java:129-130 | confirmed |
| C11 | Lower bounds: encode всегда пишет 1, decode игнорирует | array correctness | ✓ | ✓ | ✓ | M | MultiDimArrayBinary.java:132,195 | confirmed (trade-off) |
| C12 | Две конкурирующие array-абстракции: `ArrayCodec` vs `ArrayLeafStreamingCodec` | array/maintainability | ~ | ✓ | – | M | оба в registry; CodecRegistry.java:186,192 | confirmed |
| C13 | Спец-leaf переинлайнивает scalar-логику (`Int4ArrayLeafCodec` не зовёт `Int4Codec`); не масштабируется на numeric/timestamptz | perf/maintainability | ✓ | ✓ | – | M | Int4ArrayLeafCodec.java:57-58,94 | confirmed |
| C14 | Разное квотирование text-элементов: generic квотит, int4 — нет | array correctness | – | ✓ | – | L | GenericArrayLeafCodec.java:152-162 vs Int4ArrayLeafCodec.java:125 | confirmed (trade-off) |
| C15 | Нет primitive-encode на SPI: `Integer[]`/`Object[]` боксят | performance | – | ✓ | – | M | BinaryCodec: только decodeAs* | confirmed |
| C16 | `BackpatchingBinarySink` package-private → сторонний container-codec не может использовать backpatch | SPI boundary/perf | – | ✓ | – | M | BackpatchingBinarySink.java:18 | confirmed (с нюансом) |
| C17 | Binary range сломан: subtype из `typelem`=0 без guard; single-arg `getBinaryCodec` отбрасывает резолвнутый тип | ranges correctness | ✓ | – | ✓ | H | RangeCodec.java:99-101 vs 279-281 | confirmed (latent) |
| C18 | `getBinaryCodec(int)`/`getTextCodec(int)` → `getByOid(oid,null)` → Fallback на холодном кэше | registry | ✓ | – | – | M | CodecRegistry.java:540-543 | confirmed |
| C19 | SPI static per-classloader; перекрывает built-in простым `put`; `registerByClass(getDefaultJavaType)` может затенить String; ошибки загрузки глотаются | registry | ✓ | ✓ | ✓ | M | CodecRegistry.java:99-100,138-145,268-273 | confirmed |
| C20 | `unregisterCustomCodec`/`resetCustomCodecs` удаляют имя, не восстанавливая built-in/SPI | registry | – | – | ✓ | M | CodecRegistry.java:337-342,350-357 | confirmed |
| C21 | `registerByName`/`registerAlias` не инвалидируют `oidCache` | registry | ✓ | – | – | L | CodecRegistry.java:280-292 vs 329 | confirmed |
| C22 | `DomainCodec` теряет domain identity и typmod (делегирует в base type) | domains | ✓ | ~ | – | M | DomainCodec.java:54-60,74-86 | confirmed (trade-off) |
| C23 | `EnumCodec` не декодирует в Java enum (только String/Object) | enums | ✓ | – | – | L | EnumCodec.java:89-92,101-104 | confirmed |
| C24 | `ArrayCodec.encodeText` для стороннего `Array` делает `toString()` → мусор в SQL | array correctness | ✓ | – | ✓ | M | ArrayCodec.java:197-200 vs 103 | confirmed |
| C25 | `computeDimensions` считает rank по статическому классу → runtime-вложенный `Object[]` получает dims=1 | array correctness | ✓ | – | – | M | MultiDimArraySupport.java:28-36 | confirmed (регрессия — не проверена) |
| C26 | `CompositeCodec` text молча терпит рассинхрон числа полей (`Math.min`) | composite | ✓ | ~ | – | M | CompositeCodec.java:571-572 | confirmed |
| C27 | Dropped attributes (`attisdropped`) могут рассинхронить composite-decode | composite | ✗(есть фильтр) | ✓(проверить) | – | M | TypeInfoCache.java:864 фильтрует | **не воспроизводится** |
| C28 | `getCodecContext()` аллоцирует новый контекст на каждый вызов; зовётся per-element в `ArrayDecoding` | performance | ✓ | – | – | M | PgConnection.java:2102; ArrayDecoding.java:453,469 | confirmed |
| C29 | Двойная rectangular-валидация на encode (estimate + encode) | performance | ✓ | – | – | L | MultiDimArrayBinary.java:124,162 | confirmed |
| C30 | `PgArray.getArray(Map)`/`getResultSet(Map)` на binary → `notImplemented` | JDBC gap | ✓ | ~ | ✓ | M | PgArray.java:213,478 | confirmed |
| C31 | `IdentifierNormalizingTypeMap` решает qualified/quoted имена через regtype→OID (фикс Lukas Eder) | JDBC (положительное) | ✓ | ✓ | ✓ | — | IdentifierNormalizingTypeMap.java:96-112 | confirmed |
| C32 | Codec кэшируется в `Field`; `registerCodec` влияет на будущие запросы, не на инициализированные fields | registry contract | ~ | – | ✓ | L | заявлено двумя, код не открывал | partially confirmed |
| C33 | binary/text parity не закреплена тестами до конвертации типов | tests | ✓ | ~ | – | H | следует из C9/C10 | confirmed (риск) |

Легенда: ✓ — поднял, ~ — задел косвенно, – — не поднял, ✗ — заявил обратное.

---

## 4. Подтверждённые общие выводы

**Найдено всеми тремя и подтверждено кодом:**

- **C1/C2 — SPI протекает internal-типами.** Это главный консенсус. `api.codec.BinaryCodec` принимает `org.postgresql.jdbc.PgType` и `CodecContext`; `CodecContext` отдаёт `BaseConnection` и `core.TypeInfo`. Любой сторонний codec, которому нужны metadata или дочерний codec, импортирует internal-ядро. Чинить дёшево сейчас, пока всё под `@Experimental`.
- **C5 — type identity недостаточна.** Runtime-резолв идёт по OID (через `oidCache`/`explicitOidCodecs`), но built-in и custom codecs находятся через `pgType.getTypeName().getName()` — голое имя без схемы. Тип `point` в чужой схеме получит built-in `PointCodec`. Это не теория: коммит `099675898` гасит ровно этот эффект в тестах.
- **C8 — decode без слайса.** Encode уже стримит через backpatch, decode симметрии не имеет: копия на каждый элемент массива, каждую границу range, каждое поле composite. Для `array-of-struct-of-array` это O(payload × depth). Меняет сигнатуру каждого decode-метода, поэтому решать надо до написания остальных ~30 кодеков.
- **C9 — новый walker не на горячем пути.** `getArray()`/`getObject()` возвращают `PgArray`, который декодирует через legacy `ArrayDecoding`. `MultiDimArrayBinary.decode` исполняется только в `decodeBinaryAs(int[].class/Integer[].class)`. Цель «убрать `ArrayEncoding`/`ArrayDecoding`» сейчас недостижима by design — новый путь сам на них опирается (C10).
- **C7 — multirange отсутствует**, **C17 — binary range не резолвит subtype** (typelem у range = 0, а `pg_range` не загружается).

**Найдено несколькими и подтверждено:** C3 (offline заблокирован — Fable, Opus), C10 (нет text-read leaf — Fable, Opus), C11 (lower bounds — все), C12/C13 (две array-абстракции и переинлайненная scalar-логика — Fable, Opus), C24 (`toString()` для стороннего `Array` — Fable, GPT), C19 (SPI overlay — все), C30 (`getArray(Map)` binary not implemented — все).

---

## 5. Уникальные полезные findings

**Только Opus 4.8:**
- **C6 — precedence-баг domain-over-array (R3).** Самая ценная одиночная находка. `resolveByTyptype` проверяет `isArray()` (а это `typcategory=='A'`) раньше `isDomain()` (`typtype=='d'`). Домен над массивом наследует `typcategory='A'`, но `typelem=0`, поэтому уезжает в `ArrayCodec`, где `streamBinaryArrayViaCodec` берёт `getTypelem()=0` и не находит element-codec. Подтверждено и кодом, и семантикой PostgreSQL (домен наследует typcategory базового типа). Правильный фикс: проверять `typtype` (composite/domain/enum/range/multirange) раньше `typcategory`, либо считать массивом только `typtype=='b' && typcategory=='A'`.
- **C15 — нет primitive-encode на SPI**, **C16 — `BackpatchingBinarySink` package-private** (сторонний streaming-codec видит только `OutputStream`, container-level backpatch ему недоступен; для скалярного element-codec стриминг через `OutputStream` всё же работает — нюанс).

**Только GPT 5.5:**
- **C20 — `unregisterCustomCodec`/`resetCustomCodecs` теряют built-in.** `codecsByName.remove(typeName)` удаляет запись целиком; если custom codec перекрывал built-in по имени, после отмены built-in не возвращается. Реальный баг reset-сценария пула соединений.
- **C32 — контракт кэширования codec в `Field`** (заявлено и Fable, не проверял построчно): регистрация codec влияет на будущие запросы, не на уже инициализированные колонки. Это контракт, который надо зафиксировать в javadoc.

**Только Fable 5:**
- **C28 — per-call/per-element `getCodecContext()`**, **C29 — двойная rectangular-валидация**, **C25 — `computeDimensions` по статическому классу**, **C18 — single-arg `getBinaryCodec` возвращает Fallback на холодном кэше**, **C21 — `registerByName` не инвалидирует `oidCache`**, **C22 — потеря domain identity/typmod**, **C23 — enum→Java enum не декодируется**, **C24 — `toString()` для стороннего `Array`** (разделил с GPT).

---

## 6. Сомнительные или ложные claims

- **C27 (Opus CR6, dropped attributes) — на проверке не воспроизводится.** Opus честно пометил его как «обязательно проверить», но подал как вероятный «тихий» баг. Фильтр существует: `TypeInfoCache.java:864` грузит поля composite с `AND NOT a.attisdropped`. Плюс PostgreSQL `record_send` исключает dropped-колонки из binary-формата (шлёт `validcols`, не `natts`), а `row(...)::text` их тоже опускает. Список полей и wire-формат фильтруют dropped согласованно, рассинхрона по позиции нет. **Здесь Fable прав** («ленивые composite fields с фильтром `attisdropped` на сервере»), а формулировка Opus как баг-кандидата — переоценка. Стоит закрыть regression-тестом, но это не активный баг.
- **C25 (Fable, «регрессия для `createArrayOf`»)** — факт `dims=1` для runtime-вложенного `Object[]` подтверждён, но утверждение, что это именно регрессия относительно старого `ArrayEncoding`, я по коду `ArrayEncoding` не сверял. Сам факт — confirmed; слово «регрессия» — не проверено.
- **C11/C14/C22 — это design trade-offs, а не баги.** Lower bounds=1 и квотирование чисел все три подали корректно («задокументировать»), но важно не считать их дефектами: lower-bounds-нормализация совместима со старым JDBC-поведением pgjdbc, квотированные числа PostgreSQL принимает. Domain «прозрачно разворачивается» — это сознательный контракт, а не потеря (вопрос только в том, фиксировать ли его).
- **Перф-claims Fable про encode→decode round-trips** (`PgStruct.getAttributes`, `PgCallableStatement` OUT, `PgArray.toString`) — правдоподобны и согласуются с тем, что binary-backed `PgArray.getArray(Map)` не реализован (C30), но конкретные строки я не открывал. Помечаю как **не проверено**, не как подтверждённые.
- **Галлюцинаций не обнаружено** ни в одном из трёх ответов. Все file:line-ссылки, которые я проверял, попадают в реальный код (с поправкой на ±1-2 строки из-за дрейфа untracked-файлов).

---

## 7. Пропущенные темы

Пункты рубрики, которые не покрыл никто или покрыл поверхностно (мои находки, подтверждённые кодом/спецификацией):

- **Simple query mode vs extended query mode — пропущено всеми.** Рубрика спрашивала про это явно. В simple-query-протоколе сервер всегда возвращает text, binary-codecs не задействуются вовсе. Это прямо усиливает C10: пока text-read массивов живёт в legacy `ArrayDecoding`, simple-query-режим целиком зависит от старого пути, и «убрать `ArrayDecoding`» нельзя без полноценного text-read leaf. Ни одна модель не связала эти два факта.
- **Резолв имён в registry не учитывает `search_path` — затронуто лишь косвенно.** `IdentifierNormalizingTypeMap` использует серверный `regtype` (search_path-aware) — это хорошо и все три отметили. Но сам registry (`codecsByName`, таблица алиасов) ключуется по unqualified-именам без учёта схемы и `search_path`; для built-in это нормально, для будущих custom-кодеков — нет. Явно никто не развёл эти два уровня.
- **Capability-based выбор формата — только Fable (#4).** Сейчас «codec умеет binary» определяется `instanceof BinaryCodec`, а «не смог» — исключением изнутри. Для сторонних кодеков формат надо выбирать по пересечению (server-supported × codec-supported × настройки `binaryTransfer*`). Opus и GPT это не подняли.
- **Negative caching и рост `TYPE_ALIASES` — только Fable.** Промахи lookup (неизвестный OID/имя) не кэшируются; `TYPE_ALIASES` растёт process-wide. Не проверял построчно, но тема реальна и осталась вне GPT/Opus.
- **Enum-метки со спецсимволами и их квотирование в массивах/composite — пропущено всеми.** Рубрика спрашивала; `EnumCodec` просто прокидывает строки, поведение в составе array/composite literal никто не разобрал.

---

## 8. Сравнение планов

**Критерий — практическая пригодность для быстрой и масштабируемой реализации, а не красота формулировок.**

**Fable 5 — лучший по сумме.** Единственный, кто:
- разделил работы на «меняют контракт, делать до масштабирования» (периметр API, decode-слайс, text-read leaf, registry identity, `pg_range`) и «не меняют контракт, можно потом»;
- предложил **parity-harness** как входной билет на удаление legacy: property-тест «random value → text-encode → server roundtrip → binary-decode == text-decode == исходное» + матрица lower-bounds/null/empty/ragged/0-dim;
- дал конкретный JMH-набор (`getInt`/`getString` text+binary, `int4[]` 1K/100K как `int[]`/`Integer[]`/`getArray()`, composite, `-prof gc`);
- описал порядок миграции: не делать `ArrayEncoding` thin-wrapper'ом, а переключать registry с `ArrayCodec.INSTANCE` на leaf-codec по мере готовности (как уже сделано для `_int4`), а `ArrayDecoding` умрёт вместе с text-reader'ом.

Слабое место — местами слишком много пунктов сразу, нет одного «если сделать только одно».

**Opus 4.8 — лучшая архитектурная развязка по array-модели.** Его **P4** — самый прямой ответ на «как не размножить код для int2/int4/int8/float4/float8/bool»: один универсальный `ArrayCodec`, композирующий element-codecs, плюс **опциональный** fast-path-интерфейс (нынешний `ArrayLeafCodec`), который scalar-codec реализует для `int[]/long[]/...`. Это снимает C12 (две абстракции) и C13 (переинлайненная логика) одним решением. Плюс точечный P6 (фикс precedence) и чёткое меню следующего шага (A: проверить CR6/CR4 на сервере; B: прототип P4 на int8/bool; C: наброски сигнатур). Слабее по тестам/бенчмаркам и порядку миграции.

**GPT 5.5 — корректный скелет, но тоньше.** Верно называет крупные решения (public `TypeDescriptor`, слои registry, raw standalone API `RawValue`/`CodecSession`, загрузка `pg_range`/`pg_multirange`), уникально ловит C20 и C32. Но не запускал ничего, не дал ни бенчмарков, ни порядка работ, ни parity-стратегии.

### Объединённый план (что взять, что отбросить)

**Взять:**
- из **Opus**: P4 (один `ArrayCodec` + опциональный fast-path leaf) как канонический ответ на дублирование; P6 (фикс precedence в `resolveByTyptype`); идею «прототип fast-path на int8/bool до масштабирования»;
- из **Fable**: разделение «контрактные / неконтрактные» работы; parity-harness как gate на удаление legacy; JMH-набор с `-prof gc`; порядок миграции (переключать registry по мере готовности leaf, без thin-wrapper);
- из **GPT**: слоистую модель registry с явными правилами конфликтов (explicit OID > connection name > SPI > built-in > fallback); фикс `unregister`/`reset` (восстанавливать перекрытый built-in); фиксацию контракта кэширования в `Field`; форму standalone API `RawValue`/`CodecSession`.

**Отбросить / отложить:** промежуточный шаг «сделать `ArrayEncoding` thin-wrapper» (Fable прав — дешевле переключать registry); полноценный decode-streaming visitor (отложить, но заложить slice-сигнатуры под него); offline API как фичу (отложить, но не блокировать — сделать `CodecContext` интерфейсом сейчас).

---

## 9. Решения перед масштабированием

Семь решений, которые надо принять до расширения с `int4` на остальные типы (не смешиваю с обычной реализацией).

1. **Граница public API.** Ввести `api.codec.TypeDescriptor` (read-only metadata: oid, (schema,name), typtype, typmod, elementOid, arrayOid, baseTypeOid, subtypeOid, fields, typdelim) и `CodecContext` как **интерфейс** с wire-частью (charset, server params, lookup), без `getConnection()`/`BaseConnection`/`core.TypeInfo`. *Почему блокирует:* после появления сторонних зависимостей контракт не переделать. *Сейчас:* делать, пока `@Experimental`. *Отложить:* конкретный состав `TypeDescriptor` для multirange.
2. **Сигнатура decode-слайса.** Выбрать один контракт: `(byte[] data, int off, int len)` против `ByteBuffer` против курсора/reader. *Почему:* трогает каждый decode-метод каждого codec, решается один раз. *Предпочтительно сейчас:* `(byte[], off, len)` + default-делегация на полный буфер (совместимо), ownership = borrowed view на время вызова. *Отложить:* decode-streaming visitor (но slice — его фундамент).
3. **Одна array-модель.** Опус P4: универсальный `ArrayCodec` + опциональный fast-path leaf-интерфейс на scalar-codec. *Почему:* иначе по отдельному codec и leaf-классу на каждый из int2/int8/float4/float8/bool — ровно то размножение, которого боимся (C12/C13). *Предпочтительно сейчас:* P4. *Отложить:* fast-path для variable-width типов (text/numeric — сразу в generic/streaming).
4. **Text-read leaf + общий токенизатор.** Спроектировать курсорный парсер literal-формата, общий для массивов, composite и range. *Почему:* без него «убрать `ArrayDecoding`» не случится (C10), и три отдельных парсера quoting/escaping — три набора багов. *Сейчас:* хотя бы зафиксировать интерфейс, чтобы `ArrayLeafCodec` его поддержал. *Отложить:* перевод самого `PgArray.getArray()` на новый путь.
5. **Identity и конфликты registry.** Ключ — OID (runtime) + (schema-optional name) для регистрации; слои explicit OID > connection name > SPI > built-in > fallback; чинить `unregister`/`reset` (C20); инвалидировать `oidCache` при name-регистрации (C21); детерминированный выбор при конфликте SPI + логирование. *Почему блокирует:* затрагивает поведение каждого custom-codec. *Отложить:* предикатную регистрацию (для PostGIS, где OID per-database) — но заложить место под неё.
6. **Precedence классификации типов** (C6). Проверять `typtype` раньше `typcategory` в `resolveByTyptype`. *Почему:* маленькая правка, но без неё domain-over-array и потенциально composite/enum с унаследованным typcategory едут не туда. *Сейчас:* делать, это не контрактное, но блокирует целевые сценарии.
7. **Range/multirange metadata.** Загрузить `pg_range` (subtype) и решить про `pg_multirange`. *Почему:* без subtype binary range сломан (C17), multirange — в целях, но не в коде (C7). *Предпочтительно сейчас:* `pg_range` + `PgRangeType.subtypeOid`; multirange — metadata-only с явной ошибкой в v1. *Отложить:* полный multirange codec.

Дополнительно, но менее блокирующе: **контракт домена** (разворачивать в base — текущее поведение — или сохранять identity для имён в `Struct`/`SQLData`; пробрасывать ли typmod, C22) и **политика boxing** (`Array.getArray()` остаётся boxed по JDBC, primitive — только через `getObject(int[].class)`).

---

## 10. Рекомендуемый следующий шаг

На ближайшую итерацию, в порядке выполнения:

1. **Закрепить периметр и precedence (дёшево, разблокирует остальное).** Ввести `api.codec.TypeDescriptor` + `CodecContext`-интерфейс (решение 1); починить порядок в `resolveByTyptype` (решение 6) и сразу прикрыть тестом domain-over-array, domain-over-scalar, composite, enum.
2. **Зафиксировать decode-слайс-сигнатуру** с default-делегацией на полный буфер (решение 2) и перевести на неё контейнеры `GenericArrayLeafCodec`/`CompositeCodec`/`RangeCodec` — это убирает per-element `copyOfRange` (C8) до того, как кодеков станет 30.
3. **Прототип P4 на `int8` или `bool`** (решение 3): показать, что fast-path как опциональный интерфейс на scalar-codec убирает дублирование. Если прототип чистый — это и есть шаблон для остальных fixed-width типов.
4. **Поставить parity-harness** (план Fable) как gate: random value → server roundtrip → `text-decode == binary-decode == исходное`, матрица null/empty/ragged/lower-bounds/0-dim. Без него каждый перенесённый тип будет тихо менять одно из поведений (C33).
5. Параллельно — починить точечные подтверждённые баги, не меняющие контракт: `unregister`/`reset` built-in (C20), single-arg `getBinaryCodec` в `RangeCodec` (C17/C18), `encodeText` для стороннего `Array` через `getArray()` (C24), кэш `CodecContext` на соединении (C28).

Решения 4 (text-read leaf) и 7 (`pg_range`) — следующая итерация, но интерфейс text-leaf лучше набросать уже сейчас, чтобы `ArrayLeafCodec` его поддержал.

Если по решениям 1–3 договоримся — дальше имеет смысл спускаться на code-level findings по конкретным файлам (`RangeCodec` binary, text-парсер `CompositeCodec`, `MultiDimArraySupport.computeDimensions`, инварианты registry).
