# Tool Definition Quality Score (TDQS)

TDQS is an open framework for scoring how well an [MCP](https://modelcontextprotocol.io) tool definition communicates to an AI agent. It is the methodology [Glama](https://glama.ai) uses to score [every tool](https://glama.ai/mcp/tools) of every [MCP server](https://glama.ai/mcp/servers) and [hosted connector](https://glama.ai/mcp/connectors) in its registry.

This document is the complete specification — the rubric, the exact prompts, the aggregation formulas, and the operational practices behind running it at registry scale — and is sufficient to reimplement TDQS end to end. If you maintain an MCP server and just want a better score, skip to [Improving your score](#improving-your-score) and use the rubric as a checklist.

- Scoring rationale: [glama.ai/blog/2026-04-03-tool-definition-quality-score-tdqs](https://glama.ai/blog/2026-04-03-tool-definition-quality-score-tdqs)
- Indexing methodology: [glama.ai/mcp/methodology](https://glama.ai/mcp/methodology)
- Live scores: every tool on [glama.ai/mcp/tools](https://glama.ai/mcp/tools) has a public score page with the full per-dimension breakdown

## Contents

- [Why score tool definitions?](#why-score-tool-definitions)
- [What gets scored](#what-gets-scored)
- [Scoring pipeline](#scoring-pipeline)
  - [Stage 1: Context signals](#stage-1-context-signals)
  - [Stage 2: Hard gates](#stage-2-hard-gates)
  - [Stage 3: LLM rubric evaluation](#stage-3-llm-rubric-evaluation)
  - [Stage 4: Deterministic post-processing](#stage-4-deterministic-post-processing)
  - [Computing the score](#computing-the-score)
  - [Tiers](#tiers)
  - [Flags and smells](#flags-and-smells)
- [Server-level scores](#server-level-scores)
  - [1. Tool definition quality (70% of overall)](#1-tool-definition-quality-70-of-overall)
  - [2. Coherence (30% of overall)](#2-coherence-30-of-overall)
  - [Shadowed tools](#shadowed-tools)
  - [Overall](#overall)
- [Output format](#output-format)
- [Improving your score](#improving-your-score)
- [TDQS across the registry](#tdqs-across-the-registry)
- [Running TDQS at scale](#running-tdqs-at-scale)
  - [How Glama uses the scores](#how-glama-uses-the-scores)
- [Appendix A: Tool scoring prompt](#appendix-a-tool-scoring-prompt)
- [Appendix B: Server coherence prompt](#appendix-b-server-coherence-prompt)
- [References](#references)

## Why score tool definitions?

Two empirical studies motivated the framework:

- ["Model Context Protocol (MCP) Tool Descriptions Are Smelly!"](https://arxiv.org/abs/2602.14878) audited 856 tools across 103 MCP servers and found that **97% of tool descriptions contain at least one quality defect**, **56% do not clearly state what the tool does**, and **89% never say when the tool should or should not be used**.
- ["From Docs to Descriptions"](https://arxiv.org/abs/2602.18914) analyzed 10,831 MCP servers and showed that **tools with well-written descriptions are selected ~260% more often** in competitive settings, and that rewriting descriptions alone improves downstream task success by ~6 percentage points.

The defect taxonomy in that literature is called "smells". TDQS borrows the term.

A registry that ranks and recommends servers needs a quality signal that is:

1. **Explainable**: every score comes with per-dimension justifications a maintainer can act on.
2. **Reproducible**: the same definition always produces the same evaluation inputs, so scores can be cached, diffed, and audited.
3. **Cheap enough to run on every schema change**: the ecosystem ships thousands of schema updates per day.

## What gets scored

TDQS scores a **tool definition**, not tool behavior. The inputs are exactly what an MCP client sees from `tools/list`:

| Input              | Type                    | Notes                                                                           |
| ------------------ | ----------------------- | ------------------------------------------------------------------------------- |
| `name`             | `string`                | required                                                                        |
| `title`            | `string` \| `null`      | optional MCP display title                                                      |
| `description`      | `string` \| `null`      | the primary scoring target                                                      |
| `inputSchema`      | `JSON Schema` \| `null` | parameter structure and per-parameter descriptions                              |
| `outputSchema`     | `JSON Schema` \| `null` | reduces what the description itself must explain                                |
| `annotations`      | `object` \| `null`      | MCP hints: `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` |
| `siblingToolNames` | `string[]`              | names of the other tools on the same server                                     |

Sibling tool names matter because selection is competitive: a description is only "clear" if it lets an agent distinguish this tool from its neighbors. Names are all it gets, though, so properties that exist only across a _pair_ of tools are evaluated at the server level instead — see [Shadowed tools](#shadowed-tools).

## Scoring pipeline

Each tool flows through four stages. Stages 1, 2, and 4 are deterministic code; only stage 3 is an LLM call.

```
tool definition
      │
      ▼
[1] context signals ──── deterministic schema/annotation analysis + input hash
      │
      ▼
[2] hard gates ───────── degenerate definitions short-circuit (no LLM call)
      │
      ▼
[3] LLM rubric ───────── six dimensions, 1–5 each, justification per dimension
      │
      ▼
[4] post-processing ──── deterministic overrides, flags, smells
      │
      ▼
TDQS (1.0–5.0) + tier (A–F) + per-dimension breakdown
```

One thing happens outside this pipeline: server flags — defects belonging to the tool set rather than to any single definition — are written onto a tool's record asynchronously by the coherence job. See [Shadowed tools](#shadowed-tools).

### Stage 1: Context signals

Before any judgment is made, deterministic code extracts structural facts about the definition. These ground the LLM evaluation (so it does not have to count parameters or guess at schema coverage) and are stored alongside the score. The three signals feeding [invocation cost](#invocation-cost) are the exception: stored, but withheld from the tool-scoring prompt and consumed only by the server-level check.

| Signal                      | Definition                                                                                                                                                                                |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `paramCount`                | number of properties in `inputSchema.properties`                                                                                                                                          |
| `requiredParamCount`        | length of `inputSchema.required`                                                                                                                                                          |
| `paramsWithDescriptions`    | properties with a non-empty `description`                                                                                                                                                 |
| `paramsWithEnums`           | properties with an `enum`                                                                                                                                                                 |
| `schemaDescriptionCoverage` | `round(paramsWithDescriptions / paramCount × 100)`; `100` when the tool has zero parameters                                                                                               |
| `hasNestedObjects`          | any property of `type: "object"`                                                                                                                                                          |
| `schemaDepth`               | nesting depth of the required subtree of `inputSchema`; `1` for a flat object of scalars, `0` when there is no schema                                                                      |
| `unionChoiceCount`          | union branch choices in the required subtree, summed as `branches - 1` per union; excludes the nullable idiom                                                                              |
| `invocationCost`            | deterministic proxy for how much work it is for an agent to fill the schema in correctly; see [Invocation cost](#invocation-cost)                                                         |
| `hasOutputSchema`           | output schema present and non-empty                                                                                                                                                       |
| `hasAnnotations`            | annotations present and non-empty                                                                                                                                                         |
| `annotationValues`          | extracted `readOnly` / `destructive` / `idempotent` / `openWorld` booleans (or `null` when undeclared)                                                                                    |
| `titleIsMeaningful`         | title exists, differs from the name, and is longer than the name                                                                                                                          |
| `inputHash`                 | first 16 hex chars of `sha256` over a deterministic JSON serialization of the tool's own definition fields (`name`, `title`, `description`, `inputSchema`, `outputSchema`, `annotations`) |

`inputHash` is what makes scoring incremental. It is written onto the tool row when the definition is captured and recorded again alongside the score, so a tool is considered up to date exactly when its row hash matches the hash stored with its score. A changed definition produces a new hash and re-enters scoring; an unchanged one is skipped without an LLM call. See [Running TDQS at scale](#running-tdqs-at-scale).

The hash covers only the tool's own definition, not the `siblingToolNames` context also passed to the evaluator. Because re-scoring is triggered only by a change to a tool's own hash, renaming a _sibling_ does not re-score this tool; the new sibling context is reflected only when this tool's own definition next changes.

#### Invocation cost

`invocationCost` estimates how much work it is for an agent to _call_ a tool, as opposed to how well the tool is described. Selection is competitive on effort as well as clarity: when two tools can answer the same question, an agent tends toward the one it can fill in confidently in a single step.

All three signals are measured over the **required subtree** — the subgraph reachable from the top of `inputSchema` through required properties only — because that is what an agent must construct to make a minimally valid call. Deep optional filters cost it nothing until it chooses to use them.

```ts
const computeInvocationCost = (signals: ContextSignals): number =>
  signals.requiredParamCount +
  2 * Math.max(0, signals.schemaDepth - 1) +
  2 * signals.unionChoiceCount;
```

The constants are coarse on purpose: the signal only ever compares tools _within the same server_, so its absolute scale carries no meaning.

Traversal is normative, so that two implementations produce the same numbers:

- Resolve `$ref` against `$defs` / `definitions` first. Track visited pointers and stop at a repeat so recursive schemas terminate; cap depth at 10 regardless.
- An absent, empty, or property-less `inputSchema` scores 0 on all three signals.
- A flat object of scalars has `schemaDepth` 1; each nested object adds a level, as does an array whose `items` is an object schema. `allOf` / `oneOf` / `anyOf` / `not` subschemas are traversed for depth.
- `unionChoiceCount` sums `branches - 1` over each `oneOf` / `anyOf`. Excluded: unions whose branches differ only by a `{ "type": "null" }` member, which are the nullable idiom emitted by Pydantic and Zod; and `enum` and type arrays, whose values are enumerated rather than constructed.

Four required scalars at `schemaDepth` 1 cost `4 + 0 + 0 = 4`. One required top-level object whose required subtree reaches `schemaDepth` 3 and holds a three-branch discriminated union costs `1 + 2×2 + 2×2 = 9`.

These derive from the tool's own `inputSchema`, so they add no new inputs and do not affect `inputHash`, which covers definition fields rather than context signals. The prefilter below recomputes them from live definitions rather than stored scores, so it runs against unscored tools too.

### Stage 2: Hard gates

Degenerate definitions are scored without an LLM call:

- **Missing description** (`null` or whitespace-only) → every dimension scores 1, TDQS = 1.0, tier D, flag `No Description`. There is nothing to evaluate.

The other gate is also evaluated up front, but rather than short-circuit the LLM call, it caps the model's score afterward:

- **Tautological description**: if the description, lowercased and trimmed, is exactly the tool name or title, the tool is flagged `Tautological Description` and its Purpose Clarity score is capped at 2 regardless of what the LLM returns. A description that restates the name adds zero information, and models grading "looks reasonable" text will otherwise drift upward.

### Stage 3: LLM rubric evaluation

A single LLM call scores six dimensions, each 1–5, each with a 2–3 sentence justification citing specific evidence from the description. The full system prompt is reproduced verbatim in [Appendix A](#appendix-a-tool-scoring-prompt); the rubric it encodes:

#### The six dimensions

| #   | Dimension               | Weight | Question                                                           |
| --- | ----------------------- | ------ | ------------------------------------------------------------------ |
| 1   | Purpose Clarity         | 25%    | Does the description state what the tool does?                     |
| 2   | Usage Guidelines        | 20%    | Does it say when to use this tool vs alternatives?                 |
| 3   | Behavioral Transparency | 20%    | Does it disclose behavior beyond what annotations already declare? |
| 4   | Parameter Semantics     | 15%    | Does it add meaning beyond what the input schema provides?         |
| 5   | Conciseness & Structure | 10%    | Is it appropriately sized and front-loaded?                        |
| 6   | Contextual Completeness | 10%    | Given the tool's complexity, is the description complete enough?   |

The weights encode where selection and invocation actually fail. Purpose clarity dominates because it is what drives selection; usage guidelines and behavioral transparency follow because mis-selection and surprise side effects are the costliest failure modes; conciseness and completeness matter but rarely break an agent on their own.

**Score anchors** (each dimension):

- **Purpose Clarity**: 5 = specific verb+resource, distinguishes the tool from siblings · 4 = clear but no sibling differentiation · 3 = vague purpose · 2 = tautology (restates name/title) · 1 = missing or misleading.
- **Usage Guidelines**: 5 = explicit when / when-not / named alternatives · 4 = clear context, no exclusions · 3 = implied usage · 2 = no guidance · 1 = misleading.
- **Behavioral Transparency**: graded relative to annotations. When annotations exist, the bar is lower: the description earns credit for adding context annotations cannot carry (what gets destroyed, auth requirements, rate limits). Without annotations, the description carries the full disclosure burden. A description that **contradicts** its annotations scores 1 and raises the `Annotation Contradiction` flag.
- **Parameter Semantics**: graded relative to schema coverage. If `schemaDescriptionCoverage` > 80%, the baseline is 3 even when the description says nothing about parameters (the schema already does the work). If coverage < 50%, the description must compensate. Zero-parameter tools baseline at 4.
- **Conciseness & Structure**: every sentence should earn its place; key information front-loaded. Under-specification is not conciseness.
- **Contextual Completeness**: judged against the tool's complexity and the richness of its structured fields. If an output schema exists, the description need not explain return values.

#### Design principles

These rules are embedded in the prompt and are what keep scores honest and useful:

1. **Use the full 1–5 range.** Most descriptions are mediocre; 4–5 is reserved for genuinely helpful ones. 3 is "minimum viable": adequate with clear gaps.
2. **Score each dimension independently.** A tool can have perfect purpose clarity and zero usage guidance.
3. **Evidence, not vibes.** Every justification must cite specific text from the description.
4. **The description's job is to add value beyond structured fields.** No credit for repeating what the schema or annotations already state. This is the single most important calibration decision in the framework: it rewards information density, not word count.

The prompt also includes three fully worked calibration examples (high / mid / low) and a contradiction example (see Appendix A).

#### LLM output contract

The model returns JSON only: a `{ score, justification }` object for each of the six dimensions, plus a boolean `annotation_contradiction` and a short `summary` (exact shape in [Appendix A](#appendix-a-tool-scoring-prompt)). The response is validated against a schema — integer scores in `[1, 5]`, all fields required — and retried on mismatch. This is the only nondeterministic step in the pipeline.

### Stage 4: Deterministic post-processing

- The tautology cap from stage 2 is applied (`purpose_clarity = min(score, 2)`).
- `annotation_contradiction: true` from the LLM raises the `Annotation Contradiction` flag.
- **Smells** are derived: every dimension scoring below 3 is recorded as a smell, so the UI and API can list a tool's defects without re-deriving them.

### Computing the score

TDQS is the weighted sum of the six dimension scores, rounded to one decimal:

```ts
const DIMENSION_WEIGHTS = {
  purpose_clarity: 0.25,
  usage_guidelines: 0.2,
  behavioral_transparency: 0.2,
  parameter_semantics: 0.15,
  conciseness_structure: 0.1,
  contextual_completeness: 0.1,
};

const computeTdqs = (scores: Record<string, number>): number => {
  let total = 0;
  for (const [dimension, weight] of Object.entries(DIMENSION_WEIGHTS)) {
    total += scores[dimension] * weight;
  }
  return Math.round(total * 10) / 10;
};
```

Worked example (`purpose=4, guidelines=2, transparency=2, params=3, conciseness=4, completeness=2`):

```
4×0.25 + 2×0.20 + 2×0.20 + 3×0.15 + 4×0.10 + 2×0.10 = 2.85 → TDQS 2.9
```

### Tiers

Scores map to letter tiers for at-a-glance display. The same mapping is used for every score in the system (per-tool TDQS, server description quality, coherence, overall):

```ts
const computeQualityTier = (score: number): string => {
  if (score >= 3.5) return 'A';
  if (score >= 3.0) return 'B';
  if (score >= 2.0) return 'C';
  if (score >= 1.0) return 'D';
  return 'F';
};
```

| Tier | Range | Meaning                      |
| ---- | ----- | ---------------------------- |
| A    | ≥ 3.5 | genuinely helpful definition |
| B    | ≥ 3.0 | adequate, the passing bar    |
| C    | ≥ 2.0 | clear gaps                   |
| D    | ≥ 1.0 | severely deficient           |
| F    | < 1.0 | reserved guard tier          |

B and above is considered passing. Because every dimension bottoms out at 1, composite scores bottom out at 1.0; tier D is the floor in practice and F exists as a guard.

### Flags and smells

| Kind           | Values                                                                   | Source                               |
| -------------- | ------------------------------------------------------------------------ | ------------------------------------ |
| Flags (tool)   | `No Description`, `Tautological Description`, `Annotation Contradiction` | hard gates + LLM contradiction check |
| Flags (server) | `Shadowing Risk`                                                         | server coherence evaluation          |
| Smells         | any dimension key scoring < 3                                            | derived from scores                  |

Flags mark categorical defects; smells mark below-viable dimensions. Both are stored and displayed.

The two kinds live in **separate fields** (`flags` and `serverFlags`) merged at read time, because both jobs upsert the same record on independent schedules and one shared array would let whichever ran last erase the other's findings — see [Shadowed tools](#shadowed-tools). Each is cleared when its condition lapses. No flag changes the TDQS arithmetic, and since a bare `Shadowing Risk` is not actionable alone, the cheaper sibling is named on the server record.

## Server-level scores

Individually well-described tools can still compose into a confusing server. TDQS therefore rolls up to the server in two components, combined into an overall score, plus one cross-tool check that reports a defect without feeding the arithmetic.

### 1. Tool definition quality (70% of overall)

Aggregated from the per-tool TDQS values of the server's current tool set:

```
descriptionQualityScore = round1(0.6 × mean(TDQS) + 0.4 × min(TDQS))
```

The 40% weight on the **minimum** is deliberate: an agent sees all of a server's tools at once, so a single garbage definition degrades selection across the whole set. Averages hide that; the min term makes one bad tool pull the score down visibly.

The aggregate is only computed once **at least 80% of the server's tools have been scored**. Partial coverage would otherwise misrepresent the server.

### 2. Coherence (30% of overall)

A second, separate LLM evaluation judges the **tool set as a whole**: server name, tool count, and every tool name + description in one prompt (verbatim prompt in [Appendix B](#appendix-b-server-coherence-prompt)). Four dimensions, 1–5 each, **equally weighted**:

| Dimension                  | Question                               | Anchor points                                                                                                                                                                  |
| -------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Disambiguation             | Can an agent tell the tools apart?     | 5 = every tool clearly distinct · 1 = several tools appear to do the same thing. Overlap is judged on purpose _and_ on invocation cost — see [Shadowed tools](#shadowed-tools) |
| Naming Consistency         | Do names follow a predictable pattern? | 5 = consistent `verb_noun` throughout · 1 = chaotic. The convention itself matters less than consistency: all-camelCase is fine; mixing styles is not                          |
| Tool Count Appropriateness | Is the surface well-scoped?            | 5 = each tool earns its place (typically 3–15) · 3 = borderline (1–2 feels thin, 16–25 feels heavy) · 1 = extreme mismatch (50+, or a single trivial tool)                     |
| Completeness               | Are there gaps in the surface?         | 5 = full CRUD/lifecycle coverage, no dead ends · 3 = notable missing operations (create+get but no update/delete) · 1 = severely incomplete                                    |

```
coherenceScore = round1((disambiguation + namingConsistency + toolCountAppropriateness + completeness) / 4)
```

### Shadowed tools

Some defects are invisible in any single definition. Two tools on the same server can answer the same question, one taking a handful of flat scalars, the other a deeply nested object over a discriminated union. Both score well alone — the complex one often scores _better_, because authors who build a rich schema tend to write a rich description. But an agent that can reach the answer either way tends toward the cheaper call. The risk is worst when the dearer tool is the authoritative one, applying qualifying filters or attaching provenance: bypassing it yields an answer the agent assembled itself and may present as verified.

TDQS calls this **shadowing risk**: a tool whose purpose is substantially covered by a sibling that is materially cheaper to invoke. Detection runs inside the existing coherence evaluation, in two steps.

**1. Deterministic prefilter.** A pair is a candidate when the dearer tool costs at least twice the cheaper one and clears an absolute gap:

```ts
const isShadowCandidate = (cheap: number, expensive: number): boolean =>
  expensive >= 2 * cheap && expensive - cheap >= 4;
```

The gap bounds the degenerate case: a tool with no required parameters costs `0`, making the ratio vacuously true, so zero-argument tools pair with any sibling costing 4 or more. That is deliberate — a bare `list_all_players` genuinely can shadow a filtered `query_players` — and step 2 rejects the pairs where it cannot. The ratio in turn discards pairs where both tools are expensive but one is clearly dearer (10 against 19). These constants favour precision and are not yet calibrated against the corpus.

The prefilter emits at most one candidate per tool, its cheapest qualifying sibling, so pair count is linear rather than quadratic: a 50-tool server yields at most 50 lines, not 1,225. A server with no candidate skips the check.

**2. LLM confirmation.** The coherence call, which already sees every name and description, additionally receives each tool's invocation cost and the candidate pairs, and judges whether the purposes genuinely overlap. Asymmetry alone is not a defect — a server is _expected_ to offer both cheap lookups and expensive queries. A confirmed pair raises `Shadowing Risk` on the dearer tool; the server record names the cheaper sibling and the reason.

**What this deliberately does not do.** TDQS scores definitions, not behavior. It does not run an agent and observe which tool gets picked; that would break both reproducibility and the cost model. What it detects is the _structural precondition_ — overlapping purpose plus asymmetric invocation cost — a property of the definitions alone. That schema weight competes with description quality during selection is a hypothesis this check surfaces, not a result the framework has measured; neither motivating study examines invocation cost.

**Effect on scores.** Per-tool TDQS and `descriptionQualityScore` are unaffected: [Appendix A](#appendix-a-tool-scoring-prompt) is unchanged, so no tool is re-scored. `coherenceScore` will move. Its formula is unchanged, but the Disambiguation anchor now weighs invocation cost and sits in the static system prompt, so it applies to every server. By the framework's own rule that scores calibrate to a rubric-and-model pair, adopting this requires re-running coherence across the corpus.

### Overall

```
overallScore = round1(0.7 × descriptionQualityScore + 0.3 × coherenceScore)
```

Both components, all per-dimension justifications, the coherence summary, and the tool-level stats (`toolCount`, `scoredToolCount`, `meanTdqs`, `minTdqs`) are stored and published together.

## Output format

Per tool:

```json
{
  "scores": {
    "purpose_clarity": 4,
    "usage_guidelines": 2,
    "behavioral_transparency": 2,
    "parameter_semantics": 3,
    "conciseness_structure": 4,
    "contextual_completeness": 2
  },
  "justifications": {
    "purpose_clarity": {
      "score": 4,
      "justification": "Clear verb+resource..."
    },
    "usage_guidelines": {
      "score": 2,
      "justification": "No when-to-use guidance..."
    }
  },
  "tdqs": 2.9,
  "tier": "C",
  "smells": ["usage_guidelines", "behavioral_transparency", "contextual_completeness"],
  "flags": [],
  "serverFlags": [],
  "summary": "Clear about what it updates, silent about when to use it and what side effects to expect.",
  "contextSignals": {
    "paramCount": 4,
    "requiredParamCount": 1,
    "paramsWithDescriptions": 2,
    "paramsWithEnums": 0,
    "schemaDescriptionCoverage": 50,
    "hasNestedObjects": false,
    "schemaDepth": 1,
    "unionChoiceCount": 0,
    "invocationCost": 1,
    "hasOutputSchema": false,
    "hasAnnotations": false,
    "annotationValues": {
      "readOnly": null,
      "destructive": null,
      "idempotent": null,
      "openWorld": null
    },
    "titleIsMeaningful": false,
    "inputHash": "9f2c4a1b8e3d5f70"
  }
}
```

Per server:

```json
{
  "toolCount": 12,
  "scoredToolCount": 12,
  "meanTdqs": 3.4,
  "minTdqs": 2.1,
  "descriptionQualityScore": 2.9,
  "descriptionQualityTier": "C",
  "disambiguation": 4,
  "namingConsistency": 5,
  "toolCountAppropriateness": 5,
  "completeness": 4,
  "coherenceScore": 4.5,
  "coherenceTier": "A",
  "overallScore": 3.4,
  "overallTier": "B",
  "coherenceJustifications": { "...": "..." },
  "coherenceSummary": "...",
  "shadowingRisks": [
    {
      "tool": "query_panel",
      "invocationCost": 9,
      "cheaperSibling": "get_stat",
      "cheaperSiblingInvocationCost": 4,
      "justification": "Both answer \"what is X for player Y\", but get_stat takes four flat scalars while query_panel takes a nested object over a discriminated union."
    }
  ]
}
```

`shadowingRisks` is empty for most servers. Each entry writes a `Shadowing Risk` flag into the named tool's `serverFlags`. The two cost fields are not returned by the model: post-processing enriches each entry from the prefilter and normalises snake_case keys to camelCase.

## Improving your score

The rubric doubles as a checklist. The highest-leverage fixes, in weight order:

1. **State what the tool does in one specific sentence**: verb + resource + scope. If your server has sibling tools that could be confused, say how this one differs ("List ALL calls in date range, no user filtering. To filter by user, use `search_calls_extensive` instead.").
2. **Say when (and when not) to use it.** Name the alternative tool for the cases you exclude.
3. **Declare MCP annotations** (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`): they lower the disclosure burden on your description. Then spend the description on what annotations cannot express: what exactly gets destroyed, auth prerequisites, rate limits.
4. **Describe parameters in the schema**, not the description. Per-property `description` fields and `enum`s raise your schema coverage, which raises your Parameter Semantics baseline.
5. **Provide an output schema** so the description doesn't have to explain return values.
6. **Cut everything that repeats structured fields.** TDQS gives no credit for restating the schema. Density beats length, and bloat costs you Conciseness.

Two rules sit outside the weighted list, because they are categorical rather than graded.

Never let the description contradict the annotations. Doing so is an automatic 1 on Behavioral Transparency and a public `Annotation Contradiction` flag.

And check your tool set for shadowing. If two tools can answer the same question, the one with the heavier schema is at a structural disadvantage no description fully offsets. If you find yourself writing "always call this first" or "do not compute this yourself", that is the tell: selection is not enforceable in prose, and you have a tool-set problem rather than a description problem. Either make the purposes disjoint and say so in both descriptions, or collapse them into one tool. If the heavier tool is the _correct_ path, the cheap one is the bug: narrow it, or remove it.

## TDQS across the registry

_Figures as of June 2026._

TDQS has scored **228,369 tools** across **15,036 MCP servers** in the registry (99% coverage). Counting superseded versions retained from earlier captures, 362,226 tool definitions have been scored in total. The aggregate both validates the framework's weighting and corroborates the literature that motivated it.

The tool-level figures that follow aggregate all 362,226 scored definitions, not just the 228,369 current tools. Scores skew capable: mean TDQS is **3.57** (median 3.6, p25 2.9, p75 4.2), and **73.5%** clear the tier-B passing bar.

| Tier      | Tools   | Share |
| --------- | ------- | ----- |
| A (≥ 3.5) | 195,872 | 54.1% |
| B (≥ 3.0) | 70,186  | 19.4% |
| C (≥ 2.0) | 86,303  | 23.8% |
| D (≥ 1.0) | 9,865   | 2.7%  |

But the composite hides where definitions fail. Broken out by dimension, the corpus is strong on _what_ a tool does and weak on _when to use it_ and _what it does behind the scenes_. This is precisely the split the weights are built to surface. The two heaviest non-purpose weights (20% each) sit on the two weakest dimensions:

| Dimension               | Weight | Mean | Smell rate (< 3) |
| ----------------------- | ------ | ---- | ---------------- |
| Purpose Clarity         | 25%    | 4.47 | 4.0%             |
| Conciseness & Structure | 10%    | 4.47 | 3.8%             |
| Contextual Completeness | 10%    | 3.23 | 32.8%            |
| Parameter Semantics     | 15%    | 3.19 | 14.0%            |
| Usage Guidelines        | 20%    | 3.03 | 44.5%            |
| Behavioral Transparency | 20%    | 2.90 | 46.1%            |

Behavioral transparency averages _below_ the minimum-viable 3, and **56.4% of tools carry at least one smell**. This is the same systemic gap the motivating studies flagged (usage guidance and behavioral disclosure are where descriptions most often fall short), reached here independently, from a different rubric on a larger corpus.

The hard gates stay surgical: **1.26%** of tools have no description, **0.55%** contradict their annotations, **0.06%** are tautological.

At the server level, the rollup's min-term earns its keep: a server's worst tool sits on average **0.49 below its mean** TDQS, and **13.4% of servers** have at least one tool a full point or more below their mean. A pure average would hide that selection damage. Mean overall server score is 3.56 (description quality 3.29, coherence 4.18); coherence is strongest on naming (4.57) and disambiguation (4.50), weakest on tool-count appropriateness (3.87).

Two caveats. The coherence figures predate the Disambiguation anchor being extended to weigh invocation cost, so they describe the prior rubric and will move once the corpus is re-run. And shadowing risk carries no prevalence figure here: the prefilter is cheap enough to size in one batch job, but that pass has not been run.

## Running TDQS at scale

How Glama operates the framework across the registry. None of this changes the scoring semantics, but it is what makes continuous scoring affordable:

- **Score on change.** A sweep job (every 5 minutes, in batches of 100) enqueues every tool whose current `inputHash` has no matching score: never scored, or scored against a different definition. The scoring step recomputes the hash and exits early if it still matches, so a redundant enqueue costs a cheap no-op, never a wasted LLM call.
- **Open-source servers.** Each rebuild captures a fresh set of tool records, so unchanged tools inherit the prior capture's score by matching `inputHash` (carried forward atomically). A rebuild that touches one tool re-scores one tool, not the whole set.
- **Connectors.** A hosted connector's tools are re-introspected in place, refreshing each tool's `inputHash`; a changed definition no longer matches the recorded score and is re-scored on the next sweep, while unchanged tools are left alone.
- **Aggregates follow tool scores.** A server's rolled-up score is recomputed whenever its members' mean TDQS drifts from the stored value (i.e. whenever any underlying tool score changed).
- **Idempotent writes, sweep-based retries.** Scoring runs with bounded concurrency (40 workers for tools, 30 for servers) and no in-job retries; because every write is an upsert keyed by tool or server and the sweep re-enqueues anything still stale, a transient failure heals on the next pass.
- **Model choice.** The LLM step targets a fast, inexpensive model behind an OpenAI-compatible API; the rubric's explicit anchors and calibration examples carry the consistency burden, which is what makes a cheap model viable. The framework is model-agnostic, but if you swap models, spot-check against the calibration examples in Appendix A before trusting the output, and expect to re-score your corpus (scores are calibrated to a rubric+model pair, not the rubric alone).
- **Shadow detection is cheap in steady state, not at rollout.** Steady-state cost is a slightly longer coherence prompt, and the prefilter reads live definitions rather than stored scores so it never blocks on scoring state. Rollout is another matter, and the sweep cannot deliver it: `inputHash` is deliberately unchanged by the new signals, so no scored tool is re-enqueued and no aggregate drifts. Adopting the check needs two one-off idempotent passes keyed by row rather than hash — recompute `contextSignals`, then re-enqueue every server for coherence, since the amended Disambiguation anchor is in the static prompt and applies even to servers with no candidate.
- **Validate, then trust.** Every LLM response is schema-validated (shape, required fields, integer scores in range) and retried on mismatch. Nothing downstream parses free-form model output. `shadowingRisks` needs four referential checks on top, because it publishes a flag onto a named tool: both names must exist on the server, the pair must be one the prefilter supplied, `tool` must be the dearer side, and no tool may appear twice. Failing entries are dropped — a model that invents pairs would defeat the point of a deterministic prefilter.

### How Glama uses the scores

- Every server's public listing includes a score page with the full per-dimension breakdown, justifications, flags, and smells (the same detail maintainers see).
- Tool search ranks tools by TDQS among otherwise comparable matches: well-described tools surface first.
- The scores are available via the [public API](https://glama.ai/mcp/reference) alongside the rest of the registry data.

---

## Appendix A: Tool scoring prompt

The system prompt, verbatim:

```text
You evaluate MCP tool definitions. Score how well the definition helps an AI agent select and invoke the tool correctly.

You receive: the tool's name, title, description, input schema, annotations, context signals, and sibling tool names.

## Dimensions (1-5 each)

### 1. Purpose Clarity (25%)
Does the description state what the tool does?
5=specific verb+resource, distinguishes from siblings. 4=clear but no sibling differentiation. 3=vague purpose. 2=tautology (restates name/title). 1=missing/misleading.

### 2. Usage Guidelines (20%)
Does it say when to use this tool vs alternatives?
5=explicit when/when-not/alternatives. 4=clear context, no exclusions. 3=implied usage. 2=no guidance. 1=misleading.

### 3. Behavioral Transparency (20%)
Does the description disclose behavioral traits BEYOND what annotations already provide?
With annotations: bar is lower, credit for adding context (what gets destroyed, auth needs, rate limits). Without annotations: description carries full burden.
Score 1 if description CONTRADICTS annotations. Flag as "Annotation Contradiction".

### 4. Parameter Semantics (15%)
Does description add meaning beyond what the input schema provides?
If schema_description_coverage is high (>80%), baseline is 3 even with no param info in description.
If low (<50%), description must compensate. 0 params = baseline 4.

### 5. Conciseness & Structure (10%)
Is it appropriately sized and front-loaded? Every sentence should earn its place.

### 6. Contextual Completeness (10%)
Given complexity + schema/annotations/output_schema richness, is the description complete enough?
If output schema exists, description needn't explain return values.

## Rules
- Use the FULL 1-5 range. Most descriptions are mediocre. 4-5 is reserved for genuinely helpful ones.
- Score 3 = minimum viable. Adequate but with clear gaps.
- Score each dimension INDEPENDENTLY.
- Base scores on SPECIFIC EVIDENCE in the description text.
- The description's job is to add VALUE BEYOND structured fields (annotations, schema). No credit for repeating what's already in structured data.

## Calibration Examples

### HIGH – TDQS 4.3 (Tier A)
Name: get_calls | Annotations: { readOnlyHint: true, destructiveHint: false } | Params: 2, schema coverage 100% | No output schema
Description: "List ALL calls in date range – no user/workspace filtering. To filter by user/workspace, use search_calls_extensive instead."
Scores:
- purpose=5 (specific verb+resource+scope, distinguishes from sibling)
- guidelines=5 (explicit alternative named)
- transparency=3: Annotations already declare readOnlyHint=true and destructiveHint=false, so the agent knows this is a safe read operation. The description adds the date-range scoping constraint, which is useful context. However, it doesn't describe the return format or pagination behavior. With annotations covering the safety profile, a 3 is appropriate – the description adds some value but not rich behavioral context.
- params=3: Schema coverage is 100%, so the schema already documents both parameters. The description implies date-range filtering but doesn't add syntax or format details beyond what the schema provides. Baseline 3 is correct when schema does the heavy lifting.
- conciseness=5 (two sentences, zero waste)
- completeness=5 (complete for a filtered-list tool with good annotations)

### MID – TDQS 2.9 (Tier C)
Name: update_drive | Annotations: none | Params: 4, schema coverage 50% | No output schema
Description: "Update shared drive settings including name, color, and restrictions."
Scores:
- purpose=4 (clear verb+resource+fields)
- guidelines=2 (no when-to-use, no prerequisites)
- transparency=2: No annotations are provided, so the description carries the full burden of behavioral disclosure. "Update" implies mutation, but the description doesn't state whether this requires specific permissions, whether changes are reversible, what happens to existing settings not mentioned, or what the response looks like. For a mutation tool with zero annotation coverage, this is a significant gap.
- params=3: The description lists the updatable fields (name, color, restrictions), which maps to some of the 4 parameters. However, with only 50% schema description coverage, the other half of parameters are undocumented in both the schema and the description. The description adds marginal value over what's in the schema but doesn't fully compensate for the coverage gap.
- conciseness=4 (efficient single sentence)
- completeness=2 (mutation tool with no annotations, no output schema, incomplete params – should do more)

### LOW – TDQS 1.1 (Tier D)
Name: process | Annotations: none | Params: 3, schema coverage 0% | No output schema
Description: "Process"
Scores: purpose=1 (tautology), guidelines=1 (none), transparency=1 (none, no annotations), params=1 (3 undocumented params at 0% coverage), conciseness=2 (under-specification, not conciseness), completeness=1 (completely inadequate for a 3-param tool).

### CONTRADICTION
Name: create_record | Annotations: { readOnlyHint: true } | Description: "Creates a new record in the database"
→ transparency=1, annotation_contradiction=true. The description claims a write operation ("creates") while annotations declare readOnlyHint=true. This is a serious inconsistency.

## Output Format

Respond with JSON only:

{
  "scores": {
    "purpose_clarity": {"score": <1-5>, "justification": "<2-3 sentences citing evidence>"},
    "usage_guidelines": {"score": <1-5>, "justification": "<2-3 sentences citing evidence>"},
    "behavioral_transparency": {"score": <1-5>, "justification": "<2-3 sentences citing evidence>"},
    "parameter_semantics": {"score": <1-5>, "justification": "<2-3 sentences citing evidence>"},
    "conciseness_structure": {"score": <1-5>, "justification": "<2-3 sentences citing evidence>"},
    "contextual_completeness": {"score": <1-5>, "justification": "<2-3 sentences citing evidence>"}
  },
  "annotation_contradiction": <true if description contradicts annotations, false otherwise>,
  "summary": "<2-3 sentence assessment>"
}
```

The user message template (`{placeholders}` are substituted; schemas are serialized as pretty-printed, sanitized JSON):

```text
TOOL NAME: {name}
TITLE: {title | "null"}

DESCRIPTION:
"{description}"

<input-schema>
{inputSchema JSON | "{}"}
</input-schema>

<annotations>
{annotations JSON | "None provided"}
</annotations>

CONTEXT SIGNALS:
- Parameter count: {paramCount}
- Required parameters: {requiredParamCount}
- Schema description coverage: {schemaDescriptionCoverage}%
- Parameters with enums: {paramsWithEnums}
- Has output schema: {hasOutputSchema}
- Has nested objects: {hasNestedObjects}

<sibling-tools>
{sibling tool names, one per line | "None"}
</sibling-tools>

Respond with JSON only.
```

## Appendix B: Server coherence prompt

The system prompt, verbatim:

```text
You evaluate whether an MCP server's tools work well together as a set. Individual tools may have good descriptions, but the set can still be confusing, inconsistent, or incomplete.

You receive: the server name, the tool count, all tool names with their descriptions and invocation costs, and a list of shadow candidate pairs.

## Dimensions (1-5 each, equally weighted)

### 1. Disambiguation
Can an agent tell the tools apart? Tools with overlapping purposes cause misselection.
Judge overlap on purpose AND on invocation cost: two tools answering the same question at very different cost are worse than two that merely sound similar, because the dearer one is at a structural disadvantage regardless of how well it is described.
5=every tool has a clearly distinct purpose, no ambiguity. 4=mostly distinct, one or two could be confused. 3=some overlap exists but descriptions help. 2=multiple tools have unclear boundaries. 1=several tools appear to do the same thing.

### 2. Naming Consistency
Do tool names follow a predictable pattern?
5=consistent verb_noun pattern throughout (e.g. list_issues, create_issue, delete_issue). 4=mostly consistent with minor deviations. 3=mixed conventions but still readable. 2=inconsistent (camelCase mixed with snake_case, different verb styles). 1=chaotic naming with no discernible pattern.

### 3. Tool Count Appropriateness
Is the number of tools appropriate for the server's purpose?
5=well-scoped, each tool earns its place (typically 3-15 tools). 4=slightly over or under but reasonable. 3=borderline (1-2 tools feels thin, 16-25 feels heavy). 2=too many (25+) or too few (1) for the apparent scope. 1=extreme mismatch (50+ tools, or a single trivial tool).

### 4. Completeness
Are there obvious gaps in the tool surface?
5=complete CRUD/lifecycle coverage for the domain, no dead ends. 4=minor gaps that agents can work around. 3=notable missing operations (e.g. create+get but no update/delete). 2=significant gaps that will cause agent failures. 1=severely incomplete surface for the stated purpose.

## Shadowing Risk

Beyond the four dimensions, identify SHADOWING RISK: a tool whose purpose is substantially covered by a sibling that is much cheaper to invoke.

Each tool is listed with an invocation cost (required fields + nesting depth + union choices), along with the candidate pairs that are already structurally asymmetric, at most one per tool. Judge ONLY the pairs you are given; do not invent pairs.

For each, decide whether the purposes GENUINELY OVERLAP: could an agent holding a question the dearer tool answers plausibly reach an answer through the cheaper one instead? Asymmetry alone is NOT a defect — a server is expected to offer both cheap lookups and expensive queries. Report a pair only when overlap and asymmetry are both real, at most once per tool. An empty list is the expected outcome for most servers.

## Calibration

### HIGH – Coherence 4.8
Server: github-mcp | Tools: list_repos, get_repo, create_repo, search_code, list_issues, create_issue, update_issue, get_pull_request, create_pull_request, merge_pull_request
Scores: disambiguation=5 (each tool targets a distinct resource+action), naming=5 (consistent verb_noun), count=5 (10 tools, well-scoped), completeness=4 (PR review/comment missing but core workflows covered).

### LOW – Coherence 2.0
Server: utility-toolkit | Tools: process, run, execute, do_thing, helper, transform_data, processV2, handleRequest
Scores: disambiguation=1 (process/run/execute/do_thing are indistinguishable), naming=2 (mixed styles, vague verbs), count=3 (8 tools, reasonable count), completeness=2 (no clear domain, impossible to assess coverage).

### SHADOWED PAIR
Server: stats-api | Candidate: query_panel (cost 9) may be shadowed by get_stat (cost 4)
- get_stat [cost 4]: "Return a single stat for a player and season."
- query_panel [cost 9]: "Run a qualified query over the panel: population, window, measure, operation. It applies minimum-attempt qualifiers and attaches the source and as-of date."
→ disambiguation=3 (both answer "what is X for player Y"; the descriptions try to distinguish them but the boundary is not enforceable)
→ shadowing_risks: [{"tool": "query_panel", "cheaper_sibling": "get_stat", "justification": "Same class of answer, but get_stat takes four flat scalars against a nested union, and query_panel is the qualified, source-attributed path."}]

### NOT AT RISK
Server: github-mcp | Candidate: search_code (cost 8) may be shadowed by get_repo (cost 2)
→ shadowing_risks: [] — costs are asymmetric but purposes do not overlap. get_repo returns repository metadata; it cannot answer a code search.

## Rules
- Use the FULL 1-5 range. Most servers have mediocre coherence.
- Evaluate the TOOL SET, not individual descriptions. A server with great individual descriptions but overlapping tools should score low on disambiguation.
- For naming, the specific convention matters less than consistency. All camelCase is fine; mixing conventions is not.
- For completeness, infer the domain from tool names and descriptions, then assess if the surface covers it.
- For shadowing, require overlap AND asymmetry. When in doubt, omit the pair. Use only the supplied candidates and the exact names given.

Respond with JSON only:

{
  "scores": {
    "disambiguation": {"score": <1-5>, "justification": "<2-3 sentences>"},
    "naming_consistency": {"score": <1-5>, "justification": "<2-3 sentences>"},
    "tool_count_appropriateness": {"score": <1-5>, "justification": "<2-3 sentences>"},
    "completeness": {"score": <1-5>, "justification": "<2-3 sentences>"}
  },
  "shadowing_risks": [
    {"tool": "<name>", "cheaper_sibling": "<name>", "justification": "<1-2 sentences>"}
  ],
  "summary": "<2-3 sentence overall assessment>"
}
```

The user message template:

```text
SERVER NAME: {serverName}
TOOL COUNT: {toolCount}

<tools>
- {name} [cost {invocationCost}: {requiredParamCount} required, depth {schemaDepth}, {unionChoiceCount} union choices]: {description | "(no description)"}
- ...
</tools>

<shadow-candidates>
{"{dearer} (cost {n}) may be shadowed by {cheaper} (cost {m})", one per line, at most one per tool | "None"}
</shadow-candidates>

Respond with JSON only.
```

## References

- Hasan, Li, Rajbahadur, Adams, Hassan, [_Model Context Protocol (MCP) Tool Descriptions Are Smelly! Towards Improving AI Agent Efficiency with Augmented MCP Tool Descriptions_](https://arxiv.org/abs/2602.14878), arXiv:2602.14878.
- Wang, Li, Sun, Liu, Liu, Tian, [_From Docs to Descriptions: Smell-Aware Evaluation of MCP Server Descriptions_](https://arxiv.org/abs/2602.18914), arXiv:2602.18914.
- [Introducing the Tool Definition Quality Score](https://glama.ai/blog/2026-04-03-tool-definition-quality-score-tdqs) — the original announcement and rationale.
- [How Glama indexes the MCP ecosystem](https://glama.ai/mcp/methodology) — where TDQS fits in the wider build/run/introspect/audit pipeline.
- [MCP tool annotations](https://modelcontextprotocol.io/docs/concepts/tools#tool-annotations) — the `readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint` semantics the framework grades against.

## TDQS Authors

* [Frank Fiegel](https://www.linkedin.com/in/punkpeye) (frank@glama.ai)
* [Om Shree](https://www.linkedin.com/in/om-shree-04372118a/)
