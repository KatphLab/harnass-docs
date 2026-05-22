# 04 — Reasoning and Runtime Behavior

---

## Purpose

This document describes the harness runtime behavior around reasoning items, event replay, token growth, caching, compaction, recovery loops, and completion discipline.

The harness is designed for long-running software tasks where the model repeatedly plans, acts through tools, observes results, and adjusts.

---

## Reasoning Items

The model may emit reasoning items before tool calls or assistant messages. Depending on provider and API shape, these items can be plaintext summaries, hidden/encrypted payloads, or structured response events.

Architecturally, reasoning items serve four roles:

1. **Planning** — the model decides what to inspect or change next.
2. **Self-correction** — the model revises conclusions after tool results.
3. **Continuity** — later turns can replay prior reasoning context.
4. **Audit signal** — operators can infer why a tool was selected when summaries are visible.

The dominant reasoning style is action-oriented: the model typically moves from understanding the user's goal to a concrete next tool action. Self-correction language such as "wait", "actually", and "I should" is a normal part of the loop rather than a failure condition.

Reasoning content should not be treated as authoritative state. Tool outputs and committed files are the source of truth.

---

## Reasoning-to-Action Loop

A typical turn follows this loop:

```text
Context + reminders
        ↓
Reasoning item
        ↓
Tool call(s) or assistant message
        ↓
Tool output(s)
        ↓
Updated event history
        ↓
Next reasoning item
```

The loop is repeated until the task is complete, blocked, or handed off.

---

## Tool-Use Behavior

Common action patterns:

| Intent | Tool pattern |
|---|---|
| Understand codebase | `Grep` / `Glob` / `LS` → `Read` |
| Modify code | `Read` → `ApplyPatch` / `Edit` / `Create` |
| Validate change | `Execute` → inspect output → fix loop |
| Track work | `TodoWrite` before and during multi-step work |
| Ask for decision | `AskUser` |
| Submit spec | `ExitSpecMode` |
| Delegate work | `Task` |
| Complete worker run | `EndFeatureRun` |

The model often reasons in intent language rather than naming exact tools. Tool descriptions then guide the final call shape. This is why schema descriptions matter: they shape the final translation from intent to tool invocation.

---

## Parallel Tool Calls

The harness supports parallel tool calls when operations are independent. Parallelism can range from small reconnaissance bundles to large burst-reading/searching batches, but dependent edits and validation loops should remain serialized.

Good parallelism:

- read several unrelated files,
- search multiple independent patterns,
- inspect directory structure while reading docs,
- update todos while performing an independent read.

Bad parallelism:

- editing a file before reading it,
- running tests before applying the patch under test,
- launching overlapping workers on the same files,
- issuing dependent shell commands as separate parallel calls.

Dependency chains should stay sequential.

---

## Event Replay and Token Growth

The harness uses cumulative event history. Later turns can include prior user messages, reminders, reasoning items, tool calls, tool outputs, and assistant messages.

This has important consequences:

- token usage grows over the session,
- prompt caching becomes important,
- event processors must deduplicate replayed tool events,
- stale context can influence later behavior,
- compaction or summarization is needed for very long tasks.

Architecture should treat a request as a snapshot of accumulated state, not as a single isolated delta.

---

## Token Caching

Long conversations depend on provider-side or harness-side caching. Most repeated history is unchanged between turns, so cached prompt tokens can keep latency and cost manageable.

Design implications:

- stable prompt prefixes improve cache reuse,
- avoid unnecessary churn in long system/tool descriptions,
- keep dynamic reminders concise,
- summarize or compact when history becomes unwieldy,
- preserve stable ids for tool calls and outputs,
- avoid changing full tool schemas unnecessarily because schema changes can invalidate provider caches.

---

## Conversation Compaction

Compaction converts long event histories into structured summaries. It is needed when sessions become too large or when subagent work needs to be merged into a parent conversation.

A good summary preserves:

1. user objective,
2. completed work,
3. current plan,
4. files changed or inspected,
5. decisions made,
6. errors encountered and fixes tried,
7. tests/checks run,
8. remaining tasks,
9. blockers,
10. next recommended action.

Compaction should avoid losing approval state, open todos, or handoff results.

---

## Error Recovery

The harness expects retry and recovery loops rather than immediate failure.

Common recovery patterns:

| Error | Expected response |
|---|---|
| File not found | Use `LS`, `Glob`, or `Grep` to locate the correct path. |
| Patch failed | Re-read the file and apply a smaller/context-correct patch. |
| Command failed | Inspect stderr/stdout, fix root cause, rerun targeted command. |
| Missing dependency | Check package manager files before adding/installing. |
| Ambiguous user requirement | Use `AskUser` if the choice affects implementation. |
| Spec not approved | Do not mutate; revise or wait for approval. |

Recovery should remain scoped. The agent should not perform broad unrelated refactors just because a local fix failed. Long test-fix loops need an operator-visible budget or stop condition so flaky failures do not become unbounded tool cycles.

---

## Verification Behavior

Before final completion, the model should verify the result with appropriate commands.

Verification hierarchy:

1. targeted unit tests for changed code,
2. lint/typecheck for touched language/package,
3. integration or build checks when relevant,
4. full project check when feasible,
5. manual inspection when automated checks are unavailable.

Final responses should distinguish between checks that passed, checks not run, and checks blocked by environment issues.

---

## Completion Discipline

A task is complete when:

- requested work is implemented,
- obvious regressions are addressed,
- tests/checks have passed or limitations are reported,
- todos are completed or remaining work is disclosed,
- any handoff protocol has been satisfied.

The final answer should be concise and include changed areas plus validation status.
