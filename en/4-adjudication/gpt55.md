**Brief verdict**

A = [gpt55.md](../3-comparison/gpt55.md), B = [opus48.md](../3-comparison/opus48.md).

A is more useful as a comparison: claims are atomised better, the `confirmed` / `trade-off` / `false` statuses are separated more cleanly, and the matrix is more complete. B is more useful as a plan for the next architectural iteration: it states the “single `ArrayCodec` + optional fast-path” model more strongly and sharpens `domain-over-array` more precisely.

The final verdict is reliable for claims checked against the code. For PostgreSQL catalog edge cases and performance claims the reliability is moderate: I did not run server experiments, Gradle, or JMH.

**Divergence matrix**

| Topic / claim | A | B | Agree | Needs re-checking | Final verdict |
| --- | --- | --- | --- | --- | --- |
| Public SPI pulls in `jdbc.PgType` / `CodecContext` | blocker confirmed | blocker confirmed | yes | no | confirmed |
| Offline API blocked by `CodecContext` | confirmed | confirmed | yes | no | confirmed |
| Registry by simple name without schema | confirmed + trade-off | confirmed, stronger on `search_path` | partly | no | confirmed risk + trade-off |
| No explicit registry layers / restore on unregister | confirmed | confirmed | yes | no | confirmed |
| `registerByName` / alias do not invalidate the OID cache | confirmed | confirmed as Fable-only | yes | no | confirmed |
| Decode without slice makes copies | confirmed | confirmed | yes | no | confirmed |
| Array read path still legacy | confirmed | confirmed | yes | no | confirmed |
| No text-read leaf for arrays | confirmed | confirmed | yes | no | confirmed |
| Unified `ArrayCodec` + fast-path SPI | trade-off / recommendation | key recommendation | partly | prototype | trade-off, recommended |
| Lower bounds are lost | confirmed + trade-off | confirmed + trade-off | yes | policy | trade-off |
| `PgArray.getArray(Map)` binary / `getResultSet(Map)` | confirmed | confirmed | yes | no | confirmed |
| Foreign `Array` text encode via `toString()` | confirmed | confirmed | yes | no | confirmed |
| `setObject(dto)` does not use Java-class codec lookup | confirmed | B left it less verified | partly | no | confirmed |
| Named `CallableStatement` overloads | confirmed | part left unverified | partly | no | confirmed |
| `SQLInput.readArray()` missing | confirmed | more in the coverage plan | partly | no | confirmed |
| `Field` caches a codec | confirmed | partially confirmed | partly | no | confirmed |
| `getString()` text path bypasses the codec | confirmed | unverified | no | no | partially confirmed: text path yes, binary path uses the codec |
| Range subtype via `typelem=0`, no multirange | confirmed | confirmed | yes | server tests | confirmed |
| `domain-over-array` dispatch | partially confirmed | confirmed | partly | server repro | confirmed design risk |
| Dropped composite attributes are not filtered | false | not reproducible | yes on balance | no | false, not a hallucination |
| “45/45 tests” as proof of maturity | unclear | unclear | yes | Gradle/tests | unresolved |
| New missed topics | A: trailing bytes, element OID header | B: simple query mode, enum labels | no | targeted review | useful, but secondary |

**Final confirmed issues**

- `blocker`: the public SPI leaks internal API. [BinaryCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/BinaryCodec.java#L9) imports `CodecContext` and `PgType`; [CodecContext.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java#L41) is `final` and hands out `BaseConnection` / `TypeInfo`.
- `high`: registry identity and override semantics are not ready for a public API. A single `codecsByName`, simple name lookup, and a destructive unregister are visible in [CodecRegistry.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L280).
- `high`: the decode API without slice/view already creates copies in the array/range/composite paths, for example [GenericArrayLeafCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java#L116) and [RangeCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L120).
- `high`: the array migration is not finished. `ArrayCodec.decodeBinary()` returns a `PgArray`, while typed decode goes through `ArrayDecoding`.
- `high`: JDBC gaps remain: binary `PgArray.getArray(Map)` and `getResultSet(Map)` throw `notImplemented`; `SQLInput.readArray()` too.
- `high`: range metadata is not loaded from `pg_range`; binary range takes the subtype from `typelem`, which for a range is not the subtype.
- `medium/high`: `resolveByTyptype()` checks `isArray()` before `isDomain()`. For a domain that inherits the array category, this routes dispatch into `ArrayCodec`.

**Final false / hallucinated claims**

- “Dropped attributes in composites are not filtered”: false. `TypeInfoCache.loadCompositeFields()` adds `AND NOT a.attisdropped`.
- “`getArray(Map)` / `getResultSet(Map)` are already covered”: false for a non-empty map in the current code.
- “Backpatching is completely unavailable to third-party codecs”: overstated. The public boundary problem is real, but it is not an absolute ban on streaming encode.

**Unresolved questions**

- A server repro is needed for `domain-over-array`: check the real `typtype`, `typcategory`, `typelem`, `typbasetype`, and the encode/decode path.
- A parity harness is needed for binary/text arrays, composites, ranges, domains, and enum labels.
- JMH/alloc profiles are needed: claims about round-trip, boxing, and per-element context allocation remain risk-based without numbers.
- A policy decision is needed, not a “bug fix”: lower bounds, domain identity, typmod propagation, offline API scope.

**Design trade-offs**

- Custom codec registration: OID is precise within a connection, a schema-qualified name is more portable, an unqualified name is convenient but depends on `search_path`. The practical approach is to support all forms with an explicit priority.
- Domain handling: base-codec reuse is simple and compatible; preserving domain identity is needed for metadata/JDBC fidelity. Recommendation: the descriptor preserves the domain, the value codec delegates to the base type.
- Primitive arrays: JDBC `Array.getArray()` often leads to a boxed/object path; a fast path is needed for `getObject(int[].class)` and internal containers.
- Binary-only/text-only codecs: an explicit capabilities contract is needed instead of `instanceof` and exceptions from within.

**Problems missed by both comparisons**

I am not adding new ones as final findings: that would already be a new architecture review. For the current verdict the missed topics raised by A or B are enough: simple query mode, trailing bytes, binary array header element OID, buffer ownership, `PGobject` / `addDataType` compatibility.

**Decisions before scaling**

1. A public `TypeDescriptor` and a public `CodecContext` interface without `BaseConnection`.
2. Decode slice / `RawValue` / ownership model.
3. Registry layers and conflict rules: user OID, qualified name, unqualified alias, SPI, built-in, fallback.
4. A single `ArrayCodec` with an optional scalar fast-path, without proliferating leaf classes.
5. A shared text tokenizer for arrays/composites/ranges.
6. Type dispatch precedence: `typtype` before `typcategory`.
7. A `pg_range` / multirange metadata model.
8. JDBC coverage floor: `Array(Map)`, `SQLInput.readArray`, named Callable, `setObject` Java-class lookup.
9. Explicit binary/text read/write capabilities.
10. Migration gate: parity tests plus JMH before removing `ArrayEncoding` / `ArrayDecoding`.

**Recommended next iteration**

First, pin down the API sketch for `TypeDescriptor`, codec context, decode slice, and capabilities. Then fix dispatch precedence and registry restore/cache semantics. After that, prototype the unified array model on `int8` or `bool`, add parity tests for `domain-over-array`, binary/text arrays, `Array.getArray(Map)`, and range subtype.
