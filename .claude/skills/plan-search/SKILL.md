---
name: plan-search
description: >
  Searcher-agent operations for the Deep Plan system: generate the next
  search query for an active planning session, or answer a query using the
  Candidate Crossover algorithm with dual-source search (web + codebase).
  Three parallel candidates with diverse lenses search both the internet
  and the target codebase, then merge into one consolidated answer appended
  to the global research context. Use for one-off search operations outside
  the full /deep-plan loop, or when the user invokes /plan-search.
  Args: <mode> [<topic-slug>] [<query>]. Modes: query, answer.
---

# Plan Search — Dual-Source Searcher with Candidate Crossover

This skill executes a single phase of the Searcher role from the Deep Plan
system. It searches both the **web** (for SOTA approaches, best practices,
library docs) and the **target codebase** (for architecture, patterns,
existing implementations). Same per-topic working directory as `/deep-plan`:

```
.plans/<slug>/
  plan.md
  context.md
  progress.json
```

The codebase path is read from `progress.json`.

## Argument parsing

First arg is the **mode**. Second arg is an optional topic slug. For mode
`answer`, the remaining args form an explicit query (otherwise prompt for
one or take the most recent query from `context.md`).

If no slug is given and exactly one `.plans/*` directory exists, use it.
If multiple, ask the user which.

## Modes

### `query [<slug>]`

Generate the next-best search query for the active session.

Spawn a fresh `general-purpose` subagent. Brief:

> You are the Searcher in query-generation mode for the Deep Plan system.
> Read `<plan_path>` and `<context_path>`. Identify the single
> highest-value unanswered sub-question — one whose answer would most
> increase planning progress AND is **not redundant** with any query
> already logged in `<context_path>`.
>
> Output exactly two lines to stdout:
> ```
> DOMAIN: <web|codebase|both>
> QUERY: <concise search query string>
> ```
>
> Use `web` for questions about SOTA, best practices, library comparisons,
> or external knowledge. Use `codebase` for questions about existing code
> structure, patterns, dependencies, or implementation details. Use `both`
> when the question requires understanding the current code AND comparing
> it to external approaches.
>
> Return only those two lines.

Show the returned domain + query to the user.

### `answer [<slug>] <query>`

Answer a search query using the **Candidate Crossover** algorithm with
dual-source access.

Read `codebase_path` from `progress.json`.

Optionally, the query may be prefixed with a domain tag: `web:`, `codebase:`,
or `both:`. If no prefix, default to `both`.

#### Step A — Spawn 3 candidates in parallel

In a single message, issue three Agent tool calls (subagent_type:
`general-purpose`). Each candidate has access to **both** `WebSearch` and
codebase tools (`Read`, `Bash` for grep/find/etc.). The domain tag guides
emphasis but does not restrict access.

| Candidate | Lens      | Optimization                                                                  |
|-----------|-----------|--------------------------------------------------------------------------------|
| A         | Survey    | Map out the landscape — scan relevant codebase areas AND web results. Breadth. |
| B         | Technical | Deep dive into codebase internals AND the most promising SOTA approach. Depth. |
| C         | Risk      | Surface constraints, edge cases, tests, debt, pitfalls. Challenge assumptions. |

Each candidate's brief (substitute the lens row):

> You are Candidate `<A|B|C>` of the Search Agent in the Deep Plan system.
> Query: `<query>`. Domain emphasis: `<domain>`.
> Lens: `<lens optimization>`.
> Target codebase: `<codebase_path>`.
>
> Use `WebSearch` for external/SOTA information and `Read`/`Bash` (grep,
> find, etc.) for codebase exploration. Both tools are available — use
> your judgment on the right mix for this query and your lens.
>
> For web results: keep only sources you would rate >=30% relevance.
> For codebase results: include exact file paths and line numbers.
>
> Return a concise answer (<=500 words) preserving every key fact, code
> path, function signature, version number, date, and source URL. Do NOT
> write to any file. Output only your answer text.

#### Step B — Crossover synthesis

After all three candidates return, spawn a subagent for crossover. Brief:

> You are the Crossover synthesizer for query `<query>`. Below are three
> candidate answers from dual-source search (web + codebase). Merge them
> into a single consolidated answer that retains every distinct fact,
> code reference, URL, and finding — deduplicated, with contradictions
> explicitly flagged (`! conflict:`).
>
> For codebase findings, preserve exact file paths and line references.
> For web findings, preserve source URLs.
>
> Then append to `<context_path>` a new entry in this exact format:
>
> ```
> ## Iteration <N> — <query>
>
> **Domain:** <domain>
>
> <consolidated answer>
>
> **Sources:** <bulleted list of unique URLs and codebase file paths>
>
> ---
> ```
>
> Use the next iteration number — read existing iteration headers in
> `<context_path>` and increment. Return when the append is complete.
>
> Candidate A (Survey):
> <text>
>
> Candidate B (Technical):
> <text>
>
> Candidate C (Risk):
> <text>

#### Step C — Report back

Tell the user the query has been answered and appended to `<context_path>`.
Optionally show the consolidated answer text inline if the user is
reviewing iteratively.

## Notes on adaptation from the research system

- The research system's candidates only use `WebSearch`. Deep Plan
  candidates have access to both `WebSearch` and codebase exploration
  tools (`Read`, `Bash`), enabling dual-source search in a single
  candidate crossover round.
- Lenses are adapted from Breadth/Depth/Skeptic to Survey/Technical/Risk,
  reflecting the implementation-planning context.
- The domain tag (web/codebase/both) guides emphasis but never restricts
  tool access — candidates should use their judgment.
- Word limit per candidate is raised to 500 (from 400) to accommodate
  codebase references (file paths and signatures take space).
