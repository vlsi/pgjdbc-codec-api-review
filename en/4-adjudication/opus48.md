I have read all the input files: the original prompt, the three primary reviews (Fable 5, GPT 5.5, Opus 4.8), the adjudication-procedure prompt, and both comparison results. Before assembling the final verdict, I will resolve the key divergences by checking against the code.

The main divergences between the two comparisons worth resolving with code:
1. **domain-over-array**: comparison_opus48 marks it `confirmed`, comparison_gpt55 marks it `partially confirmed` ("PG catalog behavior not checked").
2. **attisdropped**: both agree this is a false claim, but they cite different lines (859 vs 864) — I will confirm the filter.
3. **binary range** severity: gpt55 = `blocker`, opus48 = `high (latent)`.
4. unique claims about the element OID in `MultiDimArrayBinary.decode` and trailing bytes.

I will check the code.

Confirmed against the code:
- `resolveByTyptype` ([CodecRegistry.java:484-506](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L484)) checks `isArray()` **first**, before `isComposite()`/`isDomain()`/`isEnum()`/range.
- `isArray()` = `typcategory == 'A'` ([PgType.java:390](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgType.java#L390)); `isDomain()` = `typtype == 'd'` ([PgType.java:372](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgType.java#L372)).
- The `attisdropped` filter exists ([TypeInfoCache.java:864](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/TypeInfoCache.java#L864)) — Opus's dropped-attributes claim really is false. gpt55 cited line 859; the filter is in fact at 864 (the query starts at 859).

The critical divergence point, domain-over-array (`confirmed` vs `partially confirmed`), depends on whether a domain inherits the `typcategory` of its base type in PostgreSQL. That decides whether the precedence fix is needed before scaling. I will check it empirically — I will look at whether PostgreSQL is running.

Confirmed: binary range takes the subtype from `type.getTypelem()` ([RangeCodec.java:99](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L99)), and for a range type `typelem == 0` → the element codec does not resolve. The bug is real (latent, since range is not in the default binary set).

PostgreSQL is not running. The `confirmed` vs `partially confirmed` divergence on domain-over-array depends entirely on whether a domain inherits `typcategory='A'` from the base array — the two comparisons diverge precisely here, and the gpt55 hedge is methodologically correct. This is the most important divergence and it determines the "decision before scaling", so I will resolve it empirically.

A full pgjdbc harness is overkill for checking a single catalog fact. I will spin up a lightweight throwaway postgres and query `pg_type`.

The experiment gave an authoritative answer. I will delete the container.

All key divergences are resolved. I am assembling the final adjudication.

---

# Final adjudication: comparison of the two adjudications

## 1. Short verdict

**The two comparisons agree on ~90% of conclusions and contradict each other on none of the confirmed bugs.** That on its own is a strong reliability signal: independent runs converged on the same risk map.

**Where they agree (full consensus, confirmed by code in both):**
- the public SPI leaks internal types (`api.codec.*` → `jdbc.PgType`/`CodecContext` → `BaseConnection`/`core.TypeInfo`);
- decode has no zero-copy slice (a copy per element/field/bound);
- the new `MultiDimArray*` walker is not on the hot read path; there is no text-read for arrays → `ArrayEncoding`/`ArrayDecoding` cannot be removed yet;
- the registry is keyed by a bare name without a schema; the override/SPI/built-in/fallback layers are not separated;
- the range subtype is not loaded from `pg_range`; multirange is not supported;
- both agree: the best plan overall is Fable's, the best single architectural decoupling is Opus P4 (one `ArrayCodec` + an optional fast-path leaf), GPT's is a correct but thin skeleton;
- both flagged "45/45 tests" as unverified and found NO hallucinations in any primary review (except for one false claim by Opus-original itself — see §4).

**Where they diverge (only three substantive points, all resolved):**

| # | Divergence | A (gpt55) | B (opus48) | Resolution |
|---|---|---|---|---|
| 1 | Status of domain-over-array | `partially confirmed` (PG catalog not checked) | `confirmed` | **B is right.** Experiment: a domain inherits the `typcategory` of its base type |
| 2 | Severity of binary range | `blocker` | `high (latent)` | **B is more precise.** range is not in the default binary set → latent |
| 3 | Severity of decode-slice | `blocker` | `high` | Semantics. Final: `high`, but **contract-gating** |

**Which comparison is more useful.** For the adjudication task — **comparison_opus48 is slightly more useful**: it (a) gave the most precise domain-over-array diagnosis, which the experiment confirmed word for word; (b) disciplinedly separated trade-offs from bugs (it explicitly classed lower-bounds=1, number quoting, and domain-unwrap as deliberate compromises rather than defects); (c) explicitly resolved the Fable↔Opus contradiction on `attisdropped` in Fable's favour. **comparison_gpt55 is stronger on atomicity**: its 40-claim matrix breaks the registry problems down more finely (separately for `registerByName`-no-invalidate, `unregister`-no-restore, SPI-class-shadow) and it is the only one to raise two code-confirmable omissions (element OID, trailing-bytes validation).

**Reliability of the final verdict: high** for the confirmed issues (double convergence + I re-checked the disputed one against code and experiment), **medium** for the severity ranking (this is a question of policy about "what counts as a blocker"), and several perf claims remain `unresolved` without JMH/line-by-line reading (§5).

**Quality of the procedure (both).** Both extracted atomic claims, checked the substantive ones against the code via `fff`/`ast-grep`, separated facts from opinions, distinguished a hallucination (`attisdropped`) from unresolved (`45/45`), did not substitute a new architecture review for the comparison, and gave a practical next step. comparison_opus48 is tidier about marking `not checked`; comparison_gpt55 is more atomic. Both are fit for purpose.

---

## 2. Divergence matrix

`A` = comparison_gpt55, `B` = comparison_opus48. "Agree?" is about the conclusion, not the wording.

| Topic / claim | A | B | Agree? | Re-checked? | Final verdict |
|---|---|---|---|---|---|
| SPI leaks internal types | confirmed/blocker | confirmed/blocker | yes | yes (code) | **confirmed, blocker** |
| Offline blocked (`CodecContext` final, ctor package-private, `withTypeMap` throws) | confirmed | confirmed | yes | no | **confirmed, high** |
| Decode without slice → copies | confirmed/**blocker** | confirmed/**high** | fact yes, severity no | yes (code) | **confirmed, high (contract-gating)** |
| Registry: bare name, schema collision | confirmed+trade-off | confirmed | yes | yes (code) | **confirmed + trade-off on registration form** |
| Registry: layers/override/unregister | confirmed (atomic) | confirmed (grouped) | yes | yes (code) | **confirmed, high** |
| **domain-over-array precedence** | **partially confirmed** | **confirmed** | **partly** | **yes (experiment)** | **confirmed, high** |
| Binary range subtype from `typelem=0` | confirmed/**blocker** | confirmed/**high latent** | fact yes, severity no | yes (code) | **confirmed, high (latent)** |
| Multirange not supported | confirmed | confirmed | yes | yes (code) | **confirmed, medium** |
| New walker not on the hot path | confirmed | confirmed | yes | yes (code) | **confirmed, high** |
| No text-read leaf for arrays | confirmed | confirmed | yes | yes (code) | **confirmed, high** |
| `attisdropped` NOT filtered (Opus-original claim) | **false** | **false (not reproduced)** | yes | yes (code) | **false / hallucinated** |
| `unregister`/`reset` loses built-in | confirmed (unique GPT) | confirmed (credits GPT) | yes | no | **confirmed, medium** |
| `registerByName` does not invalidate `oidCache` | confirmed (unique Fable) | confirmed (unique Fable) | yes | no | **confirmed, low** |
| SPI `registerByClass(defaultJavaType)` shadows String | confirmed | confirmed | yes | no | **confirmed, medium** |
| Lower bounds lost (encode=1) | confirmed+trade-off | confirmed (trade-off) | yes | yes (code) | **confirmed, trade-off** |
| `computeDimensions` rank by static class → dims=1 | confirmed (**regression**) | confirmed (fact yes, "regression" not checked) | partly | partly | **fact confirmed; "regression vs ArrayEncoding" unresolved** |
| `BackpatchingBinarySink` package-private | partially / overstated | confirmed with a caveat | yes (both add a caveat) | yes (code) | **confirmed boundary-leak; caveat: scalar-streaming via `OutputStream` works** |
| `getArray(Map)`/`getResultSet(Map)` binary → `notImplemented`; Opus-original called it "covered" | confirmed (opus overstated) | confirmed (opus overstated) | yes | yes (code) | **confirmed; Opus-original overstated** |
| `encodeText`/`setArray` for a third-party `Array` via `toString()` | confirmed | — (not raised) | confirmed-by-one | no | **confirmed, medium** |
| element OID dropped in `MultiDimArrayBinary.decode` | raised (missed-by-all) | — | confirmed-by-one | partly | **confirmed-by-one, low/medium (matters for standalone/malformed)** |
| simple vs extended query mode missed by the models | raised (superficially) | raised (strongly) | yes | no | **confirmed: omission by the primary models** |
| "45/45 tests" = maturity | unclear | unclear | yes | no | **unresolved (not run)** |
| Plan: Fable best / Opus P4 / GPT skeleton | yes | yes | yes | — | **agreed** |

---

## 3. Final confirmed issues

Confirmed by code (and domain-over-array also by experiment). Severity is consolidated.

**Blocker (cemented into the public contract, fix it under `@Experimental`):**
1. **SPI leaks internal types.** `api.codec.BinaryCodec/TextCodec` take `org.postgresql.jdbc.PgType` and `CodecContext`; `CodecContext` hands back `BaseConnection`, `core.TypeInfo`, `Encoding`. Source: `BinaryCodec.java`, `CodecContext.java:275,285`. Any third-party codec drags in the internal core.

**High:**
2. **Decode without slice** → `Arrays.copyOfRange`/`new byte[]` per element/field/bound ([GenericArrayLeafCodec.java:116](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/GenericArrayLeafCodec.java#L116), [RangeCodec.java:120](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L120)). Contract-gating: it touches the signature of every decode method → decide before writing ~30 codecs. I resolve the severity divergence (blocker vs high) as `high, but contract-gating`: it can be fixed backward-compatibly via default delegation to the full buffer, so it is not an "implementation blocker" but a "signature-stabilisation blocker".
3. **domain-over-array is mis-routed into `ArrayCodec`.** **Resolved by experiment** (PostgreSQL 17):

   | typname | typtype | typcategory | typelem | base_category |
   |---|---|---|---|---|
   | `d_arr` (domain AS int4[]) | `d` | **`A`** | **0** | A |
   | `d_comp` (domain AS comp) | `d` | `C` | 0 | C |
   | `d_int` (domain AS int4) | `d` | `N` | 0 | N |
   | `d_rng` (domain AS int4range) | `d` | `R` | 0 | R |

   A domain inherits the `typcategory` of its base type. In `resolveByTyptype` ([CodecRegistry.java:486](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L486)), `isArray()` (`typcategory=='A'`) is checked **first** → `d_arr` goes to `ArrayCodec`, where `getTypelem()==0` breaks the element-codec resolution. **Only domain-over-array is affected**: domain-over-composite/scalar/range route correctly, because their checks use `typtype`, and the only typcategory check — `isArray()` — comes first. Opus's fix is precise: treat something as an array only when `typtype=='b' && typcategory=='A'`. **comparison_opus48 is right, the comparison_gpt55 hedge is lifted.**
4. **The new walker is not on the hot path.** `getArray()`/`getObject()` return `PgArray` → legacy `ArrayDecoding`; `MultiDimArrayBinary.decode` runs only in `decodeBinaryAs(int[].class/Integer[].class)`. The goal "remove `ArrayEncoding`/`ArrayDecoding`" is unreachable by design right now.
5. **No text-read leaf** for arrays in the new architecture → all text parsing is in legacy `ArrayDecoding`. Reinforced by a topic both missed: **simple query mode is always text** → without a text-read leaf, simple-query is entirely tied to legacy.
6. **Binary range subtype from `typelem`** ([RangeCodec.java:99](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/codec/RangeCodec.java#L99)): a range has `typelem==0`, `pg_range` is not loaded. Severity: **high (latent)** — range is not in the default binary transfer set; I resolve blocker-vs-high in favour of `high/latent` (comparison_opus48). The first person to enable binary for a range gets an error.
7. **Registry identity and layers**: bare name without a schema ([CodecRegistry.java:84,410](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/CodecRegistry.java#L410)); SPI/built-in/custom in the same maps; `registerByClass(getDefaultJavaType)` may shadow String; SPI collisions are last-wins silently; load errors are swallowed.

**Medium:**
8. Multirange (`typtype='m'`) is not in `resolveByTyptype` → a silent Fallback.
9. `unregister`/`reset` do not restore an overridden built-in (a reset-scenario bug for pools).
10. `encodeText`/`setArray` of a third-party `Array` via `toString()` → garbage in the SQL without an error.
11. `computeDimensions` computes rank by the static class → a runtime-nested `Object[]` gets dims=1 (fact confirmed; "regression vs ArrayEncoding" — unresolved, §5).
12. `getArray(Map)`/`getResultSet(Map)` on binary → `notImplemented` ([PgArray.java:213,478](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/PgArray.java#L213)).
13. `getCodecContext()` allocates a new context on every call, called per-element in `ArrayDecoding`.

**Low:** `registerByName` does not invalidate `oidCache`; single-arg `getBinaryCodec(int)` → Fallback on a cold cache; enum→Java enum does not decode; double rectangular validation on encode.

---

## 4. Final false / hallucinated claims

1. **`attisdropped` is not filtered (Opus-original review claim, CR6).** **False.** The filter exists: [TypeInfoCache.java:864](https://github.com/vlsi/pgjdbc/blob/codec-api-review-2026-06-12/pgjdbc/src/main/java/org/postgresql/jdbc/TypeInfoCache.java#L864) (`AND NOT a.attisdropped`, query from 859). On top of that, PostgreSQL `record_send` excludes dropped columns from the binary format, and `row(...)::text` does too. There is no positional desync. **Both comparisons converged on false** (comparison_opus48 explicitly resolved the Fable↔Opus contradiction in Fable's favour). Opus-original honestly flagged it as "must verify" but presented it as a probable bug — that is an overestimate, not a fabrication.
2. **Opus-original: `getArray(Map)`/`getResultSet`/`getArray(Map)` "covered".** Overstated: slices and `getResultSet()` exist, but binary `getArray(Map)` and `getResultSet(Map)` throw `notImplemented`. Both comparisons caught this.
3. **Opus-original (via the comparison): "a third-party streaming codec *always* gets the slow path".** Overstated, not a lie: `BackpatchByteArrayOutputStream` is public, a scalar element-codec streams via `OutputStream`; container-level backpatch is not available. Both comparisons added the same caveat.

No pure hallucinations (invented behaviour/non-existent code) were found in the comparisons themselves — every file:line I re-checked lands in real code.

---

## 5. Unresolved questions

| Question | Why unresolved | What check is needed |
|---|---|---|
| "Regression vs `ArrayEncoding`" for a runtime-nested `Object[]` (C25/C17) | dims=1 is confirmed, but the old `ArrayEncoding` behaviour for `new Object[]{new Object[]{...}}` was not cross-checked | run `createArrayOf("int4", Object[]{Object[]{...}})` on the old and new path, compare |
| Fable perf claims: encode→decode round-trips in `PgStruct`/`PgCallableStatement`, growth of `TYPE_ALIASES`, megamorphic dispatch on `getInt`, `getString` bypassing the codec | both comparisons marked it "not checked" line by line | a targeted JMH + line-by-line reading of `PgStruct.getAttributes`/`PgCS` OUT |
| "45/45 tests" as maturity | the tests were not run in either comparison | `./gradlew --quiet test` on the codec set |
| element OID validation / trailing-bytes (raised only by gpt55) | structurally `RangeCodec` checks truncation but not trailing after the upper bound; `MultiDimArrayBinary.decode` reads the element OID from the header and discards it | code-level: add a mismatch/trailing-assert and test against a malformed payload |
| Real binary/text parity on the server | neither comparison ran parity on PostgreSQL | parity harness (Fable's plan) |

---

## 6. Design trade-offs (not bugs — PostgreSQL/JDBC has no single answer)

1. **Registration by name without a schema.** Fact: a collision of identical `typname` across different schemas is real (commit `099675898` suppresses it in tests). But this is a **trade-off, not a bug**: OID is precise but not portable across databases for user types; an unqualified name is convenient and follows `search_path`; schema-qualified is precise but requires an explicit binding `myschema.t → Codec`. **Recommendation:** runtime resolution by OID; registration in three forms (OID / schema-qualified / unqualified alias) with explicit conflict rules. Both comparisons correctly did NOT call this a pure bug.
2. **Lower bounds=1.** encode always writes 1, decode ignores non-1 bounds. Compatible with the old JDBC behaviour of pgjdbc. **Pin it as a contract in the javadoc**, do not "fix" it.
3. **Domain unwrap.** `DomainCodec` delegates to the base codec, losing the domain identity and typmod. For `getObject` the unwrap is fine; for names in `Struct`/`SQLData` and for domain-over-array it is not. **Recommendation:** keep the domain's `TypeDescriptor` (identity in the metadata), reuse the base codec for the value. A deliberate decision, not a loss.
4. **Number quoting in a text array.** generic quotes, `Int4ArrayLeafCodec` does not. PostgreSQL accepts both → functionally fine, but different bytes on the wire. **Check that the tests do not assert exact literals.**
5. **`Array.getArray()` stays boxed (`Integer[]`).** Historically per JDBC — it cannot be changed; primitives only via `getObject(int[].class)`. The current design separates this correctly.

---

## 7. Issues missed by both comparisons

I add only what is confirmed by code/spec and needed for the §8 decisions.

1. **The chain "simple query mode ⇒ always text ⇒ depends on legacy text-read".** comparison_gpt55 mentioned simple/extended in passing, comparison_opus48 more strongly, but **neither linked it to prioritising the text-read leaf as a gate on removing `ArrayDecoding`**. Fact: in the simple protocol, binary codecs are not used at all. This raises the priority of the text-read leaf (Decision 4) from "next iteration" to "mandatory before removing legacy".
2. **The element OID from the binary header is discarded** in `MultiDimArrayBinary.decode` (raised only by gpt55, I confirm the structure). Tolerable for a trusted ResultSet, but for standalone/raw decode and a malformed payload it needs mismatch validation. It maps directly to the "malformed binary payload errors" rubric from the original prompt.
3. **typmod at the field level vs the type level.** The Codec API works with `PgType`, but typmod (precision of `numeric`, length of `varchar`) lives on `PgField`/in the domain (`typtypmod`). It will be needed for offline encode and for domain types — neither comparison separated type-level from field-level metadata.

---

## 8. Decisions before scaling (from int4 to the rest of the types)

1. **Public API boundary.** Introduce `api.codec.TypeDescriptor` (read-only: oid, (schema,name), typtype, typcategory, typmod, elementOid, arrayOid, baseTypeOid, subtypeOid, fields, typdelim) and `CodecContext` as an **interface** with the wire part, without `getConnection()`/`BaseConnection`/`core.TypeInfo`. *Blocks:* once there are third-party dependencies, the contract cannot be reworked. *Now:* do it under `@Experimental`. *Defer:* the descriptor's makeup for multirange.
2. **Decode-slice signature.** `(byte[] data, int off, int len)` vs `ByteBuffer` vs a cursor. *Preferred:* `(byte[], off, len)` + default delegation to the full buffer (compatible), ownership = borrowed for the duration of the call. *Blocks:* it touches every decode method. *Defer:* a decode-streaming visitor (the slice is its foundation).
3. **A single array model (Opus P4).** A universal `ArrayCodec` composing element codecs, + an **optional** fast-path leaf interface on the scalar codec for `int[]/long[]/...`. *Blocks:* otherwise it is a separate codec+leaf per int2/int8/float4/float8/bool — the very proliferation. *Defer:* a fast-path for variable-width (text/numeric — straight to generic/streaming).
4. **Text-read leaf + a shared tokenizer** (array/composite/range). *Raised to "mandatory before removing legacy"* because of the link with simple-query-mode (§7.1). *Now:* pin the interface so that `ArrayLeafCodec` can withstand it.
5. **Registry identity and conflicts.** Key by OID (runtime) + (schema-optional name) for registration; layers `explicit OID > connection-name > SPI > built-in > fallback`; fix `unregister`/`reset`; invalidate `oidCache` on name registration; deterministic choice + logging on an SPI conflict. *Defer:* predicate registration (PostGIS, OID per-DB) — leave room for it.
6. **Classification precedence (confirmed by experiment).** Treat something as an array only when `typtype=='b' && typcategory=='A'`, or check `typtype` before `typcategory`. A small fix; without it domain-over-array drifts into `ArrayCodec`. *Now:* do it + a regression test on domain-over-array/scalar/composite/enum/range.
7. **Range/multirange metadata.** Load `pg_range` (subtype) + `PgRangeType.subtypeOid`; pass the resolved `PgType` into `getBinaryCodec(oid, pgType)`. *Now:* `pg_range`; multirange — metadata-only with an explicit error in v1.
8. **Format capabilities.** Replace `instanceof BinaryCodec`/"threw an exception" with explicit `supportsBinaryRead/Write`, `textRead/Write`; choose the format by the intersection of server × codec × settings. *Blocks:* for third-party binary-only/text-only codecs.
9. **Domain and typmod contract** (trade-off §6.3): unwrap to the base or keep identity for names/domain-over-array; whether to pass through `typtypmod`. Pin it as a contract.

---

## 9. Recommended next iteration

In order of execution, minimising contract risk:

1. **Perimeter + precedence (cheap, unblocks the rest).** `api.codec.TypeDescriptor` + a `CodecContext` interface (Decision 1); fix `resolveByTyptype` to `typtype=='b' && typcategory=='A'` (Decision 6) + a regression test for domain-over-array/scalar/composite/enum/range. **The experiment has already shown that the test will catch the bug.**
2. **Pin the decode-slice** with default delegation to the full buffer (Decision 2), convert `GenericArrayLeafCodec`/`CompositeCodec`/`RangeCodec` — remove the per-element `copyOfRange` before there are 30 codecs.
3. **Prototype Opus P4 on `int8` or `bool`** (Decision 3): prove that a fast-path as an optional interface on the scalar codec removes the duplication. A clean prototype = the template for the rest of the fixed-width types.
4. **Parity harness as a gate** (Fable's plan): random value → server roundtrip → `text-decode == binary-decode == original`; a matrix of null/empty/ragged/lower-bounds/0-dim. Without it, every migrated type silently changes one of the behaviours.
5. **In parallel — pinpoint confirmed bugs that do not change the contract:** `unregister`/`reset` built-in, single-arg `getBinaryCodec` in `RangeCodec`, `encodeText`/`setArray` of a third-party `Array` via `getArray()`, the `CodecContext` cache on the connection, `registerByName`→invalidation of `oidCache`.

The text-read leaf (Decision 4) and `pg_range` (Decision 7) are the next iteration, but sketch the text-leaf interface now. Once 1–3 are agreed, we can descend to the code-level findings (`RangeCodec` binary, the `CompositeCodec` text parser, `MultiDimArraySupport.computeDimensions`, the registry invariants).

---

**What I did beyond reading:** I re-checked `resolveByTyptype`/`PgType`/`TypeInfoCache`/`RangeCodec` against the code and **spun up PostgreSQL 17 for an experiment** that resolved the main divergence (a domain inherits `typcategory='A'` → domain-over-array confirmed). I did NOT redo the primary architecture review and did not re-compare the primary answers from scratch — I only adjudicated the two comparisons.
