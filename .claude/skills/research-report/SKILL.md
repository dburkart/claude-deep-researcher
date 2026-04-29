---
name: research-report
description: >
  Reporter-agent operation for the Deep Researcher Reflect Evolve system:
  produce a one-shot, PhD-level research report from an existing research
  plan and global research context. Use when a /deep-research session has
  accumulated enough findings and the user wants the final write-up
  generated immediately, or when the user invokes /research-report.
  Args: [<topic-slug>].
---

# Research Report — One-Shot Reporter Agent

This skill executes Step 7 of the Deep Researcher Reflect Evolve
architecture: a single-pass synthesis of the final research report from
the persisted plan and global research context. It does **not** perform
new web searches — synthesis only.

## When to use

- The user wants the final write-up from an existing research session
- A `/deep-research` loop terminated and the user wants to regenerate
  the report (e.g. with different framing) without rerunning the loop
- The user invokes `/research-report`

If the working directory has empty/sparse `context.md`, warn the user and
suggest running `/deep-research` or `/research-search answer …` first
before generating the report.

## Argument parsing

Single optional arg: a topic slug under `.research/`. If omitted and
exactly one `.research/*` directory exists, use it. If multiple, ask
the user which.

## Procedure

Resolve paths from the slug:

```
.research/<slug>/plan.md     → <plan_path>
.research/<slug>/context.md  → <context_path>
.research/<slug>/report.md   → <report_path>
```

Spawn a fresh `general-purpose` subagent. Brief:

> You are the Report Writer in the Deep Researcher Reflect Evolve
> system. Read `<plan_path>` and `<context_path>` in full. Write the
> final research report to `<report_path>` in a single inference pass.
>
> Requirements:
> - Cohesive PhD-level narrative — integrate findings into a unified
>   structure, not a list of summaries per query
> - Preserve every distinct fact, number, date, and source from
>   `<context_path>`. Maximize fact density.
> - Cite sources inline as `[domain.com]` and include a numbered
>   `## Sources` section at the end with full URLs
> - Where the context flagged contradictions (`⚠ conflict:`), surface
>   them honestly in the narrative — do not paper over them
> - Length: as long as the evidence supports — do not pad and do not
>   truncate. PhD-level depth is the target.
> - Do NOT mention the agentic process, the plan, the candidates, or
>   the iteration loop. The report is about the topic, full stop.
> - Do NOT add a "Limitations" section unless evidence in the context
>   actually identified concrete limitations of the underlying topic
>
> Suggested structure (adapt to the topic):
> 1. Abstract / executive summary (≤200 words)
> 2. Introduction & motivation
> 3. Body sections corresponding to the major investigation areas
> 4. Synthesis / cross-cutting analysis
> 5. Conclusion
> 6. Sources
>
> Return when the file is written. Do not summarize what you wrote —
> the file IS the deliverable.

When the subagent returns, tell the user the report is at
`<report_path>` and offer a one-paragraph summary.

## Notes on faithfulness to the paper

- The paper deliberately uses **One-Shot Report Generation** rather than
  TTD-DR's iterative *Report-level Denoising* — the reasoning is that
  the high-fidelity Global Research Context built during the sequential
  loop is already strong enough that one-pass synthesis preserves
  computational efficiency without quality loss. This skill matches
  that choice exactly.
- The paper does NOT prescribe a fixed report structure; the suggested
  structure above is a sensible default the writer can adapt.
