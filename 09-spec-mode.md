# 09 — Spec Mode

---

## Purpose

Spec Mode is the harness's planning gate for interactive coding work. It allows repository inspection and requirements clarification, but defers mutation until the user approves a concrete implementation plan.

Spec Mode is implemented as an overlay on Interactive Mode:

```text
Base Interactive Prompt
        +
Spec Mode <system-reminder>
        +
ExitSpecMode / AskUser tool descriptions
        +
Harness approval result after ExitSpecMode
```

The key transition is `ExitSpecMode`. The model submits a plan, the harness presents or saves that plan for user review, and the returned approval result tells the model whether implementation may begin.

---

## Activation Model

Spec Mode is activated by injecting a harness-authored `<system-reminder>` into the model-facing event stream. The reminder may be represented as a user-role content item, but it is control-plane text rather than human-authored user input.

The activation phrase is:

```text
Spec mode is active
```

Tool descriptions also reference this phrase. In particular, `ExitSpecMode` is intended to be called only when Spec Mode is active and the agent has one concrete plan ready for approval.

---

## Verbatim Spec Mode Reminder

```xml
<system-reminder>
Spec mode is active. Do NOT edit files, change configuration, make commits, issue writes to external systems (e.g., Linear/GitHub/Slack API writes, starting services or processes), or otherwise mutate the repo or system state until the user approves the spec. Read-only tools remain available: read files, run non-mutating commands, and fetch linked artifacts (tickets, bug reports, logs/traces, Sentry/Axiom, Slack threads, linked PRs, design docs), subject to autonomy and sandbox rules.

When your plan is ready, present it by calling ExitSpecMode; the user will be prompted to confirm or edit.

Use the AskUser tool to gather requirements, clarify decisions, and choose among viable implementation approaches before finalizing your spec. If there are several equally strong alternatives, or if the user asks for options to go through, review, or choose from, first output the options with enough detail for the user to compare them, then call AskUser with a concise choice prompt containing only the option labels. Do NOT call ExitSpecMode with a plan that still lists multiple unresolved options. After the user chooses, call ExitSpecMode with one concrete plan based on that choice.

When your spec involves architecture, data flows, state machines, or complex interactions, include Mermaid diagrams (using ```mermaid code blocks) in your plan to visualize the design. Only include diagrams when they add clarity -- not for simple or linear changes. Keep participant/node names short (under ~20 chars) so diagrams render as ASCII art in the terminal. Use short aliases and add a legend comment below the diagram if full names are needed. Only use these supported diagram types: flowchart/graph, stateDiagram, sequenceDiagram, classDiagram, erDiagram, xychart-beta. Do NOT use gantt, pie, gitGraph, mindmap, timeline, journey, quadrantChart, sankey, or block diagrams as they cannot be rendered.
</system-reminder>
```

---

## Behavior Contract

Spec Mode creates a three-party contract between the model, harness, and user.

### Model responsibilities

- Inspect the repository before planning.
- Ask clarifying questions when requirements are ambiguous.
- Resolve competing approaches before finalizing the spec.
- Produce one concrete implementation plan.
- Call `ExitSpecMode` instead of editing directly.
- Implement only after an approval result says implementation may continue.

### Harness responsibilities

- Inject the Spec Mode reminder.
- Provide `ExitSpecMode` and `AskUser` as interactive control tools.
- Present the plan for user approval or editing.
- Persist the approved spec.
- Return structured approval state to the model.
- Replay the approval state in later turns so the model can continue coherently.

### User responsibilities

- Approve, edit, or reject the plan.
- Resolve implementation blockers if the agent cannot safely choose a path.

---

## Allowed and Forbidden Behavior Before Approval

Allowed before approval:

- read files,
- list directories,
- search source code,
- run non-mutating inspection commands,
- fetch linked artifacts,
- ask focused questions,
- maintain a todo list,
- draft the implementation plan.

Forbidden before approval:

- edit files,
- change configuration,
- make commits,
- write to external systems,
- start long-lived services/processes,
- otherwise mutate repository or system state.

This boundary is primarily instruction-level unless the harness additionally disables mutation-capable tools. Architecture should therefore not rely on the reminder alone for hard security boundaries.

---

## ExitSpecMode Tool

`ExitSpecMode` is the planning-to-implementation handoff.

| Field | Required | Meaning |
|---|---:|---|
| `plan` | yes | Concrete markdown plan to show to the user. |
| `title` | no | Short title used for the saved spec artifact. |

The submitted plan should be specific enough for implementation. It should not contain unresolved alternatives. If multiple viable approaches remain, the agent should first call `AskUser` with a concise choice prompt, then submit the selected plan through `ExitSpecMode`.

A good plan usually includes:

1. files/modules to add or modify,
2. data model or configuration shape,
3. control-flow or UI changes,
4. compatibility/security considerations,
5. tests to add or update,
6. validation commands to run.

---

## Approval Handoff Result

After `ExitSpecMode`, the harness returns structured state to the model. Approval output has this shape:

```json
{
  "approved": true,
  "message": "Spec mode exited. The user approved the plan. Continue with the implementation.",
  "filePath": "/path/to/saved/spec.md",
  "isEdited": false
}
```

Field meanings:

| Field | Meaning |
|---|---|
| `approved` | Whether implementation may begin. |
| `message` | Human-readable instruction or status. |
| `filePath` | Saved spec artifact path. |
| `isEdited` | Whether the user edited the plan before approval. |

The model should treat `approved: true` as the implementation gate. If `isEdited: true`, the saved spec should be treated as authoritative.

---

## Handoff Lifecycle

```text
User requests a coding change
        ↓
