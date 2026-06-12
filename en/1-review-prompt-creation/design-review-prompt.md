# Prompt: design review of the Codec API for arrays, structs, and user-defined types in pgjdbc

I am building support for `struct` / `array` / `sqldata` / `sqlinput` / `sqloutput` / `Struct` / `Array` types in pgjdbc.

Previously pgjdbc supported a limited set of arrays and had weak support for custom types:

* there was no full binary protocol support for custom types;
* user-defined composite types were poorly supported;
* there was no proper support for `array-of-struct`, `struct-of-array`, and similar nested types;
* the encoding/decoding code was scattered across different classes, so the same conversions were duplicated.

The focus now is on a public Codec API: it should become the single model for encoding and decoding PostgreSQL values. pgjdbc should use this API inside JDBC operations, and third-party libraries should be able to extend it with their own codecs.

## Main goals

We need to get close to full and efficient support for PostgreSQL types:

* built-in scalar types;
* arrays;
* composite/user-defined struct types;
* domains;
* enums;
* ranges;
* multiranges;
* arbitrary nested combinations of these types.

Important scenarios:

* `array-of-struct`;
* `struct-of-array`;
* `array-of-struct-of-array`;
* `array-over-domain`;
* `domain-over-array`;
* `range-of-custom-type`;
* `multirange`;
* a custom codec for an external library, for example a PostGIS codec for geometry/geography types.

JDBC adapters (`Array`, `Struct`, `SQLInput`, `SQLOutput`, `ResultSet.getObject`, `CallableStatement`) should be thin adapters over the Codec API, not separate parsing and encoding implementations.

## Public Codec API / SPI

The Codec API is intended as a public API, not an internal implementation detail.

We need to make it possible for a third-party library to:

* implement a codec for its own PostgreSQL types;
* add the codec to the classpath;
* register the codec through `ServiceLoader` or explicit registration;
* decode and encode values through stable public abstractions;
* not depend on internal pgjdbc classes such as `PGStream`, `QueryExecutor`, `Field`, `Tuple`, `ArrayEncoding`, `ArrayDecoding`.

Check whether the design separates the following well enough:

* stable public SPI;
* connection-bound runtime context;
* type metadata / registry;
* internal protocol implementation;
* JDBC API adapters.

Assess separately:

* how third-party codecs should be registered;
* how to resolve conflicts between several codecs for the same type;
* whether priorities, override rules, and fallback are needed;
* which interfaces/classes can be stabilised as public API;
* which details should remain internal.

## Standalone encode/decode API

The design needs to provide a public standalone API for encoding and decoding values into the PostgreSQL wire representation and back.

The API should let external code:

* obtain the raw value in binary or text format without immediate decoding;
* decode a raw binary/text value into a Java object through a registered codec;
* encode a Java object, for example `CustomDto` or `CustomDto[]`, into a binary/text payload suitable for sending to PostgreSQL;
* do this through stable public abstractions.

Distinguish two modes:

1. Connection-bound encode/decode.
   The API works alongside `PGConnection` and uses the catalog metadata, server settings, codec registry, and type cache of a specific connection.

2. Offline encode/decode.
   The API works without a live connection, but the calling code must explicitly pass the `TypeDescriptor` / `CodecRegistry` / server encoding context / required metadata.

For the first version, the connection-bound API can be treated as mandatory. The offline API can be deferred, but the architecture must not block it.

Check:

* which type metadata are needed for standalone encode/decode: OID, type name, schema, typmod, array OID, element OID, composite attributes, range subtype, domain base type;
* how the API obtains metadata: from a live `Connection`, from the `TypeRegistry`, from an explicitly passed `TypeDescriptor`;
* how to represent binary/text format: `byte[]`, `ByteBuffer`, `InputStream`, reader/writer API;
* who owns the buffer: borrowed view, immutable copy, reusable scratch buffer;
* whether unnecessary copies on the hot path can be avoided;
* how a codec reports its capabilities: binary read, binary write, text read, text write;
* how the format is chosen when a codec supports only some of the operations;
* which errors the API returns for missing metadata, unsupported format, unsupported Java class;
* how to avoid pulling current protocol implementation details into the public API.

## Current implementation and review focus

I do not like the old `ArrayEncoding` / `ArrayDecoding` classes: they seem awkward for supporting `array-of-struct-of-array` and other nested cases.

As a replacement I have started building:

* `GenericArrayLeafCodec`;
* `MultiDimArrayBinary`;
* `MultiDimArrayText`;
* `MultiDimArraySupport`;
* `Int4ArrayLeafCodec`;
* the related multi-dimensional array support plumbing.

