## Verdict

**The design is viable, but it needs targeted changes before scaling to the rest of the types.**

The core is correct. Container composition through the single entry point `ctx.getCodecs().getByOid()` works: `CompositeCodec`, `RangeCodec`, and `DomainCodec` do not reinvent the resolution of child codecs but delegate it to the registry. This is exactly the abstraction you need, and the tests already prove complex scenarios — array-of-struct, struct-of-array, 4 levels of nesting, binary+text parity, SQLData, a codec via ServiceLoader, and identifier normalisation (45/45). This is a mature branch, not a sketch.

But three things are worth fixing **before** adding `int2/int8/float4/float8/bool/uuid/temporal` and **before** stabilising the API, otherwise duplication and internal-type leaks will freeze into the public contract:

1. the public SPI leaks internal types (`org.postgresql.core.*`, `org.postgresql.jdbc.*`);
2. decode has no zero-copy slice, even though encode already has streaming;
3. three paths coexist in the array layer, and the new walker is not on the hot read path.

Right now the entire SPI is marked `@Experimental` — this is the right and cheapest moment for such changes.

---

## What has already been done well

So that the criticism reads in context:

- **Centralised resolution of child codecs.** A single gateway `CodecRegistry.getByOid()` → `resolveByTyptype()`. The containers remain stateless singletons that take a `PgType` at call time.
- **Streaming encode with back-patching.** `StreamingBinaryCodec(..., OutputStream)` + `BackpatchingBinarySink` remove the per-element `byte[]` during nested encoding. This is a genuine optimisation.
- **Depth protection.** `CodecDepth` closes off a DoS on recursive types.
- **Identifier normalisation through OID.** `IdentifierNormalizingTypeMap` resolves Lukas Eder's feedback about qualified/quoted names in `Map<String,Class<?>>` elegantly — it resolves both the lookup key and the keys of the user map through `regtype` and compares by OID.
- **Primitive specialisation on scalar read.** `decodeAsInt/Long/...` remove boxing on the hot scalar read.

---

## 1. Main architectural risks

### R1. Three parallel array paths, and the new one is not on the hot path

Right now the following live simultaneously:

1. legacy `ArrayEncoding` / `ArrayDecoding` (directly from `PgArray`);
2. `ArrayCodec` — a wrapper that, for native types, calls legacy, and uses the new `MultiDimArray*` only for `Object[]` from composite/custom elements;
3. `ArrayLeafStreamingCodec` — a new specialised path, registered only for `_int4`.

Critically: `ArrayLeafStreamingCodec.decodeBinary` and `ArrayCodec.decodeBinary` return a **lazy `PgArray`** ([ArrayCodec.java:78](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayCodec.java)), and `PgArray.getArray()` decodes through legacy `ArrayDecoding`. The new `MultiDimArrayBinary.decode` fires only in `decodeBinaryAs(int[].class/Integer[].class)`. That is, on a normal `getArray()` / `getObject()` the new walker **does not execute** — there is no gain, and you cannot remove `ArrayEncoding`/`ArrayDecoding` because the new path itself relies on them.

Conclusion: the goal of "getting rid of `ArrayEncoding`/`ArrayDecoding`" is so far unreachable by design. You need to pick one model and put it on the read path.

### R2. Two competing array-codec abstractions

`ArrayCodec` (universal, composes the element codec by OID) versus `ArrayLeafStreamingCodec` (a separate codec per element OID, with a hand-written `ArrayLeafCodec`). Both implement `StreamingBinaryCodec, StreamingTextCodec`, both go into `MultiDimArray*`. It is unclear which is canonical. If you scale through `ArrayLeafStreamingCodec`, you will get a separate codec and a separate leaf class per `int2/int8/float4/float8/bool/...` — exactly the code proliferation you fear. See proposal P4.

### R3. Resolution order breaks domain-over-array

`resolveByTyptype()` checks `isArray()` (that is, `typcategory == 'A'`, [PgType.java:390](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgType.java)) **before** `isDomain()` ([CodecRegistry.java:486](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java)). A domain inherits the `typcategory` of the base type, so `CREATE DOMAIN d AS int[]` gives `typtype='d'`, `typcategory='A'` → it lands in `ArrayCodec`, not in `DomainCodec`. Then `ArrayCodec.streamBinaryArrayViaCodec` takes `arrayType.getTypelem()`, but a domain has `typelem == 0` → the element codec is not found. `typcategory` is a weaker signal than `typtype`, and it should be checked last.

### R4. Multirange is not supported at all

`typtype='m'` is absent from `resolveByTyptype()`, and `*multirange` names are not registered → `FallbackCodec`. It is in the goals, but not in the code. A silent fallback is worse than an explicit error.

---

## 2. Missing abstractions

