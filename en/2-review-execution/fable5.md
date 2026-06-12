## Verdict

**The design is viable, but needs changes before scaling up.** The core of the new array plumbing — the split between a shape walker (`MultiDimArrayBinary`/`MultiDimArrayText`) and typed leaf loops (`ArrayLeafCodec`) — is correct and replicates mechanically to `int8`/`float8`/`bool`/`uuid`. Streaming encode with length back-patching, the codec cache on `Field`, and the primitive specialisations `decodeAsInt`/`decodeTextBytesAsInt` are all worth keeping as they are.

But three things need resolving **before** replicating to the remaining types, because they change signatures and contracts, and the cost of fixing them grows with every new codec:

1. the public SPI / internal boundary does not exist yet — `api.codec.*` drags in `BaseConnection`, `TypeInfo`, and JDBC classes;
2. the decode side of the API has no slice abstraction (a `byte[]` copy per element/field) and has no text-decode path for arrays at all — `ArrayLeafCodec.readLeaf` exists only for binary;
3. the registry identity model — bare name strings without a schema, and SPI codecs that globally override built-ins.

What follows is organised by the sections you asked for.

---

## 1. What is already good (keep through any rework)

- **Leaf/walker decomposition** ([MultiDimArrayBinary.java:30-36](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java#L30)): the walker owns header/dimensions/rectangular validation and traverses the outer dimensions via reflection (cost bounded by the product of the outer dimensions), while the leaf owns the hot typed loop. `Int4ArrayLeafCodec` is ~150 lines per type; `int2/int8/float4/float8/bool` are added without duplicating the format.
- **`BackpatchingBinarySink` + `reserveInt32`/`setInt32At`** — the right solution for length-prefixed nested encode without intermediate `byte[]`. `GenericArrayLeafCodec.writeLeaf` ([GenericArrayLeafCodec.java:84-88](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java#L84)) and `CompositeCodec` (lines 765-769) already use it for streaming elements.
- **Codec caching on `Field`** (`Field.initializeCodec`, once per column) and pre-resolving per-field codecs in `PgSQLInput*`/`PgSQLOutput*` at construction time.
- **`getInt()` does not box**: binary → `decodeAsInt(byte[])`, text → `decodeTextBytesAsInt` with an ASCII fast path ([Int4Codec.java:107-117](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/Int4Codec.java#L107)).
- **Type cache invalidation epoch** on DDL and `SET search_path`, lazy composite fields with an `attisdropped` filter on the server, and name resolution via `?::regtype` — qualifying/quoting/case-folding is done by the server, not a client-side parser. This closes most of Lukas Eder's objection about `Map<String, Class<?>>`: `IdentifierNormalizingTypeMap` reduces keys to OIDs via regtype, so `"myschema.mytype"` and `"\"MyType\""` match.
- **`CodecDepth`** (limit 64) against recursive types, and **`FallbackCodec`** — the registry never returns null.

---

## 2. Main architectural risks

### R1. The public SPI is welded to internal (blocks both stabilisation and offline mode)

- `org.postgresql.api.codec.Codec/BinaryCodec/TextCodec` reference `org.postgresql.jdbc.CodecContext` and `org.postgresql.jdbc.PgType` — types from the jdbc package, which has historically been internal.
- `CodecContext.getConnection()` returns **`BaseConnection`** ([CodecContext.java:275-278](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java#L275)), `getTypeInfo()` returns **`core.TypeInfo`**, which now itself imports jdbc classes (a cyclic core → jdbc dependency). Any third-party codec that needs a lazy `PgArray` is obliged to call `ctx.getConnection()` — that is, it depends on the entire internal core.
- `ArrayLeafStreamingCodec.decodeBinary` returns `new PgArray(ctx.getConnection(), …)` ([ArrayLeafStreamingCodec.java:92-95](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayLeafStreamingCodec.java#L92)) — a JDBC object as the result of a wire decode.
- The default methods `BinaryCodec.decodeAsBoolean` call `org.postgresql.jdbc.BooleanTypeUtil` ([BinaryCodec.java:170-174](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/BinaryCodec.java#L170)).
- `PgType` is a concrete class with five public constructors and `withFields`; it cannot be pinned down as a stable contract.

Consequence for offline encode/decode: the "no live connection" mode is currently **architecturally blocked**, not merely deferred. The connectionless `CodecContext` constructor is package-private and assembles a half-alive object (`getTypeInfo()`/`getCodecs()` throw), and `withTypeMap` on it fails. You asked for "offline can be deferred, but not blocked" — the current contract does not meet that requirement.

### R2. Decode without slicing: a copy per element and per field

`decodeBinary(byte[] data, …)` accepts only the full buffer. Therefore:

- `GenericArrayLeafCodec.readLeaf` does `Arrays.copyOfRange` **per element** ([GenericArrayLeafCodec.java:116](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java#L116));
- `CompositeCodec.decodeBinaryFields` does `new byte[length]` **per field** (CompositeCodec.java:156-157);
- `RangeCodec` copies per bound.

For `array-of-struct-of-array` the same bytes are copied at every nesting level: O(payload × depth) allocations and memcpy. The encode side solved this with the streaming sink; the decode side has no symmetric solution. This is the **most expensive change to defer**, because it touches the signature of every decode method of every codec — it has to change before the other ~30 codecs are written, not after.

### R3. Array text decode is missing as a class in the new architecture

`ArrayLeafCodec` has `writeLeaf` (binary), `readLeaf` (binary), and `appendLeaf` (text encode) — but **no text read**. `MultiDimArrayText` can only encode. All parsing of array text literals still lives in `ArrayDecoding.buildArrayList`/`readStringArray` (via `PgArray`), including quoting, escaping, `typdelim`, and `[l:u]=` prefixes. The goal of "fully eliminating `ArrayEncoding`/`ArrayDecoding`" is unreachable without a fourth leaf primitive (a text reader) and a shared text-literal tokeniser. This is not a fix but a missing quarter of the design — and it is worth designing now, to make sure the `ArrayLeafCodec` interface can carry it (most likely you will need a cursor/tokeniser rather than a whole `String`, and the same one will be reused by the composite parser).

### R4. Registry identity model: bare names, global SPI, no priorities

- `Codec.getTypeName()` is a string without a schema; `codecsByName` is keyed by `pgType.getTypeName().getName()` ([CodecRegistry.java:410](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L410)). A third-party codec named `point` overrides the built-in for **all** schemas — you already tripped over this in the tests (commit 099675898 "stop ServicePointCodec from shadowing built-in point globally"). For the PostGIS scenario it works right up until the user has their own `geometry` type in another schema.
- SPI codecs are loaded into static state (`spiCodecs`) once per classloader and override built-ins with a plain `put` at the creation of every registry ([CodecRegistry.java:268-273](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L268)). A conflict between two SPI codecs over one name is resolved by classpath order — non-deterministically and silently. Exceptions during loading are swallowed without logging (the comment "In production, this should use…" acknowledges this).
- The V2 decision "SPI scope per-Driver" is effectively unimplemented: the scope is static per-classloader.
- `registerByName`/`registerAlias` are public but do not invalidate `oidCache` (unlike `registerCustomCodec`) — afterwards the registry may hand back a stale resolution.
- The single-argument `getBinaryCodec(int oid)`/`getTextCodec(int oid)` call `getByOid(oid, null)` and, for a cold cache, return `FallbackCodec` even for known types. `RangeCodec` uses exactly these (RangeCodec.java:101, 171), throwing away an already-resolved `PgType` — a latent bug.

### R5. Forked paths on the hot read paths

Three decoding systems currently coexist:

| Path | What is used |
|---|---|
| `getObject(i, int[].class)` | the new `ArrayLeafStreamingCodec` → `MultiDimArrayBinary` |
| `getArray(i).getArray()` | the old `ArrayDecoding` with hardcoded per-OID decoders (not codecs!) |
| `getDate/getTime/getTimestamp`, text `getString`, text `getBigDecimal` | a hardcoded switch / inline parsers bypassing codecs |

As a transitional state this is fine (and exactly what you planned), but this is precisely where the parity risks live: for example, `decodeBinaryAs(int[].class)` works, but `decodeTextAs(int[].class)` will fail, because the text path goes through `PgArray.getArray()`, which returns a boxed `Integer[]` ([ArrayLeafStreamingCodec.java:119-138](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayLeafStreamingCodec.java#L119)). Binary/text parity needs to be locked down with tests before converting the next types, otherwise every migrated type will silently change the behaviour of one of the paths.

---

## 3. Missing abstractions

1. **A slice/view for decode** — `decode(byte[] data, int offset, int length, …)` or a lightweight `BinaryView`. Without it, R2 cannot be fixed. (A `ByteBuffer` alternative is acceptable, but costs more in ownership discipline.)
2. **A text tokeniser + `LeafTextReader`** — the missing read counterpart to `appendLeaf` (R3). One cursor parser should serve arrays, composites, and ranges: right now composite has its own parser (`parseCompositeText`), arrays have `ArrayDecoding`, and range has `PGRange.parse`. Three quoting/escaping implementations are three sets of edge-case bugs.
3. **Range subtype in the metadata.** `PgType` does not know `rngsubtype` — `pg_range` is not read at all, and `typelem` for range types is 0. The text path works around this by "not parsing the bounds", the binary path fails (see R6-2). You need either a `PgRangeType` with `subtypeOid` (and at the same time a `PgMultirangeType` with `rngtypid`), or a shared field. Multirange is currently absent entirely — `resolveByTyptype` has no `'m'` branch, so the type falls through to `FallbackCodec` (tolerable for a first version, but it is on your list of goals).
4. **Codec capabilities.** Right now "binary supported" is determined by `instanceof BinaryCodec`, and "binary failed" by throwing an exception from inside. There is no way to ask in advance: can a codec do a binary read but not a binary write? `GeometricCodec` for circle/line is text-only, and the choice of parameter format is decided by the global `binaryTransferSend(oid)` set, which is in no way tied to a codec's capabilities. Once third-party codecs appear, the format must be chosen by the intersection (server-supported × codec-supported × settings) — design in `supportsBinaryRead()/Write()` or an enum-set of capabilities.
5. **Scoped registration instead of `getTypeName()`.** Registration should describe what a codec applies to: OID / (schema, name) / a predicate over `PgType` — plus the source (builtin/SPI/connection) for resolving conflicts. Right now a codec's identity is a single string.
6. **Decode streaming (visitor) is missing.** Encode streaming exists; decode only materialises. For large arrays/composites this is half the win. It can be deferred, but the slice-level signatures (point 1) are the foundation for it too.

---

## 4. Where the public API leaks into internal (specifically)

| Leak | Location |
|---|---|
| `BaseConnection` from `CodecContext.getConnection()` | [CodecContext.java:275](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java#L275) |
| `core.TypeInfo` from `getTypeInfo()`; `TypeInfo` itself now imports `CodecRegistry`/`JavaTypeRegistry`/`PgType` (a core→jdbc cycle) | [TypeInfo.java:8-11](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/core/TypeInfo.java#L8) |
| `PgArray` (a JDBC class) as the result of `decodeBinary` for array codecs | [ArrayCodec.java:78-82](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayCodec.java#L78) |
| `BooleanTypeUtil` in the default methods of a public interface | [BinaryCodec.java:172](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/BinaryCodec.java#L172) |
| `PgType` with public constructors and `Oid.BOX` logic inside them | [PgType.java:52-56](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgType.java#L52) |
| JDBC policy inside the wire context: `prefersJavaTimeFor*`, `convertBooleanToNumeric` in `CodecContext` | [CodecContext.java:52-61](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java#L52) |

The last item is not "remove" but "split": the wire context (charset, server params, a `TimestampUtils` analogue) versus the JDBC read policy. Third-party codecs need the former; the latter is an adapter detail. Also: the context has no server version and no `integer_datetimes`-like parameters — for standalone encode/decode they will be needed.

---

## 5. Performance risks

1. **Per-element copies on decode** — R2, the main item.
2. **`PgConnection.getCodecContext()` allocates a new context on every call** (PgConnection:2101-2106), and `ArrayDecoding.MappedTypeObjectArrayDecoder` calls it **per array element** (ArrayDecoding:453). The context is immutable — cache it on the connection and recreate it on a typemap change.
3. **Hidden encode→decode round-trips**:
   - `createArrayOf(...).getArray(map/index/count)` — a Java array is serialised to a text literal and parsed back (PgArray:206);
   - `PgStruct.getAttributes(Map)` for nested structs — encode to text → `decodeTextAs` (PgStruct:134-141); the same pattern in `PgCallableStatement` (518-524) and OUT `getDate/getTime/getTimestamp` via `result.toString()` (PgCS:549-571);
   - `PgArray.toString()` on a binary-backed array — a full decode + text encode.
   Each of these is both CPU and a potential lossy conversion (float formatting, timestamp parsing).
4. **Double rectangular validation on encode**: `estimateInitialCapacityFor` already calls `computeDimensionLengths` (a full recursive traversal), and then `encode` calls it again ([MultiDimArrayBinary.java:124,162](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java#L124)).
5. **Megamorphic dispatch on the `getInt` path.** The old path was `switch(oid)` + static; the new one is an interface call `codec.decodeAsInt` through a call site shared across all columns. The JIT usually copes (bimorphic per column), but this has to be measured, not assumed. There is already a benchmarks module (`benchmarks/src/jmh`) — add: `ProcessResultSet`-style getInt/getString over text/binary, decode of `int4[]` (1K/100K elements, `int[]` vs `Integer[]`), composite decode, and a comparison against master. Plus `-prof gc` for the alloc rate.
6. `TYPE_ALIASES` — static, process-wide, growing without bound from user strings (TypeInfoCache:563-565). Negative lookups (an unknown OID/name) are not cached — a repeated miss is paid for with up to two queries each time.

Where boxing is **unavoidable per JDBC**, and where it is not: `Array.getArray()` is obliged to return an `Object` (historically a boxed `Integer[]` — cannot be changed), but `getObject(i, int[].class)`, `ResultSet.getInt` over elements via `Array.getResultSet`, and the leaf paths are primitive. The current design splits this correctly.

---

## 6. Correctness risks

1. **`ArrayCodec.encodeText` for an arbitrary `java.sql.Array` does `value.toString()`** ([ArrayCodec.java:197-200](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayCodec.java#L197)). For `PgArray` this is a correct literal; for a third-party `Array` implementation, `com.foo.MyArray@1a2b3c` goes into the SQL with no error. The same in `setArray` (PgPS:1063-1067). The binary branch nearby honestly does `getArray()` — the text branch should do the same.
2. **Binary range decode is broken for any range type**: the subtype is taken from `typelem`, which for a range is 0 → `getPgTypeByOid(0)` → `PSQLException` (RangeCodec:99-101, with no guard, unlike the text path with its documented fallback). This is currently hidden by the fact that range OIDs are not in the default binary transfer set; the first person to enable binary for a range gets an error out of nowhere.
3. **`MultiDimArraySupport.computeDimensions` derives the dimensionality from the class** ([MultiDimArraySupport.java:28-36](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArraySupport.java#L28)). `Object[]{ Object[]{...}, ... }` (where the nesting is visible only at runtime) gets dims=1, and the leaf writer fails with "unsupported leaf" or encodes a garbage element. The old `ArrayEncoding` determined the dimensionality recursively from the first element. This is a regression for `createArrayOf("int4", new Object[]{new Object[]{1,2}})`-style calls — verify it with a test and decide deliberately (support it or give a clear error).
4. **Lower bounds**: decode silently discards them both in the new path (MultiDimArrayBinary:195) and in the old one; encode always writes 1. This is compatible with the old behaviour, but since the goal is "full support for PostgreSQL types", pin the decision down in the javadoc/spec: `getArray()` normalises to 1-based, `getArray(index,count)` interprets index as 1-based regardless of the server-side bounds.
5. **`CompositeCodec` text decode silently tolerates a field-count mismatch** (`Math.min(rawFields.length, expected)`, CC:571-573): extra wire fields are discarded, missing ones become null. After `ALTER TYPE ... ADD ATTRIBUTE` in another session you get silently distorted data instead of an error. The binary path, conversely, does not cross-check against the catalogue at all (a plus for anonymous records, but make sure a mismatch with the metadata is detected somewhere).
6. **The composite text parser does not account for nested parentheses outside quotes** (CC:427-432 scans up to a comma). The server always quotes nested values, so for server output this is fine, but `PgStruct.getValue()` literals assembled by the client, and user input, may be unprotected — a fuzz test against the server's `row(...)::text` is warranted.
7. **`DomainCodec` loses the domain identity and typmod**: the base codec gets the `PgType` of the base type (DC:74-86), and the domain's `typtypmod` is passed nowhere. For `numeric` domains with precision this does not matter yet (encode does not apply typmod), but the decision "a domain transparently unwraps" is worth pinning down as a contract — including `getObject`: a domain user gets the Java type of the base type, and the `DISTINCT` identity is visible only in the metadata.
8. **`EnumCodec` does not support typeMap → Java enum** (it throws for any class other than `String`) — documented, but verify the error is intelligible, because this is the first thing a user will try.
9. `Int4Codec.decodeAsInt(String)` does `data.trim()` — a divergence from the old strict path? The old `PgResultSet.toInt` also touched whitespace, but verify money/locales for parity.
10. **`getString()` for text columns bypasses the codec entirely** (RS:2421-2423) — after migrating all types onto codecs this becomes the last hardcoded path; account for it in the migration plan, otherwise a custom text codec will not see `getString`.

---

## 7. JDBC compatibility gaps

Well covered: `getObject(int, Map)` with qualified/quoted keys (regtype normalisation), `createArrayOf`/`createStruct` via server-side resolution, `CallableStatement` OUT via codec decode of the result, `addDataType`/`PGobject` continue to work, and the updatable ResultSet goes through codecs.

Remaining:

| Case | State |
|---|---|
| `Array.getArray(Map)` on a binary-backed array | `Driver.notImplemented` (PgArray:212-213) |
| `Array.getResultSet(Map)` | `notImplemented` (PgArray:477-479) |
| `SQLInput.readArray()` | not implemented (PgSQLInput:306-323) — and this is exactly your `struct-of-array` scenario via SQLData; `readBlob/readClob/readRef/readRowId` too |
| `setArray(third-party Array)` | encodes `toString()` without validation (see R6-1) |
| `setObject(List)` | "Can't infer the SQL type" — acceptable, but the message could hint at `createArrayOf` |
| `getObject(i, T[].class)` for text results | fails where binary works (parity, see R5) |
| Multirange | `FallbackCodec` → `PGobject` with a string; binary gets `PGUnknownBinary` |
| `Struct` OUT parameters in `CallableStatement` | work via an encode-text→decode round-trip — correct, but fragile |
| Dead code | unreachable `SQLData`/`Struct` branches in `setObject` (PgPS:877-882) |

The "either support it or give a clear error" principle is generally observed — the error wordings are concrete (type, field, OID). Add advice to them ("Use SQLData implementation", "register a codec via …") and debug logging of the codec choice (`FINEST`: oid → codec class + registration source) — this closes your point about "how to explain which codec was chosen".

---

## 8. Concrete proposals (in execution order)

**Before extending to other types (these change contracts):**

1. **Pin down the public perimeter.** Move/duplicate into `org.postgresql.api.codec`: `PgType` → a `PgTypeMeta` interface (oid, name as (schema,name), typtype, typmod, elementOid, arrayOid, baseTypeOid, subtypeOid, fields), `CodecContext` → an interface with the wire part (charset, server params, registry lookup, type lookup) without `getConnection()`. The JDBC policy (prefersJavaTime*, convertBooleanToNumeric) goes into an internal subclass/composition. The lazy-`PgArray` codecs can live in an internal adapter layer rather than in the SPI.
2. **Add slicing to the decode signatures now**: `decodeBinary(byte[] data, int off, int len, PgType, ctx)` with a default delegation to the full buffer (compatible), and the container codecs switch to slice calls. Document the ownership: a borrowed view, valid only for the duration of the call, copy it if you keep it.
3. **Design the text read leaf**: a shared cursor tokeniser for the literal format (array/composite/range) + `readLeafText` in `ArrayLeafCodec`. Without it, "remove ArrayDecoding" will not happen.
4. **Registry: scoped registration and priorities.** The key is an OID or (schema|null, name); the order is: explicit connection > connection-name > SPI > builtin; a conflict between two SPI codecs over one key is a log warning + a deterministic choice (for example, by class name). `registerByName` is either made private or invalidates `oidCache`. Load the SPI with an explicit classloader and log errors.
5. **`PgRangeType`/loading `pg_range`** (+ multirange at least in the metadata), fix binary range (pass the resolved `PgType` into `getBinaryCodec(oid, pgType)`).

**In parallel/afterwards (do not change contracts):**

6. Cache `CodecContext` on the connection; remove the per-element `getCodecContext()` in `MappedTypeObjectArrayDecoder`.
7. Remove the double rectangular validation; cache `GenericArrayLeafCodec` by element OID in the registry.
8. Fix `encodeText(Array)`/`setArray` for third-party `Array` (via `getArray()`); decide the fate of `computeDimensions` for `Object[]`-nested.
9. Parity harness: for every type, a property test "random value → text-encode → server roundtrip → binary-decode == text-decode == original", plus a lower-bounds/null/empty/ragged/0-dim matrix. This is the entry ticket for removing `ArrayEncoding`/`ArrayDecoding`.
10. JMH: `getInt`/`getString` text+binary, `int4[]` decode (`int[]`/`Integer[]`/`getArray()`), composite decode/encode, against master. Control the alloc rate (`-prof gc`).
11. Remove the empty `TypeInfoCacheLoader`, the dead branches in `setObject`, and bound the growth of `TYPE_ALIASES` (bounded or per-connection).

**The migration path for the old classes** — your plan "leave them, migrate gradually, delete them" works; I would skip the intermediate step "make `ArrayEncoding` a thin wrapper over codecs": it is cheaper to switch the registry from `ArrayCodec.INSTANCE` to `ArrayLeafStreamingCodec(name, leaf)` as the leaf codecs become ready (as already done for `_int4`), and `ArrayDecoding` will die together with the text reader and the migration of `PgArray` onto codecs. The observable behaviour to preserve: boxed types from `getArray()`, 1-based slice indexing, the `getBaseTypeName()` format, and the error texts where tests depend on them (`ArrayEncoding` produces "the same error message it always has" — you already account for this in [ArrayCodec.java:126-131](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayCodec.java#L126)).

---

## 9. Questions worth resolving before extending

1. What minimal slice/view decode contract are you prepared to pin down in the public API: `(byte[], off, len)` or `ByteBuffer`? (It affects all codecs, and is decided once.)
2. Codec identity at registration: is a (schema-optional name | OID) pair + source sufficient, or do you need a predicate over `PgType` (for PostGIS, where the OID is per-database)?
3. The domain contract: "transparently unwrap, identity only in the metadata" — yes/no? (Right now it is de facto yes; pin it down.)
4. `Array.getArray()` after migration: does it stay boxed `Integer[]` forever (compatibility), with primitive only via `getObject(i, int[].class)`?
5. The fate of the `getString`/`getDate`/`getTimestamp` hardcoded paths: migrate them onto codecs in this branch, or declare them out of scope?
6. Multirange: metadata-only in the first version (Fallback on read) or a full codec?
7. SPI: keep static per-classloader (and document it honestly) or build a genuine per-Driver/per-DataSource scope, as recorded in the V2 decisions?
8. Do you need a `CodecContext` snapshot of server parameters (server_version, integer_datetimes, DateStyle) already now — for the sake of a future offline mode?

If your decisions on points 1-4 of section 8 line up with my proposals, then the next sensible step is to go down to the level of code findings on specific files (I have already marked candidates: `RangeCodec`, the `CompositeCodec` text parser, `MultiDimArraySupport.computeDimensions`, the registry invariants).