Right now I am trying out the design on `int4` / `Int4ArrayLeafCodec` before extending it to `int8`, `int2`, `float4`, `float8`, `bool`, `text`, `uuid`, temporal types, and other types.

Analyse whether the resulting design of `Int4ArrayLeafCodec` and the multi-dimensional array plumbing is suitable for scaling to the rest of the types.

Do not limit yourself to the question "does `int4[]` work or not". The interest is whether this architecture can support the whole system:

* scalar codecs;
* array codecs;
* composite codecs;
* domain codecs;
* enum codecs;
* range codecs;
* multirange codecs;
* a public codec registry;
* JDBC adapters.

## Known external feedback

Lukas Eder has already looked at a preliminary version. He found these shortcomings:

* lack of `Array` support;
* lack of `CallableStatement` support;
* no support for qualified or quoted identifiers in `Map<String, Class<?>>` arguments, for example `ResultSet.getObject(int, Map)`.

Qualification support for type names is needed. Quoted identifiers may be optional, but it is better to assess the cost and consequences.

## What exactly needs checking

Do a design review of the public Codec API, not a line-by-line code review.

Check:

* whether the public SPI and the internal implementation are separated well enough;
* whether the design of `Int4ArrayLeafCodec` / `MultiDimArray*` scales to other scalar and container types;
* whether container codecs can be built compositionally over element / field / bound codecs;
* where the design creates unnecessary boxing, allocation, or copying;
* which JDBC API cases will remain unsupported;
* which PostgreSQL correctness edge cases are not covered;
* how codec lookup, the type metadata cache, and OID/schema/qualified-name resolution should work;
* how to handle conflicting third-party codecs;
* what should change before extending the implementation to the rest of the types.

Do not descend into a fine-grained code review until it is clear that the architecture is viable. If the design is unacceptable, propose architectural changes first.

## Architectural topics for analysis

### Type identity and metadata

Check:

* how a PostgreSQL type is identified: OID, schema-qualified name, quoted name, array OID, element OID;
* how identical type names in different schemas work;
* how `search_path` is taken into account;
* how quoted identifiers are handled;
* how domains, enums, ranges, multiranges, composites, and arrays are represented;
* whether domain identity is preserved or a domain is transparently unwrapped into its base type;
* how domains over arrays and arrays over domains are handled;
* how the type cache survives `CREATE TYPE`, `DROP TYPE`, `ALTER TYPE`, and a change of `search_path`.

### Codec registry and lookup

Check:

* lookup by OID;
* lookup by qualified type name;
* lookup by Java class;
* lookup by element/field/bound type;
* the fallback chain;
* priority of user codecs over built-in codecs;
* behaviour on an ambiguous mapping;
* behaviour on an unsupported type or unsupported Java class;
* thread-safety of the registry and codec instances;
* connection-specific state vs shared immutable codec state.

### Scalar codecs

Check whether a scalar codec can be used as the single source of truth for:

* JDBC scalar read/write;
* array element read/write;
* composite field read/write;
* range bound read/write;
* standalone encode/decode.

For example:

* `int4` binary/text encoding should not be duplicated in array/composite/range code;
* `timestamptz` bytes-to-`OffsetDateTime` should live in one place;
* UUID/text/numeric/date-time codecs should be reused by all container codecs.

### Array codecs

Check:

* null array vs empty array vs array with null elements;
* multidimensional arrays;
* lower bounds;
* non-1 lower bounds;
* rectangular vs ragged arrays;
* the binary array header;
* text array parsing/escaping/quoting;
* `typdelim`;
* primitive arrays: `int[]`, `long[]`, `short[]`, `boolean[]`, `float[]`, `double[]`;
* boxed arrays: `Integer[]`, `Long[]`, etc.;
* object arrays: `Object[]`, custom DTO arrays, arrays of composite objects;
* slices: `Array.getArray(long index, int count)`;
* `Array.getResultSet`;
* `Array.getArray(Map<String, Class<?>>)`;
* `createArrayOf`;
* `setArray`;
* `setObject` with arrays;
* binary/text parity.

It is especially important to assess where the JDBC API forces boxing and where pgjdbc can keep the primitive fast path.

### Composite / Struct codecs

Check:

* the binary composite format;
* text composite parsing/escaping/quoting;
* null fields;
* dropped attributes;
* attribute order;
* schema-qualified type names;
* quoted type/field names;
* `Struct`;
* `SQLData`;
* `SQLInput`;
* `SQLOutput`;
* `ResultSet.getObject(int, Map<String, Class<?>>)`;
* `Connection.createStruct`;
* `CallableStatement` OUT/INOUT parameters;
* arrays of composite values;
* composites containing arrays, domains, enums, ranges, multiranges.

