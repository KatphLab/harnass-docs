# 01 — Prompt Architecture

---

## Purpose

The prompt layer defines the harness operating contract. It tells the model what role it is playing, which mode it is in, how much autonomy it has, how to use tools, how to verify work, and how to terminate.

The harness uses a small set of prompt variants rather than one monolithic prompt. Each variant is selected for a runtime purpose: autonomous execution, interactive CLI work, mission workers, summaries, or title generation.

---

## Prompt Stack

A model request is assembled from several prompt-like layers:

```text
Base mode prompt
    +
Mode-specific suffixes
    +
Tool descriptions
    +
Dynamic system reminders
    +
Conversation/event history
```

The base prompt establishes the agent identity and operating mode. Tool descriptions act as secondary prompts because they contain detailed usage rules. Dynamic reminders add turn-specific constraints such as spec-mode restrictions, diagnostics, or contextual warnings.

---

## Prompt Variants

| Variant | Runtime role | Key behavior |
|---|---|---|
| Exec Mode | Autonomous task execution | Never ask the user; continue until the task is complete and verified. |
| Exec Mission Worker | Autonomous subagent inside a mission | Follow assigned skill/procedure and return structured handoff. |
| Interactive Mode | Human-in-the-loop CLI work | May ask focused questions and delegate subagents. |
| Interactive Mission Planning | Human-facing mission design | Helps define, split, and launch mission work. |
| Summary Generator | First-pass conversation compression | Produces structured state summaries. |
| Summary Updater | Incremental summary maintenance | Updates existing summaries after more work. |
| Title Generator | Conversation metadata | Produces short session titles. |

---

## Exec Mode Contract

Exec Mode is the high-autonomy prompt. The model is instructed to complete and verify the user request without interactive help.

Core rules:

- Do not ask the user for confirmation.
- Use tools whenever needed.
- Keep going until the requested work is done.
- Do exactly what was asked: no less and no unrelated extras.
- Inspect existing code structure before editing.
- Match local style, libraries, and conventions.
- Run the relevant tests/checks before final response unless explicitly told not to.
- Do not expose secrets or sensitive data.
- Do not give up on unexpected errors; debug systematically.

Exec Mode is appropriate for tasks where the harness expects full autonomy and no UI decision points.

---

## Interactive Mode Contract

Interactive Mode keeps most engineering discipline from Exec Mode but allows user interaction.

Interactive-specific capabilities:

- `AskUser` for focused clarifications or blocked decisions.
- `Task` for subagent delegation.
- skill invocation for specialized procedures.
- dynamic UI-oriented workflows such as spec approval.

Interactive Mode should still avoid unnecessary questions. It asks only when the answer materially affects implementation or when an operator-level blocker prevents safe progress.

---

## Mission Worker Prompting

Mission workers are specialized Exec sessions spawned by an orchestrator. Their prompt adds a critical-skill section:

- The initial user message names required skills.
- The worker must invoke and follow those skills.
- If a required skill is unavailable, the worker must return to the orchestrator instead of improvising.
- Completion must happen through `EndFeatureRun` with structured evidence.

This converts a general coding agent into a bounded worker with a defined assignment and exit protocol.

---

## Summary and Title Prompts

Summary and title prompts are narrow metadata prompts. They should not perform code work.

Summary prompts produce durable task state:

1. chronological progress,
2. technical work completed,
3. files/sections involved,
4. errors and resolutions,
5. pending work,
6. decisions,
7. architecture changes,
8. dependency changes,
9. testing status,
10. next steps.

Title prompts produce a short label, typically 5–8 words, for UI/session organization.

---

## Dynamic System Reminders

System reminders are harness-authored instructions injected into the event stream. They can appear alongside user-role content even though they are not human-authored requests.

Reminder examples:

- spec-mode read-only constraints,
- diagnostics from failed commands,
- todo-list nudges,
- contextual warnings,
- tool-use or safety hints.

Reminders are part of the control plane. Message processors and auditors should distinguish them from user intent.

---

## Tool Descriptions as Prompt Layer

Tool schemas do more than validate arguments. Their descriptions encode behavior:

- when to use the tool,
- what risks to consider,
- required preflight checks,
- output expectations,
- examples of valid/invalid use.

This means changing a tool description can materially change model behavior even if the JSON schema stays the same.

---

## Behavioral Mechanisms

The prompt architecture works through several repeated mechanisms:

| Mechanism | Effect |
|---|---|
| Identity anchoring | The model consistently acts as a software engineering agent. |
| Imperative redundancy | Important rules appear in prompt, reminders, and tool descriptions. |
| Verification contract | The model is conditioned to test before completion. |
| Structured handoff | Tools such as `ExitSpecMode` and `EndFeatureRun` force explicit transitions. |
| Risk reflection | Shell execution requires risk labels and reasons. |
| Scope discipline | The model is told to do exactly the requested work. |

---

## Conceptual Diagram

```text
User request
    ↓
Prompt selector
    ↓
Base mode prompt ─────┐
Mode suffixes ────────┤
Tool descriptions ────┤──> Model request
System reminders ─────┤
Event history ────────┘
    ↓
Model reasoning / tool call / answer
```

---

## Design Guidance

- Keep base prompts stable and move transient constraints into reminders.
- Put tool-specific rules in tool descriptions, not only in the system prompt.
- Use explicit handoff tools for state transitions rather than relying on prose.
- Treat reminders as authoritative control messages.
- Keep summary/title prompts narrow so they cannot accidentally perform agentic work.
