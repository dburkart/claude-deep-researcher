---
name: deep-research
description: >
  Run the Deep Researcher Reflect Evolve loop on a topic to produce a PhD-level
  research report. Implements the architecture from Prateek 2026 (arXiv:2601.20843):
  sequential research-plan refinement via reflection, candidate crossover for web
  search answers, and one-shot final report generation. Use when the user asks for
  "deep research", a long-form report, an investigation that requires iterative
  web search + planning, or invokes /deep-research. Args: <research topic>.
---

# Deep Research — Reflect Evolve

This skill orchestrates a multi-agent research loop that produces a long-form,
fact-dense report on a given topic. It implements the architecture from
*Deep Researcher with Sequential Plan Reflection and Candidates Crossover*
(Prateek 2026).

## When to use

Invoke this skill when the user asks for:
- A "deep research" report, dossier, or long-form investigation
- A topic that requires multiple rounds of web search + reasoning
- The slash command `/deep-research <topic>`

The user's argument string is the **research topic**. Quote it verbatim — do
not paraphrase.

## Architecture overview

Two persistent artifacts live in a per-topic working directory:

- `plan.md` — the **Research Plan** (a checklist of investigation steps)
- `context.md` — the **Global Research Context**: an append-only log of every
  `(query, answer, sources)` triple produced by the Search Agent

Three agent roles operate against those artifacts:

- **Planner** — creates the plan, reflects on it, updates it, scores progress
- **Searcher** — generates the next query, answers it via web search + candidate crossover
- **Reporter** — one-shot final report writer

The loop runs Steps 2–6 until progress ≥ 90% **or** `max_iterations` retries
are exhausted (default 12). Then Step 7 emits the report.

## Working directory

Pick a slug from the topic (lowercase, hyphenated, ≤40 chars). Working
directory is `.research/<slug>/` rooted at the user's current cwd. Create it
if missing. Files:

```
.research/<slug>/
├── plan.md          # Research Plan (markdown, hierarchical checklist)
├── context.md       # Global Research Context (append-only log)
├── progress.json    # {iteration, percent, max_iterations, topic}
└── report.md        # final output (written by Step 7)
```

If a directory for the same slug already exists, ask the user whether to
**resume** (continue from current `progress.json`) or **restart** (wipe and
start over). Default to resume.

## The 7-step loop

Each numbered step below corresponds to a phase in the paper. Steps 1, 4, 5,
6, 7 are sequential. Step 3 fans out 3 parallel candidates and crosses over.

### Step 1 — Plan creation (one-time, at start)

Spawn a fresh `general-purpose` subagent acting as the **Planner**. Brief:

> You are the Planner agent in the Deep Researcher Reflect Evolve system.
> Topic: `<topic>`. Write an initial research plan to `<plan_path>` as a
> hierarchical markdown checklist. The plan must (a) decompose the topic
> into 5–10 investigation areas, (b) for each area list 2–4 concrete
> sub-questions to answer via web search, (c) be exhaustive enough to
> support a PhD-level report. Output format: GitHub-flavored markdown with
> `- [ ]` checkboxes for each sub-question. Do not perform any web
> search yourself — you are a planner only. Return when the file is written.

Then initialize `progress.json` with `{iteration: 0, percent: 0, max_iterations: 12, topic: <topic>}`.

### Step 2 — Generate search query

Spawn a fresh subagent as the **Searcher (query-gen)**. Brief:

> You are the Searcher agent. Read `<plan_path>` and `<context_path>`.
> Identify the single highest-value unanswered sub-question — one whose
> answer would most increase research progress and is **not redundant**
> with any query already in the context log. Output exactly one line to
> stdout: a concise web-search query string (no quotes, no markdown).
> Return only the query.

Capture the returned query string.

### Step 3 — Answer search query (Candidate Crossover, n=3)

This is the **Candidate Crossover** algorithm. Spawn 3 subagents **in
parallel** in a single message (one Agent tool call per candidate). Each
candidate has a different *lens* (since Claude Code can't vary
temperature/top_k per call, prompt-level diversity is the variation
mechanism):

- **Candidate A — Breadth lens**: gather the widest set of relevant facts,
  numbers, and sources. Optimize for coverage.
- **Candidate B — Depth lens**: drill into mechanisms, causation, and
  technical detail on the most central aspect of the query. Optimize for
  insight.
- **Candidate C — Skeptic lens**: surface contradicting evidence, edge
  cases, recency, and source-quality concerns. Optimize for fact-checking.

Each candidate's brief:

