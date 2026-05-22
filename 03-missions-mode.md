# 03 — Missions Mode

---

## Purpose

Missions Mode coordinates complex work by splitting it across an orchestrator and one or more workers. The orchestrator plans, delegates, merges results, and decides when the mission is complete. Workers execute scoped assignments and return structured handoffs.

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
- produce the final user-facing summary.

The orchestrator should not simply dump the full parent conversation into every worker. It should synthesize a scoped prompt containing only relevant objective, files, constraints, and expected output.

---

## Worker Responsibilities

A worker should:

- follow the assigned prompt and required skills,
- inspect relevant files before editing,
- stay within scope,
- run appropriate verification,
- report blockers rather than silently skipping work,
- return through `EndFeatureRun`.

Workers are usually Exec-mode agents. They should not ask the user directly unless their assignment explicitly supports that interaction path.

---

## Mission Metadata

Mission workflows can use project-local metadata files to persist state outside model context.

Typical metadata:

| Artifact | Purpose |
|---|---|
| Mission plan | Objective, scope, assignments, acceptance criteria. |
| Worker status | Assignment state and current owner. |
| Handoff report | Worker result, verification, changed files, blockers. |
| Validation record | Final checks and pass/fail evidence. |

Metadata makes long missions auditable and recoverable after context compaction or process restart.

---

## Task Tool Prompt Construction

A good worker prompt includes:

- mission objective,
- worker-specific task,
- files/directories of interest,
- constraints and non-goals,
- required skills or procedures,
- expected verification,
- exact handoff expectations.

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

When dependencies exist, serialize the tasks.

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

The orchestrator uses this handoff to decide whether to merge, retry, delegate follow-up, or finalize.

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
- Let the orchestrator own final validation and user communication.
