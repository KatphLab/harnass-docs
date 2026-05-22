# 05 — Message Flow

---

## Purpose

Message flow describes how the harness represents conversation state, tool calls, tool outputs, reasoning, reminders, handoffs, and final responses.

The harness should be modeled as an event-driven conversation system rather than a simple alternating chat transcript.

---

## Model Routing Flow

The model-facing request can pass through a routing/proxy layer before reaching the inference provider:

```text
CLI / orchestrator
    ↓
Prompt and tool assembly
    ↓
Model proxy / router
    ↓
Inference provider
    ↓
Response parser
    ↓
Tool router or assistant output
```

The routing layer may provide model selection, fallback, rate limiting, token accounting, prompt caching, and optional request/response logging. These concerns are runtime architecture, not application logic, but they strongly affect cost, latency, privacy, and reliability.

---

## Conversation Topology

A typical agentic session has this shape:

```text
system / instructions
user request
harness reminders
assistant reasoning
assistant tool call(s)
tool output(s)
assistant reasoning
assistant tool call(s)
tool output(s)
...
assistant final response
```

In Responses-style APIs, these become an ordered event stream. Later requests may replay earlier events so the model can maintain continuity.

---

## Event Types

| Event | Purpose |
|---|---|
| System instructions | Stable operating contract. |
| User message | Human request or answer. |
| System reminder | Harness-authored dynamic instruction. |
| Reasoning item | Model planning or summarized internal state. |
| Function call | Ordinary tool invocation. |
| Function output | Result of ordinary tool invocation. |
| Custom tool call | Nonstandard tool channel, e.g. patch application. |
| Custom tool output | Result of custom tool invocation. |
| Assistant message | User-facing text. |

A user-role event is not always human-authored. Harness reminders may be represented in the same role slot and must be classified by content/metadata.

---

## Request as Snapshot

Each model request should be treated as a snapshot of current conversation state.

Implications:

- repeated replayed events are normal,
- token count grows with session length,
- tool outputs may appear many turns after they were produced,
- handoff results remain visible to the model,
- metrics must deduplicate by ids such as `call_id`.

Do not assume one logged request equals one new turn of work.

---

## Common Flow Patterns

### Reconnaissance

```text
User asks for change
    ↓
Grep / Glob / LS
    ↓
Read relevant files
    ↓
TodoWrite plan
```

### Edit and Validate

```text
Read current code
    ↓
ApplyPatch / Edit / Create
    ↓
Execute targeted tests
    ↓
Fix failures
    ↓
Execute broader checks
```

### Spec-Gated Work

```text
Spec reminder
    ↓
Read/search only
    ↓
ExitSpecMode(plan)
    ↓
Approval result
    ↓
Implementation tools
```

### User Blocker

```text
Tool/output reveals blocked decision
    ↓
AskUser(question)
    ↓
User answer
    ↓
Resume work
```

### Subagent Delegation

```text
Parent identifies independent task
    ↓
Parent synthesizes scoped prompt
    ↓
Task(worker prompt)
    ↓
Worker performs scoped work in isolated context
    ↓
Worker returns result/handoff
    ↓
Parent appends result and decides next step
```

The parent-to-worker merge is synchronous: the parent waits for the worker result before continuing that branch of reasoning.

---

## Parallel vs Sequential Calls

Parallel calls are appropriate for independent reads/searches. Sequential calls are required when later steps depend on earlier results.

| Pattern | Parallel? | Reason |
|---|---:|---|
| Read three unrelated files | Yes | Independent observations. |
| Grep for several symbols | Yes | Independent searches. |
| Read then patch same file | No | Patch depends on current content. |
| Patch then test | No | Test depends on patch. |
| Launch workers for separate modules | Yes | Independent scopes. |
| Launch workers for same file | Usually no | Conflict risk. |

---

## Handoff Events

Handoffs are explicit state transitions in the message flow.

| Handoff | Tool | Transition |
|---|---|---|
| User clarification | `AskUser` | Model waits for user decision. |
| Spec approval | `ExitSpecMode` | Planning becomes implementation after approval. |
| Subagent result | `Task` result | Worker result returns to parent. |
| Mission proposal / handoff management | Mission planning actions | Orchestrator proposes, dismisses, or updates mission handoff items. |
| Mission completion | `EndFeatureRun` | Worker returns structured completion to orchestrator. |

Handoff outputs should remain in history until they are no longer needed or until a summary preserves their state.

---

## Finalization Flow

Before final response, the model should:

1. complete or update todos,
2. run relevant validation,
3. inspect failures if any,
4. fix or disclose remaining issues,
5. summarize changed files/behavior,
6. report validation status.

Final response should be short, concrete, and status-oriented.

---

## Message Processor Requirements

A robust harness implementation should:

- preserve event order,
- keep stable call ids,
- pair tool outputs with tool calls,
- distinguish human messages from reminders,
- support function and custom tool channels,
- preserve handoff results,
- compact long histories without losing active state,
- avoid duplicating replayed events in analytics or UI summaries.
