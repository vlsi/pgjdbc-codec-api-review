**1. Brief verdict**

The most useful answer: `fable5.md`. It covers more of the original rubric: public SPI, registry, arrays, text/binary parity, JDBC gaps, migration, performance, and tests. Most of its high/blocker claims are confirmed by the code.

The best-evidenced answer: also Fable, narrowly ahead of Opus. Fable has more verifiable references to specific classes. Opus is more useful at two points: `domain-over-array` because of the ordering in `resolveByTyptype()`, and the idea of collapsing the array design down to a single `ArrayCodec` plus a fast-path SPI.

GPT 5.5 is the most concise and, on the whole, an accurate executive summary, but it skips many edge cases and barely proves any claims.

The models mostly agree on five problems: the public SPI leaks internal types; the decode path has no slice/view API; array migration still depends on `ArrayEncoding`/`ArrayDecoding`; the registry and type identity are not layered enough; range/multirange metadata is not ready.

They diverge mainly in how they assess the maturity of the array path: Fable is more cautious and more precise about the legacy-read path, Opus argues harder for a canonical model, GPT stays at the level of correct but broad recommendations.

**2. Methodology**

The following were compared:

- [design-review-prompt.md](../1-review-prompt-creation/design-review-prompt.md)
- [fable5.md](../2-review-execution/fable5.md)
- [opus48.md](../2-review-execution/opus48.md)
- [gpt55.md](../2-review-execution/gpt55.md)

The current working-tree files were checked: codec SPI, `CodecContext`, `CodecRegistry`, `TypeInfoCache`, `PgType`, `IdentifierNormalizingTypeMap`, `ArrayCodec`, `ArrayLeafStreamingCodec`, `MultiDimArray*`, `PgArray`, `ArrayDecoding`, `CompositeCodec`, `PgStruct`, `PgSQLInput*`, `PgSQLOutput*`, `RangeCodec`, `DomainCodec`, `EnumCodec`, `PgResultSet`, `Field`, `PgPreparedStatement`, `PgCallableStatement`.

I used `fff` for searching and `ast-grep` for structural checks of the key methods, in particular `CodecRegistry.resolveByTyptype()` and `PgArray.getArrayImpl(...)`.

I did not run Gradle, PostgreSQL, or JMH. For that reason, the claims about "45/45 tests", the actual binary/text parity on the server, and the performance numbers are marked as unverified.

**3. Claims matrix**

