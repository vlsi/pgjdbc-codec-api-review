# Prompt: comparing architecture reviews for the Codec API in pgjdbc

I asked several different agents to assess the quality and scalability of the `arraycodec` implementation and the public Codec API in pgjdbc.

The agents had access to the source code. Now I need not a new architecture review to replace them, but an objective comparison of their results.

## Context

The original task given to the agents was this: `../1-review-prompt-creation/design-review-prompt.md`

The agents' responses:
* `../2-review-execution/fable5.md`
* `../2-review-execution/gpt55.md`
* `../2-review-execution/opus48.md`

## Main goal

Compare the analysis quality of the different models and help me understand:

* where the results agree;
* where they differ;
* which conclusions only one model reached;
* which conclusions are confirmed by the current code;
* which conclusions look unconfirmed or hallucinated;
* which important problems from the original task the models missed;
* which of the proposed plans better helps implement `struct` / `array` / Codec API in pgjdbc quickly and scalably.

Do not rush to the final answer. If a model's claim sounds plausible but can be checked against the code, check it first. If the data is insufficient, mark the claim as unconfirmed rather than as an error.

## Important constraint

Do not perform the original architecture review again as the main result.

Use the original task as a rubric for assessing the models' responses. New findings of your own are allowed only in a separate section, `Problems missed by all models`, and only if they are confirmed by the code or follow clearly from the JDBC/PostgreSQL API.

## Comparison procedure

### 1. Extract atomic claims

First, break each model's response into individual statements.

A single statement should describe one specific problem, risk, limitation, or recommendation. Do not merge different topics into one claim.

For each claim, record:

* the claim identifier;
* which model made it;
* a brief formulation;
* the category;
* the severity;
* which classes, methods, APIs, or scenarios it refers to;
* whether the model's response contains evidence or it is only an assumption;
* whether the claim can be checked against the current code;
* the result of the check.

Categories:

* public SPI / API boundary;
* type identity and metadata;
* codec registry and lookup;
* scalar codecs;
* array codecs;
* composite / struct codecs;
* domains / enums / ranges / multiranges;
* JDBC compatibility;
* standalone encode/decode;
* performance / allocations / boxing;
* error handling / usability;
* migration from `ArrayEncoding` / `ArrayDecoding`;
* tests / benchmarks;
* maintainability;
* other.

Severity:

* `blocker` — prevents scaling the architecture or stabilising the public API;
* `high` — will lead to noticeable correctness/API/performance problems;
* `medium` — an important risk, but it can be fixed incrementally;
* `low` — a local improvement, clarification, or polish;
* `nit` — a minor remark.

Check status:

* `confirmed` — confirmed by the code or specification;
* `partially confirmed` — broadly correct, but the formulation is too broad or imprecise;
* `unclear` — not enough data, or the check requires additional experiments;
* `false / hallucinated` — contradicts the code or specification;
* `design trade-off` — not an error, but a compromise with several reasonable options;
* `opinion` — a matter-of-taste recommendation without a strict criterion.

### 2. Check the substantive claims against the code

For claims with status `blocker`, `high`, or with an impact on the public API, you must check the code where possible.

Pay particular attention to statements about:

* registering codecs by OID, type name, Java class, or schema-qualified name;
* support or lack of support for `Array`, `Struct`, `SQLData`, `SQLInput`, `SQLOutput`, `CallableStatement`;
* binary/text parity;
* lower bounds, multidimensional arrays, null elements;
* primitive fast path and boxing;
* separation of the public API and the internal implementation;
* the presence of unnecessary dependencies of the public API on `PGStream`, `QueryExecutor`, `Field`, `Tuple`, `ArrayEncoding`, `ArrayDecoding`;
* the behaviour of `ResultSet.getObject(int, Map<String, Class<?>>)`;
* qualified and quoted identifiers;
* cache invalidation for type metadata.

