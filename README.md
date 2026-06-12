# Comparing LLM architecture reviews of the pgjdbc Codec API

This repository is an experiment in comparing architecture reviews that several LLM agents produced for the public Codec API in pgjdbc.

The interesting part is not only what the models concluded, but the comparison process itself:

* how the original engineering task was framed for an architecture review;
* what problems the different models found;
* which findings agreed, diverged, or turned out to be unsupported;
* how the raw engineering brief was focused into a design-review prompt;
* how to turn several long LLM answers into a checkable matrix of claims;
* what a final adjudication pass looks like when two independent comparisons nearly agree.

Every model request — the primary reviews, the comparisons, and the final comparison — ran at maximum reasoning effort. The models were Fable 5, GPT 5.5, and Opus 4.8.

The reviewed code is the state of the `vlsi/pgjdbc` fork pinned by the tag [`codec-api-review-2026-06-12`](https://github.com/vlsi/pgjdbc/tree/codec-api-review-2026-06-12) (commit `4b2df19`).

Русская версия: [`ru/README.md`](ru/README.md).

## What's here

This is the English version; the Russian one lives in [`ru/`](ru/README.md). The files follow the pipeline stages, from the original task to the final comparison.

### The original task

* [`en/1-review-prompt-creation/design-review-prompt.md`](en/1-review-prompt-creation/design-review-prompt.md) — the prompt for the first architecture review of the Codec API: arrays, structs, user-defined types, standalone encode/decode, JDBC adapters, registry, metadata, performance, and migration away from `ArrayEncoding` / `ArrayDecoding`.
* [`en/1-review-prompt-creation/initial-task.md`](en/1-review-prompt-creation/initial-task.md) — the original task statement, before refinement.
* [`en/1-review-prompt-creation/refinement-dialogue.md`](en/1-review-prompt-creation/refinement-dialogue.md) — a condensed transcript of the refinement that turned the original statement into the final prompt.

The original statement already carried plenty of technical context, but it had not yet pinned down the important forks: code review or design review, whether the Codec API is a public SPI, which PostgreSQL types are in scope, and whether to design for a standalone encode/decode API.

What proved useful was not that an LLM "improved the prompt", but the iteration that surfaced the hidden goals of the work. After the refinement, the prompt was no longer a request to look at `Int4ArrayLeafCodec`; it had become an architecture review of a public codec system for every PostgreSQL type.

### The primary architecture reviews

* [`en/2-review-execution/fable5.md`](en/2-review-execution/fable5.md) — Fable 5's review.
* [`en/2-review-execution/gpt55.md`](en/2-review-execution/gpt55.md) — GPT 5.5's review.
* [`en/2-review-execution/opus48.md`](en/2-review-execution/opus48.md) — Opus 4.8's review.

All three answer the same prompt. Read them as independent attempts to find the architectural risks in one codebase.

### The comparison procedure

* [`en/3-comparison/comparison-prompt.md`](en/3-comparison/comparison-prompt.md) — the prompt for comparing the primary reviews.

This prompt sets the procedure: break each answer into atomic claims, check the substantive ones against the code, separate facts from opinions, flag hallucinations, build a matrix of agreement, and propose a practical next-step plan.

### Results of the comparison

* [`en/3-comparison/gpt55.md`](en/3-comparison/gpt55.md) — the comparison by GPT 5.5.
* [`en/3-comparison/opus48.md`](en/3-comparison/opus48.md) — the comparison by Opus 4.8.

Both compare the same primary reviews, independently. That is useful in itself: you can see how stable the comparison procedure turns out to be.

### Comparing the comparisons

* [`en/4-adjudication/adjudication-prompt.md`](en/4-adjudication/adjudication-prompt.md) — the prompt for the final adjudication pass.
* [`en/4-adjudication/gpt55.md`](en/4-adjudication/gpt55.md) — the final comparison of the two comparisons, by GPT 5.5.
* [`en/4-adjudication/opus48.md`](en/4-adjudication/opus48.md) — the final comparison of the two comparisons, by Opus 4.8.

The two adjudication results nearly matched. That is a good sign: the substantive claims and the practical conclusions held up when the model doing the comparison changed.

## How to read this

For the result in a hurry:

1. Start with [`en/4-adjudication/gpt55.md`](en/4-adjudication/gpt55.md) or [`en/4-adjudication/opus48.md`](en/4-adjudication/opus48.md).
2. Open [`en/3-comparison/gpt55.md`](en/3-comparison/gpt55.md) and [`en/3-comparison/opus48.md`](en/3-comparison/opus48.md) to see how the final claims were reached.
3. Go back to the primary reviews if you want to know which model first spotted a given problem.
4. Open [`en/3-comparison/comparison-prompt.md`](en/3-comparison/comparison-prompt.md) for the comparison method itself.
5. Open [`en/1-review-prompt-creation/design-review-prompt.md`](en/1-review-prompt-creation/design-review-prompt.md) for the full engineering context.

If you care about the methodology rather than pgjdbc:

1. Read [`en/1-review-prompt-creation/initial-task.md`](en/1-review-prompt-creation/initial-task.md) for the original statement.
2. Read [`en/1-review-prompt-creation/refinement-dialogue.md`](en/1-review-prompt-creation/refinement-dialogue.md) to see which goals were clarified before the reviews ran.
3. Read the final [`en/1-review-prompt-creation/design-review-prompt.md`](en/1-review-prompt-creation/design-review-prompt.md).
4. Read one primary review.
5. Read the comparison prompt.
6. Compare the two comparison results.
7. Read the final adjudication prompt and one final result.

## Method

The experiment runs in several stages.

0. First, the raw engineering statement is refined into a design-review prompt. The point of this step is not to reword the text but to surface the hidden decisions: the type of review, the boundaries of the public API, the type scope, standalone encode/decode, and what counts as a useful result.
1. Several models then run an architecture review of the same code, independently.
2. Other models do not redo the review; they compare the results — extracting claims, checking them against the code, and labelling each with a status.
3. Finally, the two comparisons are themselves compared, to see where the adjudication results already agree.

The key idea is to distrust a confident statement that comes without evidence.

Every substantive claim lands in one of these statuses:

* `confirmed` — backed by the code or the spec;
* `partially confirmed` — broadly right, but stated more widely than the facts support;
* `unclear` — not enough data;
* `false / hallucinated` — contradicted by the code or the spec;
* `design trade-off` — not a bug, but a choice between reasonable options;
* `opinion` — a recommendation with no hard criterion.

This process helps separate:

* real architectural risks;
* debatable design trade-offs;
* unsupported claims;
* hallucinations;
* useful but non-urgent recommendations.

## What the experiment showed

The most stable conclusions:

* the public Codec SPI still leaks pgjdbc's internal types;
* the registry and lookup rules need a more explicit model of type identity, override, and fallback;
* the array path is not yet a single codec-based hot path;
* range and multirange metadata need their own model, not heuristics via `typelem`;
* the JDBC compatibility gaps matter as much as the internal codec architecture;
* the primitive fast path has to be designed explicitly, or the general container model becomes too boxing-heavy;
* some of the models' claims turned out to be design trade-offs rather than bugs.

The most useful output was not a single "best" answer, but the overlap of the independent results plus a list of the divergences that could be checked against the code.

## How to reproduce the process

1. Give several models [`en/1-review-prompt-creation/design-review-prompt.md`](en/1-review-prompt-creation/design-review-prompt.md) and access to the same code.
2. Save their answers as separate Markdown files.
3. Give another model [`en/3-comparison/comparison-prompt.md`](en/3-comparison/comparison-prompt.md), the original prompt, and all the primary answers.
4. Repeat step 3 with a different model.
5. Give a third model [`en/4-adjudication/adjudication-prompt.md`](en/4-adjudication/adjudication-prompt.md) and the two comparison results.
6. Check the short list of unresolved and high-severity claims by hand.

For a new project, you only need to swap in the original architecture-review prompt and the primary model answers. The comparison procedure barely depends on pgjdbc.

To reproduce the prompt preparation as well as the comparison pipeline, start from the raw task statement and pin down the answers to a few questions separately:

* which type of review you need;
* what counts as the public API;
* which entities are in scope;
* which performance, correctness, and usability goals matter;
* what results count as useful once the review is done.

## Disclaimer

This is a research artefact, not an official pgjdbc document.

LLM answers can contain mistakes. Check the important claims against the code, the tests, the JDBC documentation, and PostgreSQL's behaviour.

The value of the experiment is not that one model is "right", but that independent answers can be reduced to a checkable form: claims, evidence, status, unresolved questions, and next steps.