| ID | Claim | Category / severity | Fable | Opus | GPT | Code check | Verdict |
| --- | --- | --- | --- | --- | --- | --- | --- |
| C01 | Public SPI depends on `org.postgresql.jdbc.*` / `core.*` | public SPI / blocker | yes | yes | yes | `BinaryCodec` imports `CodecContext`/`PgType`; `CodecContext` exposes `BaseConnection`, `TypeInfo`, `Encoding` | confirmed |
| C02 | Offline/standalone API architecturally blocked by the current `CodecContext` | standalone / high | yes | yes | partially | connectionless constructor package-private, registries null, `withTypeMap()` throws | confirmed |
| C03 | Registry keyed by simple type name, schema collisions possible | type identity / high | yes | yes | yes | `Codec.getTypeName()` without schema; `codecsByName: Map<String, Codec>` | confirmed + trade-off |
| C04 | Registry layers/override rules insufficient | registry / high | yes | yes | yes | SPI, built-in, and custom all land in the same maps; duplicate SPI last-wins | confirmed |
| C05 | `registerByName`/`registerAlias` do not invalidate the OID cache | registry / medium | yes | no | no | the methods write to the map without `oidCache.invalidateAll()` | confirmed |
| C06 | `unregisterCustomCodec` does not restore the built-in/SPI codec | registry / medium | no | no | yes | removes the name from `codecsByName`; there is no separate layer | confirmed |
| C07 | SPI default Java class can override the built-in class mapping | registry / high | no | yes | no | `registerByClass(codec.getDefaultJavaType(), codec)` after built-ins | confirmed |
| C08 | No explicit codec capabilities | registry / high | yes | yes | yes | the format is inferred via `instanceof BinaryCodec/TextCodec` and `binaryTransferSend()` | confirmed |
| C09 | Decode API without slice causes copies | performance / blocker | yes | yes | yes | `GenericArrayLeafCodec.copyOfRange`, `CompositeCodec.new byte[]`, `RangeCodec.copyOfRange` | confirmed |
| C10 | Streaming encode/backpatch is a strong part of the design | performance / medium | yes | yes | no | `BackpatchByteArrayOutputStream`, `reserveInt32`, container streaming paths | confirmed |
| C11 | The backpatch capability is not exposed as public API | public SPI / medium | no | yes | no | `BackpatchingBinarySink` package-private; the public API references the jdbc codec class | partially confirmed |
| C12 | Leaf/walker decomposition for arrays is viable | array / medium | yes | yes | yes | `MultiDimArrayBinary/Text` separate the shape from the leaf loop | design trade-off |
| C13 | The new array read path still coexists with legacy | migration / high | yes | yes | yes | `decodeBinary()` returns `PgArray`; `PgArray` reads through `ArrayDecoding` | confirmed |
| C14 | The new array architecture has no text read leaf | array / high | yes | yes | no | `ArrayLeafCodec` has binary read/write and text write; `MultiDimArrayText` is encode-only | confirmed |
| C15 | `Int4ArrayLeafCodec` duplicates scalar int4 logic | maintainability / medium | no | yes | no | the leaf reads/writes int4 itself and calls `Int4Codec.toInt`, but not the shared scalar encode/decode | confirmed |
| C16 | There is no shared primitive encode fast path in the public SPI | performance / medium | partially | yes | no | decode specialisations exist; encode specialisations do not, apart from the private leaf POC | confirmed |
| C17 | `computeDimensions()` breaks runtime-nested `Object[]{Object[]{...}}` | array / medium | yes | no | no | the dimension is computed from the class, not from the elements | confirmed |
| C18 | Lower bounds are lost / encode always writes `1` | array / medium | yes | yes | yes | `MultiDimArrayBinary` writes `1`; decode reads and ignores the lower bound | confirmed + trade-off |
| C19 | `PgArray.getArray(Map)` does not work for binary arrays | JDBC / high | yes | disputed | yes | binary branch throws `notImplemented` with map | confirmed |
| C20 | `PgArray.getResultSet(Map)` not implemented | JDBC / high | yes | disputed | no | `getResultSetImpl(..., Map)` throws `notImplemented` | confirmed |
| C21 | `setArray` for a foreign `Array` uses `toString()` | JDBC / high | yes | no | yes | `PgPreparedStatement.setArray` comments admit the limitation | confirmed |
| C22 | `SQLInput.readArray()` is missing | JDBC / high | yes | no | no | `PgSQLInput.readArray()` throws not implemented | confirmed |
| C23 | `SQLOutput.writeArray()` exists | JDBC / low | no | no | no | `PgSQLOutput.writeArray()` delegates to codec | confirmed, useful correction |
| C24 | Composite binary decode copies field bytes | performance / medium | yes | no | no | `CompositeCodec.decodeBinaryFields()` allocates per field | confirmed |
| C25 | Composite text decode tolerates field-count mismatch silently | correctness / high | yes | no | no | uses `Math.min(rawFields.length, expected)` | confirmed |
| C26 | Dropped attributes not filtered | composite / high | no | yes | no | `TypeInfoCache.loadCompositeFields()` has `NOT a.attisdropped` | false / hallucinated |
| C27 | Range subtype metadata missing; binary range broken | ranges / blocker | yes | partially | yes | `RangeCodec` uses `type.getTypelem()`; text path comments `pg_range` not loaded | confirmed |
| C28 | Multirange unsupported | multirange / high | yes | yes | yes | `resolveByTyptype()` has no `typtype == 'm'` branch | confirmed |
| C29 | Domain identity/typmod is lost through base codec delegation | domains / medium | yes | partially | partially | `DomainCodec` passes base `PgType` to base codec | confirmed |
| C30 | `domain-over-array` likely resolves as array before domain | domains/arrays / high | no | yes | no | code checks `isArray()` before `isDomain()`; PG catalog behavior not experimentally checked | partially confirmed |
| C31 | Enum decode does not map to Java enum through type map | enum / medium | yes | no | no | `decode*As` accepts only `String`/`Object`; encode accepts `Enum` | confirmed |
| C32 | `getString()` text path bypasses codec | scalar / medium | yes | no | no | text branch decodes connection encoding directly | confirmed |
| C33 | `CodecContext` allocated repeatedly | performance / low | yes | no | no | `PgConnection.getCodecContext()` returns new instance each call | confirmed |
| C34 | `Field` caches codec once | registry / medium | yes | no | yes | `Field.initializeCodec()` no-ops if already initialized | confirmed |
| C35 | `IdentifierNormalizingTypeMap` handles qualified/quoted `Map` keys | JDBC / high | yes | yes | no | resolves lookup and user keys through `regtype` OID | confirmed |
| C36 | Named `CallableStatement.registerOutParameter(String, …)` not implemented | JDBC / medium | no | partially | yes | string overloads throw `notImplemented` | confirmed |
| C37 | Positional Callable OUT exists and goes through `ResultSet.getObject` | JDBC / medium | yes | yes | no | `executeWithFlags()` fills `callResult[j] = rs.getObject(...)` | confirmed |
| C38 | `setObject(dto)` does not use Java-class codec lookup | JDBC/SPI / high | no | yes | no | `findCodecFor` exists, but untyped `setObject` does not call it | confirmed |
| C39 | Test count / "45/45" proves maturity | tests / low | no | yes | no | tests not run in this comparison | unclear |
| C40 | Simple-name registration is "wrong" rather than a trade-off | type identity / medium | implied | implied | implied | code confirms ambiguity, but unqualified names are useful with `search_path` | design trade-off |