Do not call a design trade-off an error if PostgreSQL or JDBC has no obvious universal solution.

For example, if a model claims that registering a custom codec by type name without a schema is wrong, check the alternatives:

* an OID is stable for built-in types, but is not a portable identifier of a user-defined type across databases;
* an unqualified type name is more convenient for the user, but can be ambiguous;
* schema-qualified registration is more precise, but requires the user to bind explicitly, as in `myschema.my_type -> MyCodec`;
* `search_path` makes resolution context-dependent.

In such cases, state the compromise, the cost of the alternatives, and a practical recommendation.

### 3. Build the matrix of agreement

After extracting the claims, build a matrix:

| Claim | Category | Fable 5 | Opus 4.8 | GPT 5.5 | Code check | Verdict |
| --- | --- | --- | --- | --- | --- | --- |
| ... | ... | yes/no | yes/no | yes/no | confirmed / unclear / ... | ... |

If there are two models, use only the columns for the two models.

### 4. Group the results

List separately:

* confirmed problems found by all models;
* confirmed problems found by several models;
* confirmed problems found by only one model;
* unconfirmed or doubtful claims;
* claims that look like hallucinations;
* design trade-offs that the models may have wrongly presented as bugs;
* important topics from the original task that the models missed.

### 5. Assess the models' plans

Compare the proposed action plans not by the elegance of their wording, but by their practical suitability for pgjdbc.

Criteria:

* whether the plan helps reach a working implementation quickly;
* whether it prematurely locks in a poor public API;
* whether it preserves the path to a primitive fast path without excessive duplication;
* whether it allows replacing `ArrayEncoding` / `ArrayDecoding` incrementally;
* whether it covers JDBC compatibility, not just the internal codec model;
* whether it accounts for custom codecs and third-party extensions;
* whether it gives a clear order of work;
* whether it separates the mandatory architectural decisions from tasks that can be deferred;
* whether it avoids requiring the whole system to be rewritten before the first verifiable results appear;
* whether it proposes tests or experiments that genuinely reduce risk.

At the end, say which plan best achieves the goal of a quick and scalable implementation of `struct` / `array` / Codec API, and why.

If the best result comes from a combination of plans, assemble a merged plan.

### 6. Identify the decisions that must be made before scaling

List separately 5-10 decisions that must be made before extending the implementation from `int4` to the other types.

For each decision, state:

* why it blocks or reduces the risk of scaling;
* which options exist;
* which option looks preferable now;
* what can be deferred.

Do not mix these decisions with ordinary implementation tasks.

## Format of the final answer

Give the result in this structure:

1. `Final verdict`
   * which model's response is the most useful;
   * which response is the best evidenced;
   * where the models mostly agree;
   * where they diverge.

2. `Methodology`
   * which responses were compared;
   * which parts of the code were checked;
   * which conclusions were left unchecked.

3. `Claims matrix`
   * the matrix of agreement and check statuses.

4. `Confirmed shared conclusions`
   * problems found by all or several models.

5. `Unique useful findings`
   * problems found by only one model, but confirmed by the code or specification.

6. `Doubtful or false claims`
   * unconfirmed statements;
   * hallucinations;
   * claims where the model confused a trade-off with an error.

7. `Missed topics`
   * important points of the original task that the models did not cover, or covered superficially.

8. `Plan comparison`
   * which plan is better;
   * what to take from each plan;
   * what to discard.

9. `Decisions before scaling`
   * 5-10 architectural decisions that must be made before extending the implementation to the other types.

10. `Recommended next step`
    * a short, practical plan for the next iteration.

## Style

Write in Russian. State the conclusions technically and verifiably.

Do not use judgemental phrases such as "the model understands the task more deeply" unless this is backed by specific claims and a check.

If a claim is unchecked, say so. Do not turn an assumption into a conclusion.

If a conclusion depends on project policy or the future public API, separate the fact from the recommendation.