> You are Candidate `<A|B|C>` of the Search Agent. Query: `<query>`.
> Lens: `<lens description above>`. Use the `WebSearch` tool to gather
> current information. Filter results: keep only sources you would rate
> ≥30% relevance. Return a concise answer (≤400 words) preserving all
> facts, numbers, dates, and source URLs. Do NOT write to any file. Output
> only your answer text.

After all 3 return, **fork yourself** (Agent without subagent_type) for the
**crossover step**. Brief:

> You are the Crossover synthesizer. Three candidate answers follow for
> the query `<query>`. Merge them into a single consolidated answer that
> retains every distinct fact, number, date, and URL — deduplicated, with
> contradictions explicitly noted. Then append to `<context_path>` a new
> entry in this exact format:
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
> Candidate A: <text>
> Candidate B: <text>
> Candidate C: <text>

### Step 4 — Reflect on plan

Spawn a fresh subagent as the **Planner (reflect)**. Brief:

> You are the Planner in reflection mode. Read `<plan_path>` and
> `<context_path>`. Critically assess: (1) which sub-questions are now
> answered; (2) what unforeseen sub-topics emerged; (3) what redundant
> paths should be terminated. Output a structured reflection to stdout:
>
> ```
> ANSWERED: <list of sub-questions now answered>
> EMERGED:  <list of new sub-topics discovered>
> TERMINATE: <list of plan items to drop>
> CHANGES_NEEDED: <yes|no>
> RATIONALE: <2-4 sentences>
> ```
>
> Do NOT modify the plan file. Return only the reflection block.

### Step 5 — Maybe update plan

If the reflection reports `CHANGES_NEEDED: no`, skip this step. Otherwise
spawn a fresh subagent as the **Planner (update)**. Brief:

> You are the Planner in update mode. Apply this reflection to
> `<plan_path>`:
>
> <full reflection block from Step 4>
>
> Edit the plan in place: tick `[x]` answered items, add new items
> discovered in EMERGED, strike through (`~~text~~`) terminated items.
> Preserve all already-answered history. Return when the file is updated.

### Step 6 — Analyze progress

Spawn a fresh subagent as the **Planner (progress)**. Brief:

> You are the Planner in progress-scoring mode. Read `<plan_path>` and
> `<context_path>`. Score research progress as an integer 0–100 reflecting
> coverage AND depth needed for a PhD-level report. Be conservative —
> shallow coverage of many areas is not high progress. Output exactly
> one JSON object on stdout: `{"percent": <int>, "rationale": "<one sentence>"}`.

Update `progress.json` with the returned `percent` and increment `iteration`.

**Loop control:**
- If `percent >= 90` → break and go to Step 7
- Else if `iteration >= max_iterations` → break and go to Step 7
- Else → return to Step 2

### Step 7 — One-shot report generation

Spawn a fresh subagent as the **Reporter**. Brief:

> You are the Report Writer. Read `<plan_path>` and `<context_path>`
> in full. Write the final research report to `<report_path>` in one
> pass. Requirements:
> - Cohesive PhD-level narrative, not a list of summaries
> - Integrate findings across all iterations into a unified structure
> - Preserve every distinct fact, number, date from context.md
> - Cite sources inline as `[domain.com]` and include a Sources section
> - Length: as long as the evidence supports — do not pad
> - No mention of the agentic process itself; the report is about the topic
>
> Return when the file is written.

When the Reporter returns, tell the user the report is at
`.research/<slug>/report.md` and offer a one-paragraph summary.

## User-facing communication

Throughout the loop, give the user one short status line per phase
transition (e.g. "Iteration 3: querying 'X' (3 candidates in flight)…",
"Progress: 67%"). Don't dump intermediate plan or context content unless
the user asks.

## Notes on faithfulness to the paper

- The paper uses Tavily for web search; this skill uses Claude Code's
  `WebSearch` tool. The 30% relevance filter is delegated to candidate
  judgment.
- The paper varies `temperature` and `top_k` across the 3 candidates.
  Claude Code does not expose those per-call, so we substitute
  prompt-level diversity (Breadth / Depth / Skeptic lenses). This
  preserves the *intent* of exploring a larger search space.
- The paper omits TTD-DR's Environmental Feedback (Step 2) and Revision
  (Step 3) for latency; this skill matches that choice.
- Default `max_iterations = 12` is a guardrail not specified in the
  paper. Override via the user's request if asked.

## When to depart from this loop

If the user explicitly asks for a quick answer, summary, or single search,
DO NOT invoke this skill — do the simpler thing. This skill is heavyweight
(many subagent calls, many web searches) and only worth it for genuine
deep research requests.
