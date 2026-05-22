# Harness Architecture

This folder documents the harness as an architecture: its operating modes, prompt layers, tools, message flow, safety model, and implementation plan. It is intended to describe how the harness behaves at runtime, not to serve as a dump index or forensic log report.

---

## System Overview

The harness is an agentic orchestration layer around a tool-using LLM. It supports autonomous execution, interactive CLI sessions, spec-gated implementation, and multi-agent mission workflows.

The runtime is organized around five major control surfaces:

1. **Prompt layer** — base operating-mode prompts plus dynamic reminders.
2. **Tool layer** — filesystem, shell, planning, web, skill, and subagent tools.
3. **Message/event layer** — Responses-style event history with tool calls, outputs, reasoning items, and reminders.
4. **Handoff layer** — explicit state transitions such as `ExitSpecMode`, `AskUser`, `Task`, and `EndFeatureRun`.
5. **Safety layer** — risk labels, prompt-injection guardrails, data-persistence controls, and operator policy.

---

## Operating Modes

| Mode                | Purpose                      | User interaction               | Primary handoff                                                |
| ------------------- | ---------------------------- | ------------------------------ | -------------------------------------------------------------- |
| Exec Mode           | Complete a task autonomously | No user prompts                | Final assistant response or `EndFeatureRun` in mission workers |
| Interactive Mode    | CLI-driven agentic work      | May ask user questions         | `AskUser`, final assistant response                            |
| Spec Mode           | Plan before implementation   | User approves/edits spec       | `ExitSpecMode` approval result                                 |
| Missions Mode       | Multi-agent orchestration    | Orchestrator delegates workers | `Task` and `EndFeatureRun`                                     |
| Title/Summary Modes | Metadata maintenance         | None                           | Generated title or summary text                                |

---

## Document Index

| #   | Document                                                         | Focus                                                                                       |
| --- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 01  | [System Prompts](01-system-prompts.md)                           | Prompt variants, mode behavior, dynamic reminders, and behavioral mechanisms                |
| 02  | [Tools](02-tools.md)                                             | Tool taxonomy, schemas, execution semantics, and usage patterns                             |
| 03  | [Missions Mode](03-missions-mode.md)                             | Orchestrator/worker split, mission metadata, task delegation, and worker handoff            |
| 04  | [Reasoning and Behavior](04-reasoning-and-behavior.md)           | Reasoning/event behavior, token growth, caching, compaction, and error recovery             |
| 05  | [Message Flow](05-message-flow.md)                               | Conversation topology, event replay, parallel/sequential tool calls, and lifecycle patterns |
| 06  | [Security Model](06-security-model.md)                           | Risk classification, data persistence, prompt injection, provider trust, and threat matrix  |
| 07  | [Operational Recommendations](07-operational-recommendations.md) | Operator, builder, and auditor guidance for production hardening                            |
| 08  | [Implementation Plan](08-implementation-plan.md)                 | Mapping the architecture into a reusable pi extension package                               |
| 09  | [Spec Mode](09-spec-mode.md)                                     | Spec-mode activation, reminder text, approval gate, saved specs, and handoff mechanics      |

---

## Architectural Notes

- Tool availability is mode-dependent. The architecture supports both ordinary function tools and custom-tool channels such as patch application.
- Dynamic reminders are part of the control plane. They may appear in user-role event slots even though they are harness-authored.
- Message history is cumulative. Later turns can replay prior reasoning, tool calls, tool outputs, and handoff results.
- Safety controls are layered. Some are hard validation rules in the harness; others are model-facing instructions that should be backed by operator policy where possible.
