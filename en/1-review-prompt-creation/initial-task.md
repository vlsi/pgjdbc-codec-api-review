# Original task statement

This is the original statement that started the work on the prompt for the architecture review of the Codec API. At this stage the task already carried engineering context, but it had not yet fixed the type of review, the boundaries of the public API, and the full scope of PostgreSQL types.

```text
I am developing support for struct/array/sqldata/sqlinput/sqloutput/Struct/Array types in pgjdbc.
Previously only a limited number of arrays were supported, and custom types were supported very poorly.
There was no binary-protocol support for custom types, no support for array-of-struct, and so on.

Main goals:

* support for arbitrary formats and arbitrary structures
* support for binary and text write formats for all data types, including user-defined structs
* efficient handling of primitives (for example, int[] should avoid allocating Integer where possible)
* avoiding code duplication. For example, encoding byte[] (timestamptz) -> OffsetDateTime should happen in a single class and be reused, rather than duplicated across different parts of the code. Likewise, encoding and decoding int4 should exist once and be reused everywhere else
* I want to support the most unusual user requests possible. If the JDBC API does not forbid some kind of call, then pgjdbc should support it

Right now the focus is on struct, array, user-defined types -- the Codec API.

I have already shown a preliminary version to users, and Lukas Eder found these shortcomings:

* Lack of Array support
* Lack of CallableStatement support
* No support for qualified or quoted identifiers in Map<String, Class<?>> arguments (e.g. ResultSet.getObject(int, Map)). Qualification support is necessary, quoted identifiers probably optional?

I analysed them and started working on array support.
I dislike the old ArrayEncoding / ArrayDecoding classes (they seem awkward for supporting array-of-struct-of-array), and as a replacement I designed GenericArrayLeafCodec, MultiDimArrayBinary, MultiDimArrayText, MultiDimArraySupport, Int4ArrayLeafCodec

Analyse whether the resulting design of Int4ArrayLeafCodec and the rest of the multi-dim-array wrapping is sound, and whether it can be scaled to other array types (int8, int2, ...) -- I wanted to try it on one first and expand only afterwards.

What other architectural changes are worth making to get closer to the goal of "full and efficient (low latency, high throughput, low cpu) support for arrays, structs, custom types".

If you want to run some tests (to understand, for example, whether a particular API usage is supported or not) you can start a PostgreSQL server via docker (see docker/postgres-server/docker-compose.yml)
```
