---
name: deep-plan
description: >
  Run the Deep Researcher Reflect Evolve loop to produce a comprehensive
  implementation plan for a given topic, searching both a target codebase and
  the web for SOTA approaches. Adapts the architecture from Prateek 2026
  (arXiv:2601.20843) to planning: sequential investigation-plan refinement via
  reflection, candidate crossover for dual-source answers, and one-shot final
  plan generation. Use when the user asks for a "deep plan", an implementation
  plan that requires codebase understanding + SOTA research, or invokes
  /deep-plan. Args: <topic> [--codebase <path>].
---

# Deep Plan — Reflect Evolve for Implementation Planning

This skill orchestrates a multi-agent investigation loop that produces a
detailed, actionable implementation plan for a given topic. It adapts the
Deep Researcher Reflect Evolve architecture to bridge **codebase reality**
with **state-of-the-art approaches**, producing a plan with enough fidelity
for another agent to implement at a high bar of quality.

## When to use

Invoke this skill when the user asks for:
- A "deep plan" or implementation plan requiring codebase + SOTA research
- A topic that needs understanding existing code AND researching best practices
- The slash command `/deep-plan <topic>`

The user's argument string is the **topic**. Quote it verbatim — do not
paraphrase.

## Argument parsing

The argument string is the **topic** for the plan. Optionally, the user
may append `--codebase <path>` to specify the target codebase root. If
`--codebase` is not provided, default to the user's current working
directory.

Validate that the codebase path exists and is a directory. If it does not
exist, ask the user to correct it.

## Architecture overview

Two persistent artifacts live in a per-topic working directory:

- `plan.md` — the **Investigation Plan** (a checklist of questions to
  answer about the codebase and SOTA approaches)
- `context.md` — the **Global Research Context**: an append-only log of
  every `(query, answer, sources)` triple produced by the Search Agent,
  sourced from both web search and codebase exploration

Three agent roles operate against those artifacts:

- **Planner** — creates the investigation plan, reflects on it, updates it,
  scores progress
- **Searcher** — generates the next query, answers it via dual-source
  candidate crossover (web + codebase)
- **Plan Writer** — one-shot implementation plan author

The loop runs Steps 2-6 until progress >= 90% **or** `max_iterations`
retries are exhausted (default 12). Then Step 7 emits the implementation
plan.

## Working directory

Pick a slug from the topic (lowercase, hyphenated, <=40 chars). Working
directory is `.plans/<slug>/` rooted at the user's current cwd. Create it
if missing. Files:

```
.plans/<slug>/
  plan.md                # Investigation Plan (markdown, hierarchical checklist)
  context.md             # Global Research Context (append-only log)
  progress.json          # {iteration, percent, max_iterations, topic, codebase_path}
  implementation-plan.md # Final deliverable (written by Step 7)
```

If a directory for the same slug already exists, ask the user whether to
**resume** (continue from current `progress.json`) or **restart** (wipe and
start over). Default to resume.

## The 7-step loop

Each numbered step below corresponds to a phase adapted from the paper.
Steps 1, 4, 5, 6, 7 are sequential. Step 3 fans out 3 parallel candidates
and crosses over.

### Step 1 — Investigation plan creation (one-time, at start)

Spawn a fresh `general-purpose` subagent acting as the **Planner**. Brief:

> You are the Planner agent in the Deep Plan system. Topic: `<topic>`.
> Target codebase: `<codebase_path>`.
>
> First, explore the target codebase to understand its high-level structure:
> read key files (README, package.json/Cargo.toml/go.mod/etc., top-level
> directory listing, any architecture docs). Do NOT do deep exploration yet —
> just enough to understand the landscape.
>
> Then write an initial investigation plan to `<plan_path>` as a hierarchical
> markdown checklist. The plan must have two major sections:
>
> **Part A — Codebase Understanding**: 3-6 investigation areas about the
> existing code — architecture, relevant modules, conventions, test patterns,
> dependencies, constraints, and integration points related to `<topic>`.
> Each area should have 2-4 concrete sub-questions as `- [ ]` checkboxes.
>
> **Part B — SOTA & Best Practices**: 3-6 investigation areas about
> state-of-the-art approaches, industry best practices, libraries, patterns,
> and prior art relevant to `<topic>`. Each area should have 2-4 concrete
> sub-questions as `- [ ]` checkboxes.
>
> The plan must be exhaustive enough to support a detailed implementation
> plan that another agent could pick up and execute at high quality. Return
> when the file is written.

Then initialize `progress.json`:
```json
{"iteration": 0, "percent": 0, "max_iterations": 12, "topic": "<topic>", "codebase_path": "<codebase_path>"}
```

Initialize `context.md` with a header:
```markdown
# Global Research Context — <topic>

Codebase: `<codebase_path>`

---
```

### Step 2 — Generate search query