**4. Confirmed shared conclusions**

All or almost all of the models correctly found that the public SPI cannot be stabilised in its current form. `org.postgresql.api.codec.BinaryCodec` and `TextCodec` accept `PgType` and `CodecContext` from `org.postgresql.jdbc`, and `CodecContext` exposes `BaseConnection`, `TypeInfo`, `CodecRegistry`, `JavaTypeRegistry`, and `Encoding`. This is a blocker for the public SPI and for the offline API.

All three models correctly point to the absence of a decode slice/view API. It already creates copies in arrays, composites, and ranges, and it will be more expensive to fix later: you would have to change the signatures of all container/scalar decode paths.

All the models agree that the array migration is not finished. The new `Int4ArrayLeafCodec` is useful, but the ordinary `PgArray.getArray()` still goes through `ArrayDecoding`, and the new model has no text read leaf.

All the models are right about range/multirange: `RangeCodec` has no subtype metadata from `pg_range`, the binary path currently takes `typelem`, and multirange has no registry dispatch.

Fable and Opus correctly treat `IdentifierNormalizingTypeMap` as a strong part of the current code: the feedback about qualified/quoted identifiers for `ResultSet.getObject(int, Map)` is largely addressed through `regtype`/OID normalization.

**5. Unique useful findings**

Fable:

- `registerByName` and `registerAlias` do not invalidate the OID cache, unlike `registerCustomCodec`.
- `ArrayCodec.encodeText(Array)` and `PgPreparedStatement.setArray(foreign Array)` depend on `Array.toString()`.
- `CompositeCodec.decodeTextAsStruct()` silently tolerates a mismatch in the number of fields.
- `getString()` for text values remains a hardcoded path outside the codec model.
- `PgConnection.getCodecContext()` creates a new context, and the legacy `ArrayDecoding` may do this per element for mapped custom types.

