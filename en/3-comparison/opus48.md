# Comparison of the architecture reviews for the Codec API in pgjdbc

## 1. Short verdict

**The most useful answer is Fable 5.** It gives the fullest risk map, separates the fixes that change the contract from those that can wait, and is the only one to propose a concrete risk-reduction plan (a parity harness as the entry ticket for removing `ArrayEncoding`/`ArrayDecoding`, a list of JMH benchmarks, a migration order). I found no hallucinated claim in it.

**The best-evidenced on a single finding is Opus 4.8.** Its unique R3 (the order of checks in `resolveByTyptype`: `isArray()` before `isDomain()`, which sends domain-over-array off to `ArrayCodec`) is the most precise and verifiable bug across all three answers, confirmed both by the code and by PostgreSQL semantics. Its P4 (collapse to a single `ArrayCodec` plus an optional fast-path interface) is also the clearest answer to the question "how not to multiply the code".

**GPT 5.5 is the most compact and error-free**, but thinner on coverage: fewer file-level findings, no benchmark plan, no migration order. On the other hand, it is the only one with the finding about `unregisterCustomCodec`/`resetCustomCodecs`, which remove the name without restoring the overridden built-in, and about the contract "a codec is cached in `Field`".

**Where all three agree:** the public SPI boundary leaks internal types; type identity in the registry is a bare name without a schema; decode has no zero-copy slice (a copy on every element/field); range/multirange metadata is not ready; the new array walker is not on the hot read path; lower bounds are lost; offline mode is blocked by `CodecContext` being a `final` connection-bound class.

**Where they diverge:** Opus uniquely found the domain-over-array precedence bug and proposed the strongest model "one array codec + fast-path SPI". Fable uniquely worked through the perf details (per-call `getCodecContext`, double rectangular validation, encode→decode round-trips) and gave a test plan. GPT uniquely found the loss of the built-in on unregister/reset and the caching contract in `Field`. On `attisdropped`, Fable and Opus directly contradict each other — the check below decides in Fable's favour.

---

## 2. Methodology

Three files were compared: [fable5.md](../2-review-execution/fable5.md), [gpt55.md](../2-review-execution/gpt55.md), [opus48.md](../2-review-execution/opus48.md) against the rubric from [design-review-prompt.md](../1-review-prompt-creation/design-review-prompt.md).

The current state of the code matches what the models looked at: `GenericArrayLeafCodec`, `Int4ArrayLeafCodec`, `MultiDimArraySupport`, `ArrayLeafStreamingCodec`, `BackpatchingBinarySink` are present as untracked files on top of commit `61d61d000`.

**Read and checked against the code:**
- public SPI: [Codec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/Codec.java), [BinaryCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/BinaryCodec.java), [CodecContext.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java);
- registry: [CodecRegistry.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java), classification [PgType.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgType.java);
- array path: [ArrayCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayCodec.java), [MultiDimArrayBinary.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java), [GenericArrayLeafCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java), [Int4ArrayLeafCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/Int4ArrayLeafCodec.java), [ArrayLeafStreamingCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/ArrayLeafStreamingCodec.java), [MultiDimArraySupport.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArraySupport.java);
- container/scalar codecs: [RangeCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java), [DomainCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/DomainCodec.java), [EnumCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/EnumCodec.java), [BackpatchingBinarySink.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/BackpatchingBinarySink.java);
- JDBC adapters and infrastructure: [IdentifierNormalizingTypeMap.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/IdentifierNormalizingTypeMap.java), `PgArray`, `PgConnection.getCodecContext`, `ArrayDecoding`, the `attisdropped` filter in `TypeInfoCache`.

**Left without a code check (marked "not checked" below):** some of Fable's perf claims about encode→decode round-trips in `PgStruct`/`PgCallableStatement`, the growth of `TYPE_ALIASES`, megamorphic dispatch on `getInt`, `getString()` bypassing the codec; GPT's claim about `registerOutParameter(String,...)`; Opus's CG1 claim about `setObject` on a DTO with no type hint. They are plausible and consistent with the architecture, but I did not open them line by line.

