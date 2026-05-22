# 03 — Missions Mode

---

## Purpose

Missions Mode coordinates complex work by splitting it across an orchestrator and one or more workers. The orchestrator plans, delegates, merges results, and decides when the mission is complete. Workers execute scoped assignments and return structured handoffs.

Missions Mode is a behavioral overlay on Exec and Interactive Mode, not a completely separate agent type. It combines mission-planning skills, metadata files, validation contracts, `Task` delegation, and `EndFeatureRun` handoffs.

---

## Core Roles

| Role | Responsibility |
|---|---|
| Orchestrator | Owns mission plan, decomposes work, launches workers, reviews results, resolves conflicts. |
| Worker | Executes a bounded task, follows required skills, verifies work, returns `EndFeatureRun`. |
| User | Defines mission objective and approves or redirects high-level scope when needed. |

---

## Mission Lifecycle

```text
User requests complex work
        ↓
Orchestrator clarifies objective
        ↓
Mission plan is created
        ↓
Work is split into scoped assignments
        ↓
Workers are launched with Task
        ↓
Workers inspect / implement / verify
        ↓
Workers return EndFeatureRun handoffs
        ↓
Orchestrator merges results
        ↓
Final validation and user report
```

---

## Orchestrator Responsibilities

The orchestrator should:

- understand the user objective,
- choose task boundaries,
- avoid overlapping worker edits where possible,
- supply each worker with enough context to act independently,
- track worker status,
- inspect handoff results,
- resolve conflicts,
- perform final validation,
- produce the final user-facing summary,
- manage validation contracts and assertion coverage,
- own mission metadata such as milestones and validation state.

The orchestrator should not simply dump the full parent conversation into every worker. It should synthesize a scoped prompt containing only relevant objective, files, constraints, expected output, and any prior findings needed by that worker.

Orchestrator-only responsibilities include user communication, planning, acceptance criteria, milestone ordering, scope changes, and worker lifecycle management.

---

## Worker Responsibilities

A worker should:

- follow the assigned prompt and required skills,
- inspect relevant files before editing,
- stay within scope,
- run appropriate verification,
- report blockers rather than silently skipping work,
- return through `EndFeatureRun`,
- re-invoke required skills if compaction or context loss removes them.

Workers are usually Exec-mode agents. They should not ask the user directly unless their assignment explicitly supports that interaction path. They are stateless relative to the parent: they receive a curated prompt and their own prompt context, not the full parent thread.

---

## Mission Metadata

Mission workflows can use project-local metadata files to persist state outside model context.

Typical metadata:

| Artifact | Purpose |
|---|---|
| `mission.md` | Mission description and durable context. |
| `validation-contract.md` | Definition of done, acceptance criteria, and assertion IDs. |
| `features.json` | Maps implemented features to validation assertions. |
| `validation-state.json` | Current validation state, progress, and blockers. |
| `milestones.json` | Ordered milestone list and completion state. |
| Worker status | Assignment state and current owner. |
| Handoff report | Worker result, verification, changed files, blockers. |
| Worker artifacts directory | Files and reports written by workers for orchestrator review. |

Metadata makes long missions auditable and recoverable after context compaction or process restart. The validation contract is the definition of done: every assertion should be claimed by exactly one feature entry, with no duplicates and no orphans.

---

## Task Tool Prompt Construction

A good worker prompt includes:

- mission objective,
- worker-specific task,
- step-by-step instructions,
- files/directories of interest, ideally absolute paths,
- relevant context snippets from prior work,
- constraints and non-goals,
- required skills or procedures,
- expected verification,
- exact handoff expectations,
- requested output format, usually a structured markdown report.

Example structure:

```text
You are worker N for mission X.
Objective: ...
Your scope: ...
Do not modify: ...
Required checks: ...
Return via EndFeatureRun with: ...
```

---

## Parallelism

Missions support parallel workers when their scopes are independent.

Good parallel splits:

- frontend vs backend,
- implementation vs tests,
- documentation vs validation,
- separate packages/modules,
- independent research tracks.

Poor parallel splits:

- multiple workers editing the same file,
- worker B needing worker A's output,
- unclear ownership of shared state,
- broad overlapping bug fixes.

When dependencies exist, serialize the tasks. The `Task` merge model is synchronous and blocking: the parent waits for worker results, appends them to the parent event history, and then decides the next step.

---

## Handoff via EndFeatureRun

`EndFeatureRun` is the worker completion boundary. It should include structured evidence, not just a prose summary.

Expected handoff content:

- success state,
- implemented work,
- unfinished work,
- changed files,
- tests/checks run,
- command exit codes or observations,
- blockers,
- whether the orchestrator should continue or reassign work.

The orchestrator uses this handoff to decide whether to merge, retry, delegate follow-up, or finalize. Worker failure is treated as information, not automatic mission failure; the orchestrator may rephrase the assignment, add context, or launch a debug/fix worker.

---

## Conflict Resolution

When worker outputs conflict, the orchestrator should:

1. inspect the changed areas,
2. identify which worker owned each scope,
3. prefer the result that satisfies mission acceptance criteria,
4. run targeted validation,
5. document any discarded or merged work.

---

## Design Guidance

- Give workers narrow, self-contained assignments.
- Avoid parallel edits to the same file.
- Require structured handoffs from every worker.
- Keep mission metadata durable.
- Treat validation contracts as the source of truth for done-ness.
- Bootstrap required mission skills explicitly before planning.
- Keep `ProposeMission`/handoff-management actions separate from worker implementation tasks.
- Let the orchestrator own final validation and user communication.
