---
name: plan-planner
description: >
  Planner-agent operations for the Deep Plan system: create an initial
  investigation plan (analyzing both codebase and SOTA), reflect on the
  current plan against accumulated findings, update the plan, or score
  planning progress (0-100). Use when a planning session needs a one-off
  planner operation outside the full /deep-plan loop, or when the user
  invokes /plan-planner with one of the modes below.
  Args: <mode> [<topic-slug>] [--codebase <path>]. Modes: create, reflect,
  update, progress.
---

# Plan Planner — Investigation Planner Agent

This skill executes a single phase of the Planner role from the Deep Plan
system. It operates on the per-topic working directory used by `/deep-plan`:

```
.plans/<slug>/
  plan.md
  context.md
  progress.json
```

## Argument parsing

The first arg is the **mode**. The optional second arg is a topic slug
(matches a directory under `.plans/`). If no slug is given and exactly one
`.plans/*` directory exists, use it. If multiple exist, ask the user which.

For mode `create`, the args are `create <full topic string> [--codebase <path>]`
instead — a new working directory is created from a slugified version of the
topic. If `--codebase` is omitted, use the current working directory.

## Modes

### `create <topic> [--codebase <path>]`

Create the working directory, initialize `progress.json`, and produce the
initial **Investigation Plan**.

Slugify: lowercase, replace non-alphanumerics with `-`, collapse repeats,
trim to 40 chars. Working dir: `.plans/<slug>/`.

Validate the codebase path exists. Default to cwd.

Spawn a fresh `general-purpose` subagent. Brief:

> You are the Planner agent in the Deep Plan system. Topic: `<topic>`.
> Target codebase: `<codebase_path>`.
>
> First, explore the target codebase to understand its high-level structure:
> read key files (README, package.json/Cargo.toml/go.mod/etc., top-level
> directory listing, any architecture docs). Do NOT do deep exploration yet —
> just enough to understand the landscape.
>
> Then write an initial investigation plan to `<plan_path>` as a
> hierarchical markdown checklist with two major sections:
>
> **Part A — Codebase Understanding**: 3-6 investigation areas about the
> existing code — architecture, relevant modules, conventions, test
> patterns, dependencies, constraints. Each with 2-4 `- [ ]` sub-questions.
>
> **Part B — SOTA & Best Practices**: 3-6 investigation areas about
> state-of-the-art approaches, libraries, patterns, and prior art. Each
> with 2-4 `- [ ]` sub-questions.
>
> Do not perform any web search — you are a planner only. You may read
> the codebase to inform the plan. Return when the file is written.

Then write `progress.json`:
```json
{"iteration": 0, "percent": 0, "max_iterations": 12, "topic": "<topic>", "codebase_path": "<codebase_path>"}
```

Initialize `context.md`:
```markdown
# Global Research Context — <topic>

Codebase: `<codebase_path>`

---
```

### `reflect [<slug>]`

Critically review the current investigation plan against accumulated findings.

Spawn a fresh subagent. Brief:

> Read `<plan_path>` and `<context_path>`. Critically assess:
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
> ANSWERED: <list>
> EMERGED:  <list>
> TERMINATE: <list>
> BALANCE: <codebase vs SOTA coverage assessment>
> CHANGES_NEEDED: <yes|no>
> RATIONALE: <2-4 sentences>
> ```
>
> Do NOT modify the plan file.

Show the reflection block to the user.

### `update [<slug>]`

Apply the most recent reflection to the investigation plan. If no reflection
has been run in this conversation, run `reflect` first, then update.

Spawn a fresh subagent. Brief:

> Apply this reflection to `<plan_path>`:
>
> <reflection block>
>
> Edit the plan in place: tick `[x]` answered items, add new items from
> EMERGED, strike through terminated items as `~~text~~`. If BALANCE
> indicates a skew, add items to the underrepresented section. Preserve
> all already-answered history. Return when the file is updated.

### `progress [<slug>]`

Score investigation progress 0-100.

Spawn a fresh subagent. Brief:

> Read `<plan_path>` and `<context_path>`. Score investigation progress
> as an integer 0-100 reflecting whether we have enough understanding of
> BOTH the codebase AND SOTA approaches to write a high-fidelity
> implementation plan.
>
> Scoring criteria:
> - Codebase architecture and relevant modules understood (0-30)
> - SOTA approaches and best practices researched (0-30)
> - Integration strategy and constraints identified (0-20)
> - Risks, edge cases, and testing covered (0-20)
>
> Be conservative. Output exactly one JSON object on stdout:
> `{"percent": <int>, "rationale": "<one sentence>"}`

Update `progress.json` with the returned `percent` and increment
`iteration`. Show the `percent` and rationale to the user.

## Notes

- This skill is for one-off planner operations. For the full loop, use
  `/deep-plan`.
- All modes spawn fresh subagents so context stays narrow.
- The codebase path is read from `progress.json` for all modes except
  `create`.
