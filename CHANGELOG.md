# Changelog

Notable changes to the TDQS specification. Scores are calibrated to a rubric+model pair, so each entry states what re-scores.

## v1.3 — 2026-09-03

The tool-scoring prompt now carries the output schema itself. This re-scores the registry.

- **`outputSchema` is passed to the evaluator in full**, in an `<output-schema>` block beside `<input-schema>`. It was already a scored input and already covered by `inputHash`, but the prompt only ever saw the boolean `hasOutputSchema` — so Contextual Completeness discounted the description's duty to explain return values on the strength of a schema nothing had read.
- **Contextual Completeness now grades the output schema by what it documents.** A return shape with described fields relieves the description of explaining return values; a bare `{"type": "object"}` relieves it of nothing. Same correction in the maintainer checklist, which said "provide an output schema" full stop — advice a bare `{"type": "object"}` satisfied.
- **`hasOutputSchema` is withheld from the prompt**, joining the invocation-cost signals and `definitionBytes`. Still stored and published. Once the block is in the prompt, a boolean restating whether it is empty grounds nothing.

Rollout needs a one-off pass. No definition field changed, so no `inputHash` changed, so nothing re-enqueues on its own and no aggregate drifts — the v1.1 situation, needing the v1.1 remedy: re-enqueue keyed by row rather than by hash. Unlike v1.1 it cannot be scoped to where there is something to find, because the prompt text changed for every tool. The information delta is confined to tools carrying an output schema; for the rest, `Has output schema: false` became an empty block.

Deferred: none of the three calibration examples carries an output schema, so the documented-vs-bare distinction ships unexemplified. It belongs with the two examples v1.2 deferred.

## v1.2 — 2026-09-01

The first three changes respond to [#3](https://github.com/glama-ai/tool-definition-quality-score/issues/3). Nothing here re-runs an LLM call, and only the last change moves a stored score.

- **Parameter Semantics guidance is now channel-neutral.** The checklist told maintainers to document parameters in the schema, not the description. The rubric never required that: low coverage shifts the burden onto the description, and either channel earns the credit. Guidance only; no prompt changed.
- New context signal `definitionBytes`: the tool's share of the `tools/list` payload. Stored and published, withheld from every prompt, priced by nothing. It derives from the serialization already built for `inputHash`, so backfilling it is a one-off pass keyed by row and re-scores nothing.
- Fixed an overlap in the Tool Count Appropriateness anchors, which claimed 25 tools as both `16-25` (score 3) and `25+` (score 2). The upper band now starts at 26. Affects only servers with exactly 25 tools.
- **Rounding is now defined: integer half-up, everywhere.** `round1(p, q)` rounds the exact rational `p / q` half-up to one decimal in integer arithmetic. `computeTdqs` and the three server rollups all use it, and the rollups are restated with integer arguments. Before, `computeTdqs` accumulated float weights and called `Math.round`: for 612 of the 7,500 score vectors whose exact sum ends in `.X5`, the float sum lands just below the tie and rounds down, and 134 of those land a tier low. The rollups had no definition at all, and the coherence mean is a tie for every odd dimension sum. Dimension scores are untouched; affected composites get a deterministic recompute with no LLM call.

Deferred until the `definitionBytes` corpus run reports: a calibration example for low-coverage-but-documented parameters, and rebasing the tool-count anchor on domain breadth rather than raw count. Both edit prompts and therefore re-score the registry, so they ship as one batch — and only if the corpus shows the rubric actually rewards length.

## v1.1 — 2026-08-23

Added **shadowing risk**: a flag for a tool whose purpose is substantially covered by a sibling that is materially cheaper to invoke.

- New context signals `requiredFieldCount`, `schemaDepth`, `unionChoiceCount`, and the derived `invocationCost` — stored on every tool, withheld from the tool-scoring prompt.
- A deterministic prefilter emits at most one candidate pair per tool; the existing server coherence call confirms genuine purpose overlap.
- New `Shadowing Risk` server flag (`serverFlags`) and `shadowingRisks` on the server record. It changes no score.
- Coherence gains a second re-run trigger: a change to the server's candidate set.

## v1.0 — 2026-06-07

Initial publication: the six-dimension tool rubric, hard gates, deterministic post-processing, four-dimension server coherence, the server rollup, and both prompts verbatim.
