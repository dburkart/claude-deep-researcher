---
name: research-search
description: >
  Searcher-agent operations for the Deep Researcher Reflect Evolve system:
  generate the next search query for an active research session, or answer
  a query using the Candidate Crossover algorithm (3 parallel candidates
  with diverse lenses, merged into one consolidated answer that appends to
  the global research context). Use for one-off search operations outside
  the full /deep-research loop, or when the user invokes /research-search.
  Args: <mode> [<topic-slug>] [<query>]. Modes: query, answer.
---

# Research Search — Searcher Agent with Candidate Crossover

This skill executes a single phase of the Searcher role from the Deep
Researcher Reflect Evolve architecture. Same per-topic working directory
as `/deep-research`:

```
.research/<slug>/
├── plan.md
├── context.md
└── progress.json
```

## Argument parsing

First arg is the **mode**. Second arg is an optional topic slug. For
mode `answer`, the third arg is an explicit query (otherwise prompt for
one or take the most recent query from `context.md`).

If no slug is given and exactly one `.research/*` directory exists, use
it. If multiple, ask the user which.

## Modes

### `query [<slug>]`

Generate the next-best search query for the active session.

Spawn a fresh `general-purpose` subagent. Brief:

> You are the Searcher in query-generation mode. Read `<plan_path>` and
> `<context_path>`. Identify the single highest-value unanswered
> sub-question — one whose answer would most increase research progress
> AND is **not redundant** with any query already logged in
> `<context_path>`. Output exactly one line to stdout: a concise
> web-search query string (no quotes, no markdown). Return only the query.

Show the returned query to the user.

### `answer [<slug>] <query>`

Answer a search query using the **Candidate Crossover** algorithm.

#### Step A — Spawn 3 candidates in parallel

In a single message, issue three Agent tool calls (subagent_type:
`general-purpose`). Since Claude Code does not expose per-call
`temperature` / `top_k`, diversity comes from **lenses** instead — this
preserves the paper's intent of exploring a larger search space:

| Candidate | Lens     | Optimization                                                                  |
|-----------|----------|--------------------------------------------------------------------------------|
| A         | Breadth  | Widest set of relevant facts, numbers, sources. Coverage > depth.              |
| B         | Depth    | Mechanism, causation, technical detail on the central aspect. Insight > breadth. |
| C         | Skeptic  | Contradicting evidence, edge cases, recency, source quality. Fact-check.       |

Each candidate's brief (substitute the lens row):

> You are Candidate `<A|B|C>` of the Search Agent.
> Query: `<query>`.
> Lens: `<lens optimization>`.
> Use the `WebSearch` tool to gather current information. Filter
> results: keep only sources you would rate ≥30% relevance — drop the
> rest. Return a concise answer (≤400 words) preserving every fact,
> number, date, and source URL you found relevant. Do NOT write to any
> file. Output only your answer text.

#### Step B — Crossover synthesis

After all three candidates return, **fork yourself** (Agent without
subagent_type so it inherits this conversation's context). Brief:

> You are the Crossover synthesizer for query `<query>`. Below are
> three candidate answers. Merge them into a single consolidated answer
> that retains every distinct fact, number, date, and URL —
> deduplicated, with contradictions explicitly flagged (`⚠ conflict:`).
> Then append exactly this block to `<context_path>`:
>
> ```
> ## Iteration <N> — <query>
>
> <consolidated answer>
>
> **Sources:** <bulleted list of unique URLs>
>
> ---
> ```
>
> Use the next iteration number — read existing iteration headers in
> `<context_path>` and increment. Return when the append is complete.
>
> Candidate A (Breadth):
> <text>
>
> Candidate B (Depth):
> <text>
>
> Candidate C (Skeptic):
> <text>

#### Step C — Report back

Tell the user the query has been answered and appended to
`<context_path>`. Optionally show the consolidated answer text inline
if the user is reviewing iteratively.

## Notes on faithfulness to the paper

- The paper uses **n=3** candidates with varied `temperature` and
  `top_k`. Claude Code does not expose those per call, so diversity
  comes from prompt-level lenses (Breadth / Depth / Skeptic). This
  preserves the *intent* — exploring a broader search space — but the
  mechanism is different.
- The paper omits TTD-DR's *Environmental Feedback* (auto-rater) and
  *Revision* (iterative critique) steps to reduce latency; this skill
  matches that choice.
- The paper uses Tavily web search with a 30% relevance score filter;
  here, candidates apply the threshold via judgment over `WebSearch`
  results.
- The paper takes top-5 web results; candidates here should aim for
  similar coverage, but final filtering is by relevance not rank.