- **`api.codec.TypeDescriptor`** — a public read-only metadata interface. Right now the entire SPI is tied to the concrete `org.postgresql.jdbc.PgType`.
- **Decode slice** — symmetric to `StreamingBinaryCodec`. A signature of the form `decodeBinary(byte[] data, int offset, int length, ...)` or a `ByteBuffer`/cursor is needed, so that containers do not do a `copyOfRange` on each element.
- **`CodecContext` as an interface.** Right now it is a `final` class, constructed only from `BaseConnection`. This blocks offline mode (see L3).
- **A public resolver of child codecs.** So that a third-party container codec can resolve nested types without touching `CodecRegistry` and `TypeInfo` (both effectively internal).
- **Explicit capability flags** (`supportsBinaryRead/Write`, `textRead/Write`). Right now the capability is inferred through `instanceof BinaryCodec/TextCodec` and "is this streaming?" — implicitly and not self-documenting.
- **An SPI conflict model** — priorities/duplicate detection (see CG in gaps).

---

## 3. Where the public API leaks the internal implementation

This is the most important section, because fixing it after third-party dependencies appear is the most expensive of all.

- **SPI interfaces from `api.codec` reference `org.postgresql.jdbc.PgType` and `org.postgresql.jdbc.CodecContext`** — the package where all the internal machinery sits. The `api.codec` package is clean by name but not by references.
- **`CodecContext` (public, but in `org.postgresql.jdbc`) hands out through getters:** `BaseConnection`, `TypeInfo`, `Encoding` (all `org.postgresql.core`), plus `CodecRegistry`, `JavaTypeRegistry`, `TimestampUtils`. A third-party codec that needs metadata or a child codec inevitably imports `org.postgresql.core.*` — exactly the "internals" (next to `Field`, `Tuple`, `QueryExecutor`) that you want to hide.
- **`PgType` is constructed from `ObjectName` and `List<PgField>`** — both internal. For offline/standalone this is an unworkable boundary.
- **`BackpatchingBinarySink` is package-private** ([BackpatchingBinarySink.java:18](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/BackpatchingBinarySink.java)). A third-party `StreamingBinaryCodec` does not see it and cannot do an `instanceof`, so it always gets the slow `byte[]` path. In effect, the streaming gain is closed off for third parties and available only to built-in codecs in the same package.

---

## 4. Performance risks

- **PR1. `GenericArrayLeafCodec.readLeaf` does an `Arrays.copyOfRange` on each element** ([GenericArrayLeafCodec.java:116](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java)). Any non-specialised element (composite, uuid, numeric, text, int8…) allocates a copy on decode. Only `_int4` avoids the copy. This is a direct consequence of the missing decode slice (the L-abstraction above).
- **PR2. Decode goes through legacy `PgArray`/`ArrayDecoding`** even for `_int4` on `getArray()` — the typed leaf gain is not realised on the hot path (see R1).
- **PR3. There is no primitive specialisation on encode.** Encoding `Integer[]`/`Object[]` boxes each element; the SPI has no `encodeInt`/`encodeLong`. Specialisation exists only on the decode scalar.
- **PR4. The specialised leaf re-inlines the scalar logic.** `Int4ArrayLeafCodec.writeLeaf`/`appendLeaf` duplicate int4 encoding instead of calling `Int4Codec`. For int this is tolerable; for `numeric`/`timestamptz` you cannot re-inline, so they will go into the boxing generic path. The performance story forks, and the goal of "not proliferating logic for int2/int4/int8/float4/float8/bool" is not achieved.
- **PR5.** `MultiDimArrayBinary` uses reflection on the outer dimensions — acceptable, the cost is bounded by the product of the outer dimensions, not the number of elements.

---

## 5. Correctness risks

- **CR1.** domain-over-array (R3) and **CR2.** multirange (R4) — above.
- **CR3. Lower bounds are lost silently.** encode always writes lower bound = 1 ([MultiDimArrayBinary.java:132](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java)), decode reads and ignores it (line 195). This matches legacy JDBC behaviour, but non-1 bounds disappear — this needs to be documented explicitly.
- **CR4. Different quoting in the text array.** `GenericArrayLeafCodec.appendLeaf` quotes every non-null element, `Int4ArrayLeafCodec` does not, and legacy leaves numbers without quotes. PostgreSQL accepts quoted numbers, so functionally it is fine, but these are different bytes on the wire. Check that the tests do not assert exact literals, and the edge cases (an empty-string element, elements with a separator/braces).
- **CR5. The new generic path has no text-read leaf.** Text decode of arrays is entirely on legacy `ArrayDecoding`. The path is asymmetric: it writes text but does not read it.
- **CR6. Dropped attributes in `CompositeCodec` (`attisdropped`).** The agent did not find filtering. In the binary record format, dropped columns arrive as NULL with their own OID; if the field list in text and binary treats them differently, decode desynchronises by position. This is a classic composite bug — **must be checked** before expanding.
- **CR7. `registerByClass(codec.getDefaultJavaType(), codec)` for the SPI** ([CodecRegistry.java:271](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java)). An SPI codec with a default type of `String`/`Object` will silently override the built-in encoding of `String`. Collision detection is needed.