Opus:

- `resolveByTyptype()` checks array before domain. For `domain-over-array` this is a likely correctness bug.
- `registerByClass(codec.getDefaultJavaType(), codec)` for the SPI can override the built-in `String`/`Object`.
- `setObject(dto)` does not use `CodecRegistry.findCodecFor()`.
- The plan of "one universal `ArrayCodec` + optional fast-path on the scalar codec" scales better than a separate public-ish codec per element OID.

GPT:

- Clearly states the registry-layers problem: `unregisterCustomCodec()` removes the custom entry but does not restore the hidden built-in/SPI codec.
- Correctly singles out the contract for the `Field` codec cache: registry changes must not promise any effect on already-initialised result-set fields.
- States the future standalone shape well, via `RawValue(format, type, bytes/view, ownership)`.

**6. Doubtful or false claims**

False claim by Opus: dropped attributes in composites are supposedly not filtered. In the current code [TypeInfoCache.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/TypeInfoCache.java#L859) loads fields with `AND NOT a.attisdropped`.

Doubtful claim by Opus: `getArray(Map)`, `getResultSet`, `getArray(Map)` are "covered". Slices and the ordinary `getResultSet()` exist, but [PgArray.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgArray.java#L213) throws `notImplemented` for binary `getArray(Map)`, and [PgArray.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgArray.java#L478) throws `notImplemented` for `getResultSet(Map)`.

Overstated claim by Opus: a third-party streaming codec "always" gets the slow path because `BackpatchingBinarySink` is package-private. The problem is real as an API boundary leak, but not absolute: `BackpatchByteArrayOutputStream` is public, and third-party code can technically depend on it. It would just be a dependency on the internal `org.postgresql.jdbc.codec`.

Partially confirmed claim by Fable/Opus: simple-name codec registration is "wrong". The fact: a schema collision is possible. But this is not a pure bug, it is a trade-off: an unqualified name is convenient and follows `search_path`, an OID is precise but connection-bound, a schema-qualified name is precise for the user but requires an explicit binding. Practical recommendation: support all three forms, but make the conflict rules explicit.

Unverified claim by Opus: "45/45 tests". I did not run the tests, so this remains unconfirmed.

**7. Missed topics**

Problems missed by all the models:

- `MultiDimArrayBinary.decode()` ignores the element OID from the binary header. The code reads the OID and discards it, relying on a caller-known leaf. For an ordinary trusted ResultSet this is tolerable, but for standalone/raw decode and a malformed payload it is worth validating the mismatch.
- Binary decoders barely check trailing bytes. `CompositeCodec.decodeBinaryFields()` does not check that the `ByteBuffer` is exhausted after the declared number of fields; `RangeCodec.decodeBinary()` does not check trailing bytes after the upper bound. This falls under the original rubric item "malformed binary payload errors".
- `PgSQLInputBinary`/`PgSQLInputText` pre-cache codecs via `castNonNull(ctx.getCodecs().get*Codec(...))`. If the type dispatch returns a codec without the required format, the error will be less clear than in the container encode paths. This is another symptom of the missing capabilities.

Topics from the original task that are covered only superficially:

- explicit ownership model for raw binary/text buffers: borrowed view, copy, reusable scratch.
- exact `simple query mode` vs extended query mode policy for custom codecs.
- compatibility contract for `PGobject`/`Connection.addDataType` once the Codec SPI appears.
- typmod propagation: `PgField` stores the typmod, but the codec API actually works with `PgType`, not with a field-level typmod.
- concrete migration test matrix for removing `ArrayEncoding`/`ArrayDecoding`.

**8. Comparison of plans**

Best overall plan: Fable, because it puts the contract-changing tasks before replicating codecs: public boundary, decode slice, text array reader, scoped registry, range metadata. This does a better job of reducing the risk of cementing a poor SPI.

Best standalone architectural proposal: Opus P4 — one universal `ArrayCodec` that compositionally uses element codecs, plus an optional fast-path interface on the scalar codec. This better answers the question "how to add `int2/int8/float/bool/text/uuid/...` quickly and at scale without copy-pasting leaf codecs".

GPT's plan is useful as a short roadmap, but it lacks verification tests, order-of-work, and the details of migrating the old array path.

What to take:

- from Fable: public boundary, decode slice, text tokenizer/read leaf, range metadata, parity/JMH plan;
- from Opus: unified array model + scalar fast-path, fix `resolveByTyptype` precedence, SPI conflict handling;
- from GPT: registry layers and `RawValue`/standalone API as the public form.

What to drop or defer:

- do not try to rewrite the whole driver onto codecs at once;
- do not make a separate public array codec per element OID;
- do not fix `PgType`/`CodecContext` as a stable API;
- do not require a full multirange implementation before the metadata layer, but do not leave a silent fallback without a clear status either.

**9. Decisions before scaling**

| Decision | Why it blocks scaling | Options | Preferred now | Can defer |
| --- | --- | --- | --- | --- |
| Public SPI boundary | After third-party codecs, breaking the API will be expensive | keep `PgType/CodecContext`; introduce `TypeDescriptor`/public context | introduce public interfaces, internal adapters | a full offline runtime |
| Type identity | a simple name breaks on schema collisions | OID; unqualified name; schema-qualified; predicate | OID for connection-bound + schema-qualified registration + unqualified aliases | advanced predicates |
| Registry layers | otherwise override/unregister/cache semantics are unclear | one map; layered maps; priority numbers | explicit user OID > user qualified > SPI > built-in > fallback | dynamic rebinding existing fields |
| Decode buffer API | without it there will be copies in all container codecs | `byte[]`; `(byte[], off, len)`; `ByteBuffer`; `RawValue` | borrowed `(byte[], off, len)` plus future `RawValue` | streaming visitor decode |
| Array architecture | otherwise leaf codecs proliferate | per-OID leaf codec; universal array codec; hybrid | universal `ArrayCodec` + optional scalar fast-path | all primitive specializations |
| Text parsing | without text read you cannot remove `ArrayDecoding` | keep legacy; shared tokenizer | shared cursor/tokenizer for array/composite/range | full legacy deletion |
| Domain semantics | affects domain-over-array and metadata | unwrap always; preserve descriptor + delegate base codec | preserve domain descriptor, reuse base codec for value | client-side domain constraints |
| Range/multirange metadata | range-of-custom-type is impossible without subtype | text-only fallback; load `pg_range`; full codec | load `pg_range`/multirange metadata first | full multirange encode/decode |
| JDBC coverage boundary | public API must be predictable | support all; clear unsupported errors | fix `SQLInput.readArray`, binary `Array.getArray(Map)`, foreign `Array`, named Callable | LOB/Ref/RowId |
| Format capabilities | binary/text choice is currently implicit | `instanceof`; capability enum | explicit read/write capabilities per format | advanced negotiation policy |

**10. Recommended next step**

The immediate iteration:

1. Pin down a small API sketch: `TypeDescriptor`, public `CodecContext`, decode slice, format capabilities, registry layers.
2. Prototype it on `int4[] + composite[]`: remove the per-element copy in generic array/composite decode and keep the current `int[]` fast path.
3. Add regression tests for `domain-over-array`, binary `Array.getArray(Map)`, foreign `Array.setArray`, binary range explicit failure/metadata, and `SQLInput.readArray`.
4. After that, port the next scalar (`int8` or `bool`) not as a copy of `Int4ArrayLeafCodec`, but through the chosen fast-path interface.