Spawn a fresh subagent as the **Searcher (query-gen)**. Brief:

> You are the Searcher agent in query-generation mode. Read `<plan_path>`
> and `<context_path>`. Identify the single highest-value unanswered
> sub-question — one whose answer would most increase planning progress
> and is **not redundant** with any query already in the context log.
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

Capture the returned domain and query string.

### Step 3 — Answer search query (Candidate Crossover, n=3)

This is the **Candidate Crossover** algorithm, adapted for dual-source
search. Spawn 3 subagents **in parallel** in a single message. Each
candidate has a different *lens* and access to **both** `WebSearch` and
the codebase (via `Read`, `Bash` with grep/find/etc.):

- **Candidate A — Survey lens**: Map out the landscape — scan relevant
  codebase files/directories AND search the web for approaches. Optimize
  for breadth and coverage of options.
- **Candidate B — Technical lens**: Deep dive into the most relevant
  codebase internals (read specific files, trace call chains) AND the
  most promising SOTA approach. Optimize for mechanistic understanding.
- **Candidate C — Risk lens**: Surface constraints, edge cases, existing
  tests, version conflicts, breaking changes, and technical debt in the
  codebase. Search the web for known pitfalls. Optimize for risk
  identification.

The domain tag from Step 2 guides emphasis but does NOT restrict access:

- `web` → candidates emphasize web search but may still glance at the
  codebase for grounding
- `codebase` → candidates emphasize file reading/grep but may still web
  search for context
- `both` → candidates actively use both search spaces

Each candidate's brief:

> You are Candidate `<A|B|C>` of the Search Agent in the Deep Plan system.
> Query: `<query>`. Domain emphasis: `<domain>`.
> Lens: `<lens description above>`.
> Target codebase: `<codebase_path>`.
>
> Use `WebSearch` for external/SOTA information and `Read`/`Bash` (grep,
> find, etc.) for codebase exploration. Both are available regardless of
> domain emphasis — use your judgment on the right mix for this query
> and your lens.
>
> Return a concise answer (<=500 words) preserving all key facts, code
> paths, file names, function signatures, version numbers, dates, and
> source URLs. For codebase findings, include file paths and line numbers.
> Do NOT write to any file. Output only your answer text.

After all 3 return, spawn a subagent for the **crossover step**. Brief:

> You are the Crossover synthesizer. Three candidate answers follow for
> the query `<query>`. Merge them into a single consolidated answer that
> retains every distinct fact, code reference, URL, and finding —
> deduplicated, with contradictions explicitly flagged (`! conflict:`).
>
> For codebase findings, preserve exact file paths and line references.
> For web findings, preserve source URLs.
>
> Then append to `<context_path>` a new entry in this exact format:
>
> ```
> ## Iteration <N> — <query>
>
> **Domain:** <web|codebase|both>
>
> <consolidated answer>
>
> **Sources:** <bulleted list of unique URLs and codebase file paths>
>
> ---
> ```
>
> Candidate A (Survey): <text>
> Candidate B (Technical): <text>
> Candidate C (Risk): <text>

### Step 4 — Reflect on investigation plan

Spawn a fresh subagent as the **Planner (reflect)**. Brief:

> You are the Planner in reflection mode. Read `<plan_path>` and
> `<context_path>`. Critically assess:
> (1) which sub-questions are now answered;
> (2) what unforeseen aspects emerged (new codebase constraints, new
>     SOTA options, integration challenges);
> (3) what redundant paths should be terminated;
> (4) whether the balance between codebase understanding and SOTA
>     research is appropriate.
>
> Output a structured reflection to stdout:
>
> ```
> ANSWERED: <list of sub-questions now answered>
> EMERGED:  <list of new sub-topics discovered>
> TERMINATE: <list of plan items to drop>
> BALANCE: <assessment of codebase vs SOTA coverage>
> CHANGES_NEEDED: <yes|no>
> RATIONALE: <2-4 sentences>
> ```
>
> Do NOT modify the plan file. Return only the reflection block.

### Step 5 — Maybe update investigation plan

If the reflection reports `CHANGES_NEEDED: no`, skip this step. Otherwise
spawn a fresh subagent as the **Planner (update)**. Brief:

> You are the Planner in update mode. Apply this reflection to
> `<plan_path>`:
>
> <full reflection block from Step 4>
>
> Edit the plan in place: tick `[x]` answered items, add new items
> discovered in EMERGED, strike through (`~~text~~`) terminated items.
> If BALANCE indicates a skew, add items to the underrepresented section.
> Preserve all already-answered history. Return when the file is updated.

### Step 6 — Analyze progress

Spawn a fresh subagent as the **Planner (progress)**. Brief:

> You are the Planner in progress-scoring mode. Read `<plan_path>` and
> `<context_path>`. Score investigation progress as an integer 0-100
> reflecting whether we have enough understanding of BOTH the codebase
> AND SOTA approaches to write a high-fidelity implementation plan.
>
> Scoring criteria:
> - Codebase architecture and relevant modules are well understood (0-30)
> - SOTA approaches and best practices are researched (0-30)
> - Integration strategy and constraints are identified (0-20)
> - Risks, edge cases, and testing considerations are covered (0-20)
>
> Be conservative — shallow coverage of many areas is not high progress.
> Output exactly one JSON object on stdout:
> `{"percent": <int>, "rationale": "<one sentence>"}`

Update `progress.json` with the returned `percent` and increment
`iteration`.

**Loop control:**
- If `percent >= 90` -> break and go to Step 7
- Else if `iteration >= max_iterations` -> break and go to Step 7
- Else -> return to Step 2

### Step 7 — One-shot implementation plan generation

Spawn a fresh `general-purpose` subagent as the **Plan Writer**. Brief:

> You are the Implementation Plan Writer in the Deep Plan system. Read
> `<plan_path>` and `<context_path>` in full. Write the final
> implementation plan to `<implementation_plan_path>` in one pass.
>
> This plan will be handed off to another AI agent for implementation.
> It must have enough detail and fidelity for that agent to execute at a
> high bar of quality without needing to redo the research.
>
> Required structure:
>
> ## 1. Executive Summary
> 2-3 paragraphs: what is being built/changed, why, and the chosen
> approach at a high level.
>
> ## 2. Background & Motivation
> Context from both codebase analysis and SOTA research that motivates
> the approach. Reference specific existing code and external sources.
>
> ## 3. Architecture & Design Decisions
> Key design decisions with rationale. For each decision, explain:
> - What was decided
> - What alternatives were considered (from SOTA research)
> - Why this approach was chosen given the codebase constraints
> Include diagrams in ASCII/mermaid if they clarify the design.
>
> ## 4. Implementation Steps
> Ordered, numbered steps. Each step must specify:
> - What to do (concrete action)
> - Which files to create or modify (exact paths)
> - Key code patterns to follow (referencing existing codebase conventions)
> - Dependencies on other steps
> - Estimated complexity (low/medium/high)
>
> ## 5. Detailed File Changes
> For each file that needs creation or modification:
> - File path
> - What changes are needed and why
> - Key interfaces/types/functions to implement
> - How it integrates with existing code (reference specific existing
>   functions, types, modules)
>
> ## 6. Testing Strategy
> - What to test and how
> - Existing test patterns in the codebase to follow
> - Edge cases identified during research
> - Integration test considerations
>
> ## 7. Risks & Mitigations
> Known risks from the investigation with concrete mitigation strategies.
>
> ## 8. Open Questions
> Anything that could not be fully resolved during investigation, with
> recommended approaches for resolving them during implementation.
>
> ## 9. References
> All sources: codebase file paths with descriptions, and external URLs
> with titles.
>
> Requirements:
> - Preserve every relevant fact, code reference, and finding from
>   context.md
> - Be specific — file paths, function names, line numbers, not vague
>   hand-waving
> - Write for an agent implementer: assume strong coding ability but
>   zero prior context on this codebase or topic
> - Do NOT mention the investigation process itself — the plan is about
>   the implementation, full stop
> - Length: as long as needed for completeness — do not truncate
>
> Return when the file is written.

When the Plan Writer returns, tell the user the implementation plan is at
`.plans/<slug>/implementation-plan.md` and offer a one-paragraph summary.

## User-facing communication

Throughout the loop, give the user one short status line per phase
transition (e.g. "Iteration 3: querying 'X' [codebase+web] (3 candidates
in flight)...", "Progress: 67% — codebase well-mapped, SOTA coverage
needs work"). Don't dump intermediate plan or context content unless the
user asks.

## Notes on adaptation from the research system

- The paper's architecture (Prateek 2026) is preserved: sequential plan
  refinement via reflection, candidate crossover for answers, one-shot
  final generation. The adaptation adds a second search space (codebase)
  alongside web search.
- Candidate diversity still comes from prompt-level lenses (Survey /
  Technical / Risk), adapted from the research system's Breadth / Depth /
  Skeptic lenses.
- The investigation plan has two sections (Codebase + SOTA) to ensure
  balanced coverage. The reflect step monitors this balance.
- The final deliverable is an implementation plan, not a research report.
  It is structured for actionability, not narrative.
- Default `max_iterations = 12` matches the research system. Override
  if the user requests it.

## When to depart from this loop

If the user explicitly asks for a quick implementation sketch, a single
file change, or something that doesn't need research, DO NOT invoke this
skill — do the simpler thing. This skill is heavyweight (many subagent
calls, many web+codebase searches) and only worth it for genuine deep
planning where understanding the codebase AND SOTA matters.