Harness activates Spec Mode
        ↓
Model performs read/search reconnaissance
        ↓
Model asks clarifying questions if needed
        ↓
Model forms one concrete implementation plan
        ↓
Model calls ExitSpecMode(plan, title)
        ↓
Harness presents/saves plan for review
        ↓
User approves, edits, or rejects
        ↓
Harness returns structured result
        ↓
If approved: model implements and validates
If not approved: model revises, asks, or stops
```

---

## AskUser vs ExitSpecMode

| Tool | Purpose | When used |
|---|---|---|
| `AskUser` | Transfer a focused decision or blocker to the user. | Before spec submission for ambiguity; after approval for operator-level blockers. |
| `ExitSpecMode` | Transfer a complete implementation plan to the approval UI. | Once one concrete plan is ready. |

`AskUser` is not a replacement for `ExitSpecMode`. It resolves a question. `ExitSpecMode` exits the planning gate.

---

## Message/Event Representation

Spec Mode runs on the same cumulative event stream as other interactive sessions. The request history can contain:

| Event kind | Meaning |
|---|---|
| `role: user` | Human messages and harness-authored reminders |
| `type: reasoning` | Prior model reasoning item, sometimes summarized or encrypted |
| `type: function_call` | Historical function-tool call |
| `type: function_call_output` | Historical function-tool result |
| `type: custom_tool_call` | Historical custom-tool call, e.g. patch application |
| `type: custom_tool_call_output` | Historical custom-tool result |

Later turns replay prior events. Implementations and event processors should deduplicate calls by stable ids such as `call_id` rather than treating each request as a fresh delta.

---

## Typical Tool Surface

Spec Mode commonly uses the ordinary interactive tools plus `ExitSpecMode`:

```text
Read, LS, Execute, ApplyPatch, Grep, Glob, ExitSpecMode, AskUser,
WebSearch, TodoWrite, FetchUrl, ToolSearch, Skill, Task
```

Expected phase split:

| Phase | Dominant tools |
|---|---|
| Planning | `Read`, `LS`, `Grep`, `Glob`, `TodoWrite`, `AskUser` |
| Handoff | `ExitSpecMode` |
| Implementation | `ApplyPatch`, `Read`, `Execute`, `TodoWrite` |
| Validation | `Execute`, `Read` |
| Finalization | `TodoWrite`, assistant final response |

`ApplyPatch` may appear as a custom-tool channel rather than as an ordinary function call. Message processors should handle both ordinary function calls and custom tool calls.

---

## Relationship to Other Handoffs

| Handoff type | Tool/mechanism | Purpose |
|---|---|---|
| Spec approval handoff | `ExitSpecMode` | Planning -> implementation after user approval. |
| User clarification handoff | `AskUser` | Agent -> user decision, then resume. |
| Subagent handoff | `Task` result | Subagent -> parent agent report. |
| Mission worker handoff | `EndFeatureRun` | Worker -> mission orchestrator completion state. |

---

## Edge Cases

### Ambiguous requirements

Use `AskUser` before `ExitSpecMode`.

### Multiple viable approaches

Present options, ask the user to choose, then submit the chosen plan only.

### Edited plan

If the approval output indicates the spec was edited, implement the edited saved spec.

### Rejected plan

Do not implement. Revise, ask follow-up questions, or stop according to the returned message.

### Implementation blocker

Use `AskUser` when the blocker requires user/operator judgment rather than unilateral agent action.

---

## Design Implications

- Spec Mode is a state overlay, not a separate agent type.
- The hard boundary is the approval result, not the model's plan text.
- The saved spec artifact is part of the audit trail and should be durable.
- Reminder-based restrictions should be supplemented with harness-side policy where mutation must be impossible before approval.
- Message processors must distinguish harness-authored reminders from human user input even when both are represented with user-role events.
