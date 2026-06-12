# The clarifications that focused the prompt

This is a condensed transcript of the process in which the initial problem statement turned into the final [`design-review-prompt.md`](design-review-prompt.md).

An important part of the process: the task was not a single-step "improve the prompt". The clarifications surfaced hidden open decisions: what kind of review was needed, whether the Codec API is a public SPI, which PostgreSQL types are in scope, and whether to provide for a standalone encode/decode API.

## First layer of clarifications

After the initial problem statement, the following questions were raised:

* Should the Codec API be a public API for users and extensions, or an internal pgjdbc detail?
* Does the scope of "all types" include domains, enums, ranges, and multiranges, or only built-in scalar types, composites, and arrays?
* What result is needed: an architectural design review, a list of code findings by file, or a roadmap for removing `ArrayEncoding` / `ArrayDecoding`?
* Should a standalone API for encoding and decoding the PostgreSQL wire representation be considered?

## Answers and their consequences

The Codec API was fixed as a public API.

This added a separate focus on the public SPI to the prompt:

* third-party libraries, for example PostGIS, can implement their own codecs;
* registration must be possible through `ServiceLoader`, explicit registration, or a similar mechanism;
* the public API must not require a dependency on internal pgjdbc classes;
* codec conflicts, priorities, fallback, and the boundaries of the stable/internal API must be thought through in advance.

The scope was expanded to all PostgreSQL types:

* built-in scalar types;
* arrays;
* composites;
* domains;
* enums;
* ranges;
* multiranges;
* nested combinations of these types.

The result format was focused on a design review:

* architectural viability first;
* then risks, missing abstractions, and design changes;
* code findings only once it is clear that the architecture is acceptable;
* the roadmap for removing `ArrayEncoding` / `ArrayDecoding` is not the main topic, because it depends on the readiness of the new codec-based implementation.

## Standalone encode/decode API

A separate clarification decided to provide for a public API for direct encoding and decoding of PostgreSQL values:

* obtain the raw binary/text value without immediate decoding;
* decode a raw binary/text payload through a registered codec;
* encode a Java object, for example `CustomDto` or `CustomDto[]`, into a binary/text representation;
* use this through stable public abstractions rather than through `PGStream`, `QueryExecutor`, `Field`, `Tuple`, `ArrayEncoding`, or `ArrayDecoding`.

After this, the prompt explicitly separated two modes:

1. Connection-bound encode/decode.
   This mode uses the metadata, registry, server settings, and type cache of a specific connection. For the first version it is considered mandatory.

2. Offline encode/decode.
   This mode works without a live connection, but requires explicitly passed type descriptors and server context. It can be deferred, but the architecture must not block it.

## What changed compared to the initial problem statement

The initial task was mostly about a specific array support implementation: `Int4ArrayLeafCodec`, `GenericArrayLeafCodec`, `MultiDimArrayBinary`, `MultiDimArrayText`, and the ability to replace `ArrayEncoding` / `ArrayDecoding`.

The final prompt became broader and more precise:

* `Int4ArrayLeafCodec` is treated as a check of the architectural direction, not as a goal in itself;
* arrays, structs, domains, enums, ranges, and multiranges are analysed as parts of one codec system;
* the JDBC APIs are described as adapters on top of the Codec API;
* the public SPI and standalone encode/decode became part of the design constraints;
* performance/correctness/usability questions are split out into separate checkable blocks;
* the review result is focused on architectural risks rather than on a random set of code remarks.
