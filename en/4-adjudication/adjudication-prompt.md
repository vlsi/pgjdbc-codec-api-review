# Prompt: final comparison of two comparison procedures of the architecture review

I am researching the results of an architecture review of the public Codec API for `struct` / `array` / user-defined types in pgjdbc.

I have:

* the original prompt for the primary architecture review;
* several primary model responses;
* the prompt for the "comparison procedure" that compares these responses;
* two independent results of this comparison procedure, produced by different agents.

Now I need to run a final adjudication pass: compare not the primary reviews directly, but the two comparison results, resolve the divergences, and assemble a single consolidated verdict.

## Input files

The original prompt for the primary architecture review:

* `../1-review-prompt-creation/design-review-prompt.md`

The primary model results:

* `../2-review-execution/fable5.md`
* `../2-review-execution/gpt55.md`
* `../2-review-execution/opus48.md`

The prompt used to run the comparison procedure:

* `../3-comparison/comparison-prompt.md`

The results of the two independent comparisons:

* `<insert path to the first comparison result>`
* `<insert path to the second comparison result>`

If the names of the files with the comparison results are already known, use them instead of the placeholders above.

## Whether to take the comparison-procedure prompt into account

Yes. Use `../3-comparison/comparison-prompt.md` as methodological context.

It is needed to understand:

* which criteria the agents were supposed to apply;
* which claim statuses they were supposed to distinguish;
* what counted as confirmed, unconfirmed, a trade-off, or a hallucination;
* where a comparison result could have deviated from the prescribed procedure.

That said, do not re-run the whole comparison procedure from scratch. Use its prompt only as a rubric for assessing the quality of the two comparisons already produced.

## Main task

Do not re-run the primary architecture review.

Do not compare the primary model responses from scratch unless this is needed to resolve a specific divergence between the two comparisons.

Compare precisely the two comparison-procedure results and assemble a final consolidated verdict:

* which conclusions agreed;
* where the comparisons diverge;
* which divergences are material;
* which divergences can be resolved by checking the code or the specification;
* which remain unresolved;
* which final claims should be considered confirmed;
* which claims should be considered false, hallucinated, or unconfirmed;
* which decisions need to be made before scaling the Codec API implementation.

## Procedure

### 1. Extract the final claims from the two comparisons

From each comparison result, extract:

* confirmed issues;
* partially confirmed issues;
* unresolved / unclear claims;
* false / hallucinated claims;
* design trade-offs;
* topics missed by the primary models;
* recommended architectural decisions;
* recommended next steps.

Do not mix claims of different types. If one comparison result merged several problems into a single item, decompose it into atomic claims.

### 2. Build a divergence matrix

Build a table:

| Topic / claim | Comparison A | Comparison B | Agree? | Needs re-checking? | Final verdict |
| --- | --- | --- | --- | --- | --- |
| ... | ... | ... | yes / partially / no | yes / no | confirmed / false / unresolved / trade-off |

Mark separately:

* divergences over the very presence of a claim;
* divergences in severity;
* divergences in the status `confirmed` / `unclear` / `false` / `design trade-off`;
* divergences in the recommended plan of work;
* cases where one comparison checked a claim against the code and the other did not.

### 3. Resolve the important divergences

For important divergences, do not pick a side based on style, confidence, or text detail.

If a divergence affects the public API, the architectural order of work, correctness, JDBC compatibility, or the performance hot path, try to resolve it by:

* checking the current code;
* checking the JDBC/PostgreSQL API;
* cross-checking against the original prompt;
* cross-checking against the comparison-procedure prompt.

If a divergence cannot be reliably resolved without additional experiments, mark it as `unresolved` and state which experiment or check is needed.

### 4. Do not confuse errors and trade-offs

Do not call a design trade-off an error if PostgreSQL, JDBC, or pgjdbc has no obvious universal solution.

Be especially careful when assessing claims about:

* registering custom codecs by OID, type name, schema-qualified name, or Java class;
* the effect of `search_path`;
* preserving domain identity versus unwrapping a domain into its base type;
* quoted identifiers;
* primitive arrays vs boxed arrays;
* the public API for offline encode/decode;
* fallback and priority rules for third-party codecs;
* binary-only or text-only codecs.

For such topics, state:

* the fact according to the current code or API;
* the possible design options;
* the cost of each option;
* a practical recommendation.

### 5. Check whether the comparisons followed the procedure

Briefly assess the quality of the two comparison results:

* whether they extracted atomic claims;
* whether they checked material claims against the code;
* whether they separated facts from opinions;
* whether they distinguished hallucinations from unresolved claims;
* whether they replaced the comparison with a new architecture review;
* whether they gave a practical next-step plan.

Do not turn this section into an assessment of "which model is smarter". Assess only the fitness of the result.

## Output format

Give the answer in the following structure:

1. `Brief verdict`
   * where the two comparison procedures agree;
   * where they diverge;
   * which comparison result is more useful and why;
   * how reliable the final verdict is.

2. `Divergence matrix`
   * a table of claims and final statuses.

3. `Final confirmed issues`
   * confirmed problems only;
   * state the severity and the source of confirmation.

4. `Final false / hallucinated claims`
   * claims that contradict the code or the specification;
   * claims where the model clearly invented the behaviour.

5. `Unresolved questions`
   * questions that could not be resolved;
   * which check is needed for each question.

6. `Design trade-offs`
   * topics where there is no single correct solution;
   * options and recommendations.

7. `Problems missed by both comparisons`
   * add only if it is needed for the final verdict;
   * confirm with the code, the specification, or the original prompt.

8. `Decisions before scaling`
   * 5-10 architectural decisions that need to be made before extending the implementation from `int4` to the remaining types.

9. `Recommended next iteration`
   * a short, practical plan for the next iteration.

## Constraints

* Do not produce another full architecture review.
* Do not retell the primary model responses at length.
* Do not turn a confident wording into proof.
* If a claim is not checked, mark it as `unresolved` or `unclear`.
* If a conclusion depends on a future public-API policy, separate the fact from the recommendation.
* Write in Russian, technically and verifiably.
