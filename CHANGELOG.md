# Changelog

Notable changes to the TDQS specification. Scores are calibrated to a rubric+model pair, so each entry states what re-scores.

## v1.1 — 2026-08-23

Added **shadowing risk**: a flag for a tool whose purpose is substantially covered by a sibling that is materially cheaper to invoke.

- New context signals `requiredFieldCount`, `schemaDepth`, `unionChoiceCount`, and the derived `invocationCost` — stored on every tool, withheld from the tool-scoring prompt.
- A deterministic prefilter emits at most one candidate pair per tool; the existing server coherence call confirms genuine purpose overlap.
- New `Shadowing Risk` server flag (`serverFlags`) and `shadowingRisks` on the server record. It changes no score.
- Coherence gains a second re-run trigger: a change to the server's candidate set.

## v1.0 — 2026-06-07

Initial publication: the six-dimension tool rubric, hard gates, deterministic post-processing, four-dimension server coherence, the server rollup, and both prompts verbatim.
