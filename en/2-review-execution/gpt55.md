**Verdict**

The design is viable, but scaling it to the remaining types is premature. The direction is right: the scalar codec becomes the source of truth, `MultiDimArray*` separates the array's shape from leaf encoding, and the composite codec is already built on top of field codecs. But before extending it to `int8`, `text`, `uuid`, temporal, ranges, and third-party codecs, you need to fix the public API boundaries, registry/type identity, and the decode hot path.

**Main risks**

1. The public API still drags in the internal/JDBC layer.
   `BinaryCodec` accepts `PgType` and `CodecContext` from `org.postgresql.jdbc` [BinaryCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/api/codec/BinaryCodec.java#L60), and `CodecContext` hands out `BaseConnection` and `TypeInfo` [CodecContext.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecContext.java#L275). For a stable SPI it is better to introduce a public `TypeDescriptor`, `CodecSession`/`EncodingContext` and not expose `BaseConnection`, `TypeInfoCache`, or `TimestampUtils`.

2. Type identity is insufficient.
   The registry stores codecs by plain `String` name [CodecRegistry.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L83), and the OID lookup takes only `pgType.getTypeName().getName()` [CodecRegistry.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L410). This breaks scenarios with the same `typname` in different schemas. You need a key: OID for the connection-bound runtime, plus a schema-qualified name for registration and diagnostics.

3. The registry overlay is mixed in with the built-ins.
   `registerCustomCodec` overwrites the entry in the same map, and `unregisterCustomCodec` removes the name without restoring the built-in/SPI codec [CodecRegistry.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L350). You need separate layers: user override, explicit OID, SPI, built-in, fallback. SPI conflicts are currently last-wins, in effect, and silent.

4. Range/multirange metadata is not ready yet.
   `RangeCodec` tries to take the subtype from `type.getTypelem()` [RangeCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L99), but the code itself admits that the subtype lives in `pg_range` and is not loaded yet [RangeCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L275). Without `pg_range`/`pg_multirange` descriptors you cannot close `range-of-custom-type` and multirange.

5. Arrays: the new architecture is good for encode, but the read path is still the old one.
   `MultiDimArrayBinary` writes the header and delegates the leaf loop, which is a good shape [MultiDimArrayBinary.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java#L114). But the lower bound is always `1` [MultiDimArrayBinary.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/MultiDimArrayBinary.java#L132), and `PgArray` still does not support a map for binary arrays [PgArray.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgArray.java#L210). For a full replacement of `ArrayDecoding` you need a value model with dimensions, lower bounds, an element descriptor, and type-map aware binary decode.

6. Copy-free decode is not laid down yet.
   The generic array decode does an `Arrays.copyOfRange` per element [GenericArrayLeafCodec.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java#L116). Composite/range do similar copies. Streaming encode exists, but you also need a `RawBinaryValue`/slice API for decode: `byte[] + offset + length` or a `ByteBuffer` with an explicit ownership model.

**Assessment of `Int4ArrayLeafCodec`**

As a POC it is a good one: the primitive `int[]` is not boxing-heavy, `Integer[]` carries nulls correctly, and the multi-dimensional shape is moved out into shared code. It can be scaled, but not by copy-pasting per type. It is better to make an internal template for fixed-width primitive leaves: `int2/int4/int8/float4/float8/bool/date/time/timestamp/uuid`, and route the variable-width types (`text`, `numeric`, `bytea`, composite, range) through the generic/streaming path.

**What to change before extending**

- Introduce a public `TypeDescriptor` with a kind: scalar, array, domain, enum, composite, range, multirange; include OID, qualified name, typmod, array OID, element/base/subtype metadata, composite attrs, `typdelim`.
- Replace `Codec.getTypeName()` as the primary SPI with `CodecProvider`/`CodecFactory`: names, Java classes, capabilities, priority, replace policy. At the connection-bind stage the provider resolves into OID bindings.
- Split the registry layers and explicitly describe the conflict rules: user explicit OID > user qualified name > SPI > built-in > fallback. A duplicate SPI must give a clear error or deterministic priority.
- Make a raw standalone API: `RawValue(format, type, bytes/view, ownership)`, `CodecSession`, `encode(value, type, preferredFormat)`, `decode(raw, targetClass)`.
- Move the `PgArray` read path onto the new array codec, including binary `getArray(map)`, `getResultSet(map)`, slices, and lower bounds.
- Add `pg_range`/`pg_multirange` loading before working on range/multirange codecs.
- Pin down the contract: codecs are cached in `Field`, so `registerCodec()` affects future queries/result sets, not fields that are already initialised [Field.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/core/Field.java#L196), [PgResultSet.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgResultSet.java#L3326).
- Close the JDBC gaps: a foreign `Array` in `setArray` currently depends on `toString()` [PgPreparedStatement.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgPreparedStatement.java#L1063); `CallableStatement.registerOutParameter(String, ...)` is still not implemented [PgCallableStatement.java](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgCallableStatement.java#L620).

I did not run the tests: this was an architectural pass over the current code and the repro tests.
