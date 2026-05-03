---
name: plan-report
description: >
  Plan-writer operation for the Deep Plan system: produce a one-shot,
  comprehensive implementation plan from an existing investigation plan
  and global research context. The deliverable is detailed enough for
  another AI agent to implement at a high bar of quality. Use when a
  /deep-plan session has accumulated enough findings and the user wants
  the plan generated immediately, or when the user invokes /plan-report.
  Args: [<topic-slug>].
---

# Plan Report — One-Shot Implementation Plan Writer

This skill executes Step 7 of the Deep Plan system: a single-pass
synthesis of the final implementation plan from the persisted investigation
plan and global research context (which includes findings from both web
search and codebase exploration). It does **not** perform new searches —
synthesis only.

## When to use

- The user wants the final implementation plan from an existing session
- A `/deep-plan` loop terminated and the user wants to regenerate the
  plan (e.g. with different framing) without rerunning the loop
- The user invokes `/plan-report`

If the working directory has empty/sparse `context.md`, warn the user and
suggest running `/deep-plan` or `/plan-search answer ...` first before
generating the plan.

## Argument parsing

Single optional arg: a topic slug under `.plans/`. If omitted and exactly
one `.plans/*` directory exists, use it. If multiple, ask the user which.

## Procedure

Resolve paths from the slug:

```
.plans/<slug>/plan.md                  -> <plan_path>
.plans/<slug>/context.md               -> <context_path>
.plans/<slug>/progress.json            -> <progress_path>
.plans/<slug>/implementation-plan.md   -> <output_path>
```

Read `codebase_path` from `progress.json`.

Spawn a fresh `general-purpose` subagent. Brief:

> You are the Implementation Plan Writer in the Deep Plan system. Read
> `<plan_path>` and `<context_path>` in full. Write the final
> implementation plan to `<output_path>` in a single inference pass.
>
> This plan will be handed off to another AI agent for implementation.
> It must contain enough detail and research context for that agent to
> execute at a high bar of quality without needing to redo any research.
> Target codebase: `<codebase_path>`.
>
> Required structure:
>
> ## 1. Executive Summary
> 2-3 paragraphs: what is being built/changed, why, and the chosen
> approach at a high level.
>
> ## 2. Background & Motivation
> Context from both codebase analysis and SOTA research that motivates
> the approach. Reference specific existing code files and external sources.
>
> ## 3. Architecture & Design Decisions
> Key design decisions with rationale. For each decision:
> - What was decided
> - What alternatives were considered (from SOTA research)
> - Why this approach was chosen given the codebase constraints
> Include ASCII/mermaid diagrams if they clarify the design.
>
> ## 4. Implementation Steps
> Ordered, numbered steps. Each step specifies:
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
> - How it integrates with existing code (reference specific functions,
>   types, modules by name and path)
>
> ## 6. Testing Strategy
> - What to test and how
> - Existing test patterns in the codebase to follow
> - Edge cases identified during research
> - Integration test considerations
>
> ## 7. Risks & Mitigations
> Known risks with concrete mitigation strategies.
>
> ## 8. Open Questions
> Anything not fully resolved, with recommended approaches for resolving
> them during implementation.
>
> ## 9. References
> All sources: codebase file paths with descriptions, external URLs with
> titles.
>
> Requirements:
> - Preserve every relevant fact, code reference, and finding from
>   context.md — this is the research the implementer will rely on
> - Be specific: file paths, function names, line numbers, not vague
>   descriptions
> - Write for an AI agent implementer: assume strong coding ability but
>   zero prior context on this codebase or topic
> - Where the context flagged contradictions (`! conflict:`), present
>   both sides with a recommended resolution
> - Do NOT mention the investigation process — the plan is about the
>   implementation
> - Length: as long as needed for completeness — do not truncate
>
> Return when the file is written. Do not summarize what you wrote —
> the file IS the deliverable.

When the subagent returns, tell the user the implementation plan is at
`<output_path>` and offer a one-paragraph summary.

## Notes

- Like the research system's reporter, this uses **one-shot generation**
  — the high-fidelity Global Research Context built during the loop is
  strong enough for single-pass synthesis.
- The implementation plan structure is fixed (unlike the research report's
  suggested structure) because implementation plans need consistent
  sections for an agent to act on reliably.
- The plan writer does NOT perform new web searches or codebase reads —
  all findings must already be in `context.md`. If the user wants more
  research first, direct them to `/plan-search answer` or resume
  `/deep-plan`.