### Domains, enums, ranges, multiranges

Check:

* domain identity vs base type codec reuse;
* domain constraints and whether client-side code should know about them;
* enum labels with unusual characters;
* enum binary/text behavior;
* the range empty value;
* infinite bounds;
* inclusive/exclusive bounds;
* canonicalization assumptions;
* range subtype codec reuse;
* a custom range subtype;
* the multirange binary/text format;
* arrays of ranges and ranges inside composites.

### JDBC API coverage

Check that the design does not leave out:

* `ResultSet.getObject(int)`;
* `ResultSet.getObject(int, Class<T>)`;
* `ResultSet.getObject(int, Map<String, Class<?>>)`;
* `PreparedStatement.setObject`;
* `PreparedStatement.setArray`;
* `CallableStatement` IN/OUT/INOUT;
* `Connection.createArrayOf`;
* `Connection.createStruct`;
* `Array.getArray`;
* `Array.getResultSet`;
* `Struct.getAttributes`;
* `SQLData`;
* `SQLInput`;
* `SQLOutput`;
* `PGobject`;
* `Connection.addDataType`;
* simple query mode vs extended query mode;
* binary transfer enable/disable settings.

If the JDBC API does not forbid a call, it is preferable for pgjdbc to support it or to raise a clear error explaining the limitation.

### Performance

The following matter especially:

* low latency;
* high throughput;
* low CPU;
* a low number of allocations;
* minimal boxing for primitive arrays;
* no unnecessary intermediate `Object[]`;
* no unnecessary `String`/`byte[]` copies;
* the ability to do streaming/visitor-style traversal for large arrays/composites;
* reusing buffers where it is safe;
* a clear ownership model for borrowed/copy buffers.

Check:

* where the current design is inevitably boxing-heavy;
* where primitive specialisation can be added without duplicating logic;
* how to avoid multiplying code for `int2`, `int4`, `int8`, `float4`, `float8`, `bool`;
* how to measure regressions;
* which JMH benchmarks or integration benchmarks are worth adding;
* which performance counters/alloc profiles are worth checking.

### Error handling and usability

Check:

* the clarity of errors for an unsupported type;
* the clarity of errors for an ambiguous type name;
* the clarity of errors for a missing codec;
* the clarity of errors for a malformed array/composite/range literal;
* what the user sees for an unsupported binary/text format;
* how to explain which codec was chosen and why;
* whether a debug/tracing capability for codec lookup is needed.

## Compatibility and migration

My ultimate goal is to get rid of `ArrayEncoding` / `ArrayDecoding` entirely.

The order of removal does not matter: the old code can stay at first, the remaining types can be migrated gradually, and the old classes can then be removed.

Check:

* which observable behaviors of the old `ArrayEncoding` / `ArrayDecoding` need to be preserved;
* which incompatibilities are acceptable;
* which tests should exist before removing the old path;
* which old extension points (`PGobject`, `Connection.addDataType`) should keep working;
* whether the old classes can be made thin wrappers over the new Codec API for the transition period.

## Tests and experiments

If you need to check pgjdbc or PostgreSQL behaviour, you can run tests and a PostgreSQL server.

You can bring up a PostgreSQL server through Docker:

```bash
docker compose -f docker/postgres-server/docker-compose.yml up -d
```

If you run Gradle, use `--quiet` so the output does not bloat.

Examples of what can be checked experimentally:

* binary/text parity for arrays/composites;
* the behavior of `ResultSet.getObject(int, Map<String, Class<?>>)`;
* the behavior of qualified/quoted identifiers;
* `CallableStatement` with custom/composite/array OUT parameters;
* `createArrayOf` and `createStruct`;
* arrays with lower bounds;
* arrays containing nulls;
* multidimensional arrays;
* domains over arrays;
* arrays over domains;
* enum/range/multirange behavior;
* the performance/allocation profile for `int[]` vs `Integer[]`.

## Expected result

First give an architectural verdict:

* the design is viable;
* the design is viable, but needs changes before scaling;
* the design should not be scaled without substantial rework.

Then give:

* the main architectural risks;
* missing abstractions;
* places where the public API leaks internal implementation;
* performance risks;
* correctness risks;
* JDBC compatibility gaps;
* concrete proposals for changing the architecture;
* a list of questions that need to be resolved before extending to the rest of the types.

If the architecture looks acceptable, you can then move on to code findings for specific files.