---

## 6. JDBC compatibility gaps

- **CG1. `setObject(i, obj)` without a type hint for an arbitrary DTO** does not look up a codec by Java class (`findCodecFor` exists but is not wired into this path) → error. Writing a custom DTO requires an explicit type.
- **CG2. CallableStatement OUT parameters** are decoded "in passing" through the ResultSet (works), but `registerOutParameter` does not pick a codec by type; named getters are partially unimplemented (plan T2). Tests have been added.
- **CG3. Offline encode/decode** is blocked by the `CodecContext` design (L3).
- **CG4. `addDataType` and the codec registry are different mechanisms.** A `PGobject` subclass registered by the user is not used as a codec; it lives only in the legacy-fallback `getObject`. The relationship is worth documenting.
- The good news: slices `getArray(index,count)`, `getResultSet`, `getArray(Map)`, `createArrayOf`/`createStruct`, and updatable ResultSet for Struct/array — are already covered by tests. `getObject(i, T[].class)` is deliberately deferred (decision V2 #7).

---

## 7. Concrete architecture proposals (before scaling)

- **P1. Introduce `api.codec.TypeDescriptor` (metadata) and `api.codec.CodecContext` (an interface).** Take the SPI off `org.postgresql.jdbc.PgType`/`CodecContext`, leaving the internal implementations. The most important change: once third-party dependencies appear it can no longer be done, and right now the API is `@Experimental`.
- **P2. Add a decode slice:** `decodeBinary(byte[] data, int offset, int length, TypeDescriptor, CodecContext)` (or a `BinaryReader` cursor). Containers pass a slice instead of `copyOfRange`. Symmetric to `StreamingBinaryCodec`.
- **P3. Raise `BackpatchingBinarySink` (or an equivalent) into `api.codec`**, otherwise the streaming SPI works only for built-ins.
- **P4. Collapse to a single array model:** one universal `ArrayCodec` that composes element codecs, plus an **optional** fast-path SPI interface (the current `ArrayLeafCodec`) that a scalar codec can implement for `int[]/long[]/...`. Then `int2/int8/float4/float8/bool` get a fast path by implementing one small interface on the already-existing scalar codec — without a parallel codec and without re-inlined scalar logic. This is a direct answer to "how not to proliferate code".
- **P5. Add primitive encode** to the same fast-path leaf, so that `int[]` is encoded without boxing.
- **P6. Fix the precedence in `resolveByTyptype`:** typtype checks (composite/domain/enum/range/multirange) before the typcategory array check; or detect an array only when `typtype=='b' && typcategory=='A'`.
- **P7. Add a multirange codec** (`typtype='m'`) that composes the range codec, or an explicit error "not supported, because…".
- **P8. Define the SPI conflict policy:** duplicate detection by name, an explicit priority (`getPriority()` or an ordered ServiceLoader), a log on overriding a built-in or another SPI. Decide whether an SPI may override a built-in at all.
- **P9. Schema-qualified registration.** `codecsByName` is keyed by simple name, so identical names in different schemas collide. At minimum — document that name registration is schema-agnostic, and OID registration (`registerByOid`) is the way to separate the types.

---

## 8. Questions to resolve before expanding

1. **One array codec or a codec per element OID?** (I recommend one + a fast-path SPI, P4).
2. **`CodecContext`/`PgType` — part of the stable API, or do we introduce `api.codec` abstractions?** (I recommend the second, P1).
3. **Ownership of `byte[] data` on decode:** borrowed (the codec is not allowed to retain it) or owned? Document it, then add the slice (P2).
4. **Domain identity:** unwrap to the base (the current `DomainCodec`) or keep it? For `getObject` unwrapping is fine, for type names in `Struct`/SQLData and for domain-over-array — not.
5. **Composite dropped attributes** — handled? (CR6, requires checking in the code/test).
6. **Multirange — in scope for v1?**
7. **The SPI override and priority policy** (P8).
8. **Offline API:** fix `CodecContext` as an interface now, so as not to block it (CG3).
9. **Text read of arrays:** will the new path ever take parsing on itself, or does delegation to `ArrayDecoding` remain forever? (CR5, R1).

---

The architecture is acceptable, so the logical next step is to move on to code-level findings. I am ready to:

- **(A)** verify CR6 (dropped attributes in `CompositeCodec`) and CR4 (quoting) against the code — these are the most likely "silent" bugs; I can bring up PostgreSQL via Docker and run binary/text parity tests;
- **(B)** prototype P4 on `int8` or `bool` — to show that a fast path as an optional interface on a scalar codec removes the code proliferation;
- **(C)** sketch the signatures of `api.codec.TypeDescriptor` / `CodecContext` / the decode slice (P1–P3), to fix the public boundary before expanding.
