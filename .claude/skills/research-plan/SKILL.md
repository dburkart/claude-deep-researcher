---
name: research-plan
description: >
  Planner-agent operations for the Deep Researcher Reflect Evolve system:
  create an initial research plan, reflect on the current plan against the
  global research context, update the plan, or score research progress
  (0-100). Use when a research session needs a one-off planning operation
  outside the full /deep-research loop, or when the user invokes
  /research-plan with one of the modes below. Args: <mode> [<topic-slug>].
  Modes: create, reflect, update, progress.
---

# Research Plan — Planner Agent

This skill executes a single phase of the Planner role from the Deep
Researcher Reflect Evolve architecture. It operates on the same per-topic
working directory used by `/deep-research`:

```
.research/<slug>/
├── plan.md
├── context.md
└── progress.json
```

## Argument parsing

The first arg is the **mode**. The optional second arg is a topic slug
(matches a directory under `.research/`). If no slug is given and exactly
one `.research/*` directory exists, use it. If multiple exist, ask the
user which.

For mode `create`, the args are `create <full topic string>` instead — a
new working directory is created from a slugified version of the topic.

## Modes

### `create <topic>`

Create the working directory, initialize `progress.json`, and produce
the initial **Research Plan**.

Slugify: lowercase, replace non-alphanumerics with `-`, collapse repeats,
trim to 40 chars. Working dir: `.research/<slug>/`.

Spawn a fresh `general-purpose` subagent. Brief:

> Topic: `<topic>`. Write an initial research plan to `<plan_path>` as
> a hierarchical markdown checklist that decomposes the topic into 5–10
> investigation areas, each with 2–4 concrete sub-questions written as
> `- [ ]` checkboxes. The plan must be exhaustive enough to support a
> PhD-level report on this topic. Do not perform any web search — you
> are a planner only. Return when the file is written.

Then write `progress.json`:
```json
{"iteration": 0, "percent": 0, "max_iterations": 12, "topic": "<topic>"}
```

### `reflect [<slug>]`

Critically review the current plan against accumulated findings.

Spawn a fresh subagent. Brief:

> Read `<plan_path>` and `<context_path>`. Critically assess:
> (1) which sub-questions are now answered;
> (2) what unforeseen sub-topics emerged;
> (3) what redundant paths should be terminated.
>
> Output a structured reflection to stdout in this exact format:
>
> ```
> ANSWERED: <list>
> EMERGED:  <list>
> TERMINATE: <list>
> CHANGES_NEEDED: <yes|no>
> RATIONALE: <2-4 sentences>
> ```
>
> Do NOT modify the plan file.

Show the reflection block to the user.

### `update [<slug>]`

Apply the most recent reflection to the plan. If no reflection has been
run in this conversation, run `reflect` first, then update.

Spawn a fresh subagent. Brief:

> Apply this reflection to `<plan_path>`:
>
> <reflection block>
>
> Edit the plan in place: tick `[x]` answered items, add new items from
> EMERGED, strike through terminated items as `~~text~~`. Preserve all
> already-answered history. Return when the file is updated.

### `progress [<slug>]`

Score research progress 0–100.

Spawn a fresh subagent. Brief:

> Read `<plan_path>` and `<context_path>`. Score research progress as
> an integer 0–100 reflecting coverage AND depth needed for a PhD-level
> report. Be conservative — shallow coverage of many areas is not high
> progress. Output exactly one JSON object on stdout:
> `{"percent": <int>, "rationale": "<one sentence>"}`

Update `progress.json` with the returned `percent` and increment
`iteration`. Show the `percent` and rationale to the user.

## Notes

- This skill is for one-off planner operations. For the full loop, use
  `/deep-research`.
- All four modes spawn fresh subagents so context stays narrow — the
  invoking conversation is not polluted with plan/context content.