**A note on code drift:** I did not run the PostgreSQL record_send semantics (dropped columns are excluded from the binary format) on a server — this is knowledge of the specification, and Opus's CR6 analysis rests on it.

---

## 3. Claims matrix

Severity: B=blocker, H=high, M=medium, L=low. The verification status is in the last column.

| # | Claim (brief) | Category | F5 | O48 | G55 | Sev | Code check | Status |
|---|---|---|---|---|---|---|---|---|
| C1 | SPI `api.codec.*` pulls in `jdbc.PgType`/`CodecContext` | SPI boundary | ✓ | ✓ | ✓ | B | BinaryCodec.java:9-10,60 | confirmed |
| C2 | `CodecContext.getConnection()`→`BaseConnection`, `getTypeInfo()`→`core.TypeInfo` | SPI boundary | ✓ | ✓ | ✓ | B | CodecContext.java:275,285 | confirmed |
| C3 | Offline mode blocked: `CodecContext` final, connectionless ctor package-private, `withTypeMap` throws | SPI boundary | ✓ | ✓ | ✓ | H | CodecContext.java:41,155,218 | confirmed |
| C4 | `decodeAsBoolean` calls `jdbc.BooleanTypeUtil` from the public interface | SPI boundary | ✓ | – | – | M | BinaryCodec.java:172 | confirmed |
| C5 | Registry keys codecs by bare name; OID resolution via `getTypeName().getName()` → collision of identical names across schemas | registry | ✓ | ✓ | ✓ | H | CodecRegistry.java:84,410; commit 099675898 | confirmed |
| C6 | `resolveByTyptype`: `isArray()` (typcategory='A') checked before `isDomain()` → domain-over-array goes to ArrayCodec, typelem=0 breaks resolution | scalar/registry | – | ✓ | – | H | CodecRegistry.java:486→494; PgType.java:372,390 | confirmed |
| C7 | Multirange (`typtype='m'`) not supported, `*multirange` names not registered → Fallback | ranges | ✓ | ✓ | ✓ | M | CodecRegistry.java:484-506 | confirmed |
| C8 | Decode without a slice: `copyOfRange`/`new byte[]` on every element/bound/field | performance | ✓ | ✓ | ✓ | H | GenericArrayLeafCodec.java:116; RangeCodec.java:120,139 | confirmed |
| C9 | The new `MultiDimArray*` walker is not on the hot path: `getArray()`/`getObject()` go through `PgArray`→`ArrayDecoding` | array | ✓ | ✓ | ✓ | H | ArrayCodec.java:81,250; ArrayLeafStreamingCodec.java:94,113 | confirmed |
| C10 | Text reads of arrays are missing in the new architecture; delegated to legacy `ArrayDecoding` | array | ✓ | ✓ | – | H | ArrayCodec.java:272-273; ArrayLeafStreamingCodec.java:129-130 | confirmed |
| C11 | Lower bounds: encode always writes 1, decode ignores them | array correctness | ✓ | ✓ | ✓ | M | MultiDimArrayBinary.java:132,195 | confirmed (trade-off) |
| C12 | Two competing array abstractions: `ArrayCodec` vs `ArrayLeafStreamingCodec` | array/maintainability | ~ | ✓ | – | M | both in registry; CodecRegistry.java:186,192 | confirmed |
| C13 | The special leaf re-inlines scalar logic (`Int4ArrayLeafCodec` does not call `Int4Codec`); does not scale to numeric/timestamptz | perf/maintainability | ✓ | ✓ | – | M | Int4ArrayLeafCodec.java:57-58,94 | confirmed |
| C14 | Inconsistent quoting of text elements: generic quotes, int4 does not | array correctness | – | ✓ | – | L | GenericArrayLeafCodec.java:152-162 vs Int4ArrayLeafCodec.java:125 | confirmed (trade-off) |
| C15 | No primitive encode on the SPI: `Integer[]`/`Object[]` box | performance | – | ✓ | – | M | BinaryCodec: only decodeAs* | confirmed |
| C16 | `BackpatchingBinarySink` package-private → a third-party container codec cannot use backpatch | SPI boundary/perf | – | ✓ | – | M | BackpatchingBinarySink.java:18 | confirmed (with a caveat) |
| C17 | Binary range broken: subtype from `typelem`=0 with no guard; single-arg `getBinaryCodec` discards the resolved type | ranges correctness | ✓ | – | ✓ | H | RangeCodec.java:99-101 vs 279-281 | confirmed (latent) |
| C18 | `getBinaryCodec(int)`/`getTextCodec(int)` → `getByOid(oid,null)` → Fallback on a cold cache | registry | ✓ | – | – | M | CodecRegistry.java:540-543 | confirmed |
| C19 | SPI static per-classloader; overrides the built-in with a plain `put`; `registerByClass(getDefaultJavaType)` can shadow String; loading errors are swallowed | registry | ✓ | ✓ | ✓ | M | CodecRegistry.java:99-100,138-145,268-273 | confirmed |
| C20 | `unregisterCustomCodec`/`resetCustomCodecs` remove the name without restoring built-in/SPI | registry | – | – | ✓ | M | CodecRegistry.java:337-342,350-357 | confirmed |
| C21 | `registerByName`/`registerAlias` do not invalidate `oidCache` | registry | ✓ | – | – | L | CodecRegistry.java:280-292 vs 329 | confirmed |
| C22 | `DomainCodec` loses domain identity and typmod (delegates to the base type) | domains | ✓ | ~ | – | M | DomainCodec.java:54-60,74-86 | confirmed (trade-off) |
| C23 | `EnumCodec` does not decode into a Java enum (only String/Object) | enums | ✓ | – | – | L | EnumCodec.java:89-92,101-104 | confirmed |
| C24 | `ArrayCodec.encodeText` for a third-party `Array` does `toString()` → garbage in SQL | array correctness | ✓ | – | ✓ | M | ArrayCodec.java:197-200 vs 103 | confirmed |
| C25 | `computeDimensions` counts rank by the static class → a runtime-nested `Object[]` gets dims=1 | array correctness | ✓ | – | – | M | MultiDimArraySupport.java:28-36 | confirmed (regression — not checked) |
| C26 | `CompositeCodec` text silently tolerates a field-count mismatch (`Math.min`) | composite | ✓ | ~ | – | M | CompositeCodec.java:571-572 | confirmed |
| C27 | Dropped attributes (`attisdropped`) can desync composite decode | composite | ✗(there is a filter) | ✓(to verify) | – | M | TypeInfoCache.java:864 filters | **not reproducible** |
| C28 | `getCodecContext()` allocates a new context on every call; called per-element in `ArrayDecoding` | performance | ✓ | – | – | M | PgConnection.java:2102; ArrayDecoding.java:453,469 | confirmed |
| C29 | Double rectangular validation on encode (estimate + encode) | performance | ✓ | – | – | L | MultiDimArrayBinary.java:124,162 | confirmed |
| C30 | `PgArray.getArray(Map)`/`getResultSet(Map)` on binary → `notImplemented` | JDBC gap | ✓ | ~ | ✓ | M | PgArray.java:213,478 | confirmed |
| C31 | `IdentifierNormalizingTypeMap` resolves qualified/quoted names via regtype→OID (Lukas Eder's fix) | JDBC (positive) | ✓ | ✓ | ✓ | — | IdentifierNormalizingTypeMap.java:96-112 | confirmed |
| C32 | A codec is cached in `Field`; `registerCodec` affects future queries, not already-initialised fields | registry contract | ~ | – | ✓ | L | claimed by two, code not opened | partially confirmed |
| C33 | binary/text parity not pinned by tests before type conversion | tests | ✓ | ~ | – | H | follows from C9/C10 | confirmed (risk) |

Legend: ✓ — raised, ~ — touched indirectly, – — did not raise, ✗ — claimed the opposite.

---

## 4. Confirmed shared conclusions

**Found by all three and confirmed against the code:**

- **C1/C2 — the SPI leaks internal types.** This is the main consensus. `api.codec.BinaryCodec` takes `org.postgresql.jdbc.PgType` and `CodecContext`; `CodecContext` returns `BaseConnection` and `core.TypeInfo`. Any third-party codec that needs metadata or a child codec imports the internal core. Cheap to fix now, while everything is under `@Experimental`.
- **C5 — type identity is insufficient.** Runtime resolution goes by OID (via `oidCache`/`explicitOidCodecs`), but built-in and custom codecs are found through `pgType.getTypeName().getName()` — a bare name without a schema. A `point` type in someone else's schema gets the built-in `PointCodec`. This is not theory: commit `099675898` suppresses exactly this effect in the tests.
- **C8 — decode without a slice.** Encode already streams through backpatch; decode has no symmetry: a copy on every array element, every range bound, every composite field. For `array-of-struct-of-array` this is O(payload × depth). It changes the signature of every decode method, so it must be decided before the remaining ~30 codecs are written.
- **C9 — the new walker is not on the hot path.** `getArray()`/`getObject()` return a `PgArray`, which decodes through legacy `ArrayDecoding`. `MultiDimArrayBinary.decode` runs only in `decodeBinaryAs(int[].class/Integer[].class)`. The goal "remove `ArrayEncoding`/`ArrayDecoding`" is currently unattainable by design — the new path itself relies on them (C10).
- **C7 — multirange is missing**, **C17 — binary range does not resolve the subtype** (a range's typelem is 0, and `pg_range` is not loaded).

**Found by several and confirmed:** C3 (offline blocked — Fable, Opus), C10 (no text-read leaf — Fable, Opus), C11 (lower bounds — all), C12/C13 (two array abstractions and re-inlined scalar logic — Fable, Opus), C24 (`toString()` for a third-party `Array` — Fable, GPT), C19 (SPI overlay — all), C30 (`getArray(Map)` binary not implemented — all).

---

## 5. Unique useful findings

**Only Opus 4.8:**
- **C6 — the domain-over-array precedence bug (R3).** The most valuable single finding. `resolveByTyptype` checks `isArray()` (which is `typcategory=='A'`) before `isDomain()` (`typtype=='d'`). A domain over an array inherits `typcategory='A'`, but `typelem=0`, so it goes off to `ArrayCodec`, where `streamBinaryArrayViaCodec` takes `getTypelem()=0` and finds no element codec. Confirmed both by the code and by PostgreSQL semantics (a domain inherits the typcategory of the base type). The correct fix: check `typtype` (composite/domain/enum/range/multirange) before `typcategory`, or treat as an array only when `typtype=='b' && typcategory=='A'`.
- **C15 — no primitive encode on the SPI**, **C16 — `BackpatchingBinarySink` package-private** (a third-party streaming codec sees only `OutputStream`, container-level backpatch is unavailable to it; for a scalar element codec, streaming through `OutputStream` does still work — a caveat).

**Only GPT 5.5:**
- **C20 — `unregisterCustomCodec`/`resetCustomCodecs` lose the built-in.** `codecsByName.remove(typeName)` removes the entry entirely; if a custom codec overrode the built-in by name, the built-in does not come back after the unregister. A real bug in the reset scenario of a connection pool.
- **C32 — the contract of caching a codec in `Field`** (claimed by Fable too, I did not check it line by line): registering a codec affects future queries, not already-initialised columns. This is a contract that should be pinned in the javadoc.

**Only Fable 5:**
- **C28 — per-call/per-element `getCodecContext()`**, **C29 — double rectangular validation**, **C25 — `computeDimensions` by the static class**, **C18 — single-arg `getBinaryCodec` returns Fallback on a cold cache**, **C21 — `registerByName` does not invalidate `oidCache`**, **C22 — loss of domain identity/typmod**, **C23 — enum→Java enum not decoded**, **C24 — `toString()` for a third-party `Array`** (shared with GPT).

---

## 6. Doubtful or false claims

- **C27 (Opus CR6, dropped attributes) — not reproducible on checking.** Opus honestly marked it as "must verify", but presented it as a likely "silent" bug. The filter exists: `TypeInfoCache.java:864` loads composite fields with `AND NOT a.attisdropped`. On top of that, PostgreSQL `record_send` excludes dropped columns from the binary format (it sends `validcols`, not `natts`), and `row(...)::text` omits them too. The field list and the wire format filter dropped columns consistently, so there is no positional desync. **Here Fable is right** ("lazy composite fields with an `attisdropped` filter on the server"), and Opus's framing as a bug candidate is an overestimate. Worth closing with a regression test, but it is not an active bug.
- **C25 (Fable, "a regression for `createArrayOf`")** — the fact `dims=1` for a runtime-nested `Object[]` is confirmed, but the claim that this is specifically a regression relative to the old `ArrayEncoding` I did not cross-check against the `ArrayEncoding` code. The fact itself is confirmed; the word "regression" is not checked.
- **C11/C14/C22 — these are design trade-offs, not bugs.** Lower bounds=1 and the quoting of numbers were all presented correctly by all three ("document it"), but it matters not to treat them as defects: lower-bounds normalisation is compatible with pgjdbc's old JDBC behaviour, and PostgreSQL accepts quoted numbers. The domain "transparently unwraps" — this is a deliberate contract, not a loss (the only question is whether to pin it).
- **Fable's perf claims about encode→decode round-trips** (`PgStruct.getAttributes`, `PgCallableStatement` OUT, `PgArray.toString`) — plausible and consistent with the fact that binary-backed `PgArray.getArray(Map)` is not implemented (C30), but I did not open the specific lines. I mark them as **not checked**, not as confirmed.
- **No hallucinations found** in any of the three answers. All the file:line references I checked land on real code (with an allowance of ±1-2 lines for the drift of the untracked files).

---

## 7. Missed topics

Rubric points that no one covered, or covered superficially (my findings, confirmed against the code/specification):

- **Simple query mode vs extended query mode — missed by all.** The rubric asked about this explicitly. In the simple query protocol the server always returns text, and binary codecs are not used at all. This directly reinforces C10: as long as text reads of arrays live in legacy `ArrayDecoding`, simple query mode depends entirely on the old path, and "remove `ArrayDecoding`" cannot happen without a full text-read leaf. No model connected these two facts.
- **Name resolution in the registry ignores `search_path` — touched only indirectly.** `IdentifierNormalizingTypeMap` uses the server's `regtype` (search_path-aware) — this is good, and all three noted it. But the registry itself (`codecsByName`, the alias table) keys by unqualified names without accounting for the schema and `search_path`; for built-in this is fine, for future custom codecs it is not. No one explicitly separated these two levels.
- **Capability-based format choice — only Fable (#4).** Right now "a codec supports binary" is determined by `instanceof BinaryCodec`, and "it could not" by an exception from inside. For third-party codecs the format should be chosen by the intersection (server-supported × codec-supported × `binaryTransfer*` settings). Opus and GPT did not raise this.
- **Negative caching and the growth of `TYPE_ALIASES` — only Fable.** Lookup misses (an unknown OID/name) are not cached; `TYPE_ALIASES` grows process-wide. I did not check it line by line, but the topic is real and was left out of GPT/Opus.
- **Enum labels with special characters and their quoting in arrays/composite — missed by all.** The rubric asked; `EnumCodec` simply passes strings through, and no one worked out the behaviour inside an array/composite literal.

---

## 8. Comparison of the plans

**The criterion is practical fitness for a fast and scalable implementation, not the elegance of the wording.**

**Fable 5 — best overall.** The only one to:
- split the work into "change the contract, do before scaling" (the API perimeter, the decode slice, the text-read leaf, registry identity, `pg_range`) and "do not change the contract, can wait";
- propose a **parity harness** as the entry ticket for removing legacy: a property test "random value → text-encode → server roundtrip → binary-decode == text-decode == original" plus a matrix of lower-bounds/null/empty/ragged/0-dim;
- give a concrete JMH set (`getInt`/`getString` text+binary, `int4[]` 1K/100K as `int[]`/`Integer[]`/`getArray()`, composite, `-prof gc`);
- describe the migration order: do not make `ArrayEncoding` a thin wrapper, but switch the registry from `ArrayCodec.INSTANCE` to a leaf codec as each becomes ready (as already done for `_int4`), and `ArrayDecoding` will die together with the text reader.

The weak spot — in places too many items at once, no single "if you do only one thing".

**Opus 4.8 — the best architectural resolution of the array model.** Its **P4** is the most direct answer to "how not to multiply the code for int2/int4/int8/float4/float8/bool": one universal `ArrayCodec` composing element codecs, plus an **optional** fast-path interface (the current `ArrayLeafCodec`) that a scalar codec implements for `int[]/long[]/...`. This removes C12 (two abstractions) and C13 (re-inlined logic) with one decision. Plus the pointed P6 (fix the precedence) and a clear menu of next steps (A: verify CR6/CR4 on a server; B: prototype P4 on int8/bool; C: sketch the signatures). Weaker on tests/benchmarks and migration order.

**GPT 5.5 — a correct skeleton, but thinner.** It correctly names the major decisions (a public `TypeDescriptor`, registry layers, a raw standalone API `RawValue`/`CodecSession`, loading `pg_range`/`pg_multirange`), and uniquely catches C20 and C32. But it ran nothing, gave no benchmarks, no work order, no parity strategy.

### Combined plan (what to take, what to drop)

**Take:**
- from **Opus**: P4 (one `ArrayCodec` + optional fast-path leaf) as the canonical answer to duplication; P6 (fix the precedence in `resolveByTyptype`); the idea "prototype the fast-path on int8/bool before scaling";
- from **Fable**: the split into "contract / non-contract" work; the parity harness as a gate on removing legacy; the JMH set with `-prof gc`; the migration order (switch the registry as each leaf becomes ready, no thin wrapper);
- from **GPT**: the layered registry model with explicit conflict rules (explicit OID > connection name > SPI > built-in > fallback); the fix for `unregister`/`reset` (restore the overridden built-in); pinning the caching contract in `Field`; the shape of the standalone API `RawValue`/`CodecSession`.

**Drop / defer:** the intermediate step "make `ArrayEncoding` a thin wrapper" (Fable is right — it is cheaper to switch the registry); a full decode-streaming visitor (defer it, but lay down slice signatures for it); the offline API as a feature (defer it, but do not block it — make `CodecContext` an interface now).

---

## 9. Decisions before scaling

Seven decisions to make before extending from `int4` to the rest of the types (not mixed in with ordinary implementation).

1. **The public API boundary.** Introduce `api.codec.TypeDescriptor` (read-only metadata: oid, (schema,name), typtype, typmod, elementOid, arrayOid, baseTypeOid, subtypeOid, fields, typdelim) and `CodecContext` as an **interface** with the wire part (charset, server params, lookup), without `getConnection()`/`BaseConnection`/`core.TypeInfo`. *Why it blocks:* once third-party dependencies appear, the contract cannot be reworked. *Now:* do it while it is `@Experimental`. *Defer:* the concrete makeup of `TypeDescriptor` for multirange.
2. **The decode-slice signature.** Pick one contract: `(byte[] data, int off, int len)` versus `ByteBuffer` versus a cursor/reader. *Why:* it touches every decode method of every codec, and is decided once. *Preferably now:* `(byte[], off, len)` + default delegation to the full buffer (compatible), ownership = a borrowed view for the duration of the call. *Defer:* the decode-streaming visitor (but the slice is its foundation).
3. **One array model.** Opus P4: a universal `ArrayCodec` + an optional fast-path leaf interface on a scalar codec. *Why:* otherwise a separate codec and leaf class for each of int2/int8/float4/float8/bool — exactly the multiplication we fear (C12/C13). *Preferably now:* P4. *Defer:* the fast-path for variable-width types (text/numeric — straight to generic/streaming).
4. **A text-read leaf + a shared tokeniser.** Design a cursor parser for the literal format, shared across arrays, composite, and range. *Why:* without it, "remove `ArrayDecoding`" will not happen (C10), and three separate quoting/escaping parsers are three sets of bugs. *Now:* at least pin the interface, so that `ArrayLeafCodec` can withstand it. *Defer:* moving `PgArray.getArray()` itself onto the new path.
5. **Identity and registry conflicts.** The key — OID (runtime) + (schema-optional name) for registration; layers explicit OID > connection name > SPI > built-in > fallback; fix `unregister`/`reset` (C20); invalidate `oidCache` on a name registration (C21); a deterministic choice on an SPI conflict + logging. *Why it blocks:* it affects the behaviour of every custom codec. *Defer:* predicate-based registration (for PostGIS, where the OID is per-database) — but lay down a place for it.
6. **Type-classification precedence** (C6). Check `typtype` before `typcategory` in `resolveByTyptype`. *Why:* a small fix, but without it domain-over-array and potentially composite/enum with an inherited typcategory go the wrong way. *Now:* do it, it is not a contract change, but it blocks the target scenarios.
7. **Range/multirange metadata.** Load `pg_range` (subtype) and decide about `pg_multirange`. *Why:* without the subtype, binary range is broken (C17), and multirange is in the goals but not in the code (C7). *Preferably now:* `pg_range` + `PgRangeType.subtypeOid`; multirange — metadata-only with an explicit error in v1. *Defer:* the full multirange codec.

Additionally, but less blocking: **the domain contract** (unwrap into the base — the current behaviour — or keep identity for names in `Struct`/`SQLData`; whether to pass through typmod, C22) and **the boxing policy** (`Array.getArray()` stays boxed per JDBC, primitive — only via `getObject(int[].class)`).

---

## 10. Recommended next step

For the next iteration, in execution order:

1. **Pin the perimeter and the precedence (cheap, unblocks the rest).** Introduce `api.codec.TypeDescriptor` + a `CodecContext` interface (decision 1); fix the order in `resolveByTyptype` (decision 6) and immediately cover with a test domain-over-array, domain-over-scalar, composite, enum.
2. **Pin the decode-slice signature** with default delegation to the full buffer (decision 2) and move the containers `GenericArrayLeafCodec`/`CompositeCodec`/`RangeCodec` onto it — this removes the per-element `copyOfRange` (C8) before there are 30 codecs.
3. **Prototype P4 on `int8` or `bool`** (decision 3): show that the fast-path as an optional interface on a scalar codec removes the duplication. If the prototype is clean — that is the template for the rest of the fixed-width types.
4. **Stand up the parity harness** (Fable's plan) as a gate: random value → server roundtrip → `text-decode == binary-decode == original`, a matrix of null/empty/ragged/lower-bounds/0-dim. Without it, every migrated type will silently change one of the behaviours (C33).
5. In parallel — fix the pointed confirmed bugs that do not change the contract: `unregister`/`reset` built-in (C20), single-arg `getBinaryCodec` in `RangeCodec` (C17/C18), `encodeText` for a third-party `Array` via `getArray()` (C24), caching `CodecContext` on the connection (C28).

Decisions 4 (the text-read leaf) and 7 (`pg_range`) are the next iteration, but the text-leaf interface is better sketched now, so that `ArrayLeafCodec` can withstand it.

If we agree on decisions 1–3 — it then makes sense to descend to code-level findings on the specific files (`RangeCodec` binary, the `CompositeCodec` text parser, `MultiDimArraySupport.computeDimensions`, the registry invariants).
