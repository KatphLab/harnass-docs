# 02 — Tool Architecture

---

## Purpose

The tool layer gives the model controlled capabilities over the repository, shell, web, user interaction, skills, and subagents. Tools are the harness boundary between model intent and real-world effects.

Tools should be understood as architectural capabilities, not just function signatures.

---

## Capability Layers

```text
Model
  ↓ tool call
Tool router
  ↓ validation / policy / execution
Capability adapters
  ↓
Filesystem | Shell | Web | UI | Skills | Subagents | Handoffs
```

Each tool call passes through schema validation and harness execution. Some tools are read-only, some mutate local state, and some transfer control to another actor.

---

## Filesystem and Search Tools

| Tool         | Purpose                                            | Mutation |
| ------------ | -------------------------------------------------- | -------- |
| `Read`       | Read text files and supported media.               | No       |
| `LS`         | List directories with optional ignore patterns.    | No       |
| `Grep`       | Search file contents using ripgrep-like semantics. | No       |
| `Glob`       | Discover paths by glob patterns.                   | No       |
| `Edit`       | Replace text in existing files.                    | Yes      |
| `Create`     | Create/write new files.                            | Yes      |
| `ApplyPatch` | Apply patch-style file edits and creations.        | Yes      |

The architecture encourages a discover-then-inspect flow:

```text
Grep / Glob / LS → Read → Edit / Create / ApplyPatch
```

Mutation tools should be gated by mode policy. For example, Spec Mode allows search/read but forbids edits before approval.

---

## Execution Tool

### `Execute`

`Execute` runs shell commands in isolated processes.

Important semantics:

- working directory and environment changes do not persist between calls unless explicitly encoded in the command,
- commands require a short human-readable summary,
- commands include risk level and risk reason,
- long-running commands need explicit timeout or fire-and-forget behavior,
- output becomes part of the event history,
- descriptions can include directory-verification, path-quoting, git-safety, and PR-safety guidance.

`Execute` is used for:

- tests,
- linters,
- typechecks,
- build commands,
- local diagnostics,
- safe repository inspection.

Risk classification is advisory unless backed by harness-side enforcement.

---

## Planning and Progress Tools

| Tool           | Purpose                                                                             |
| -------------- | ----------------------------------------------------------------------------------- |
| `TodoWrite`    | Maintains explicit task state during multi-step work.                               |
| `ExitSpecMode` | Submits a concrete plan and exits Spec Mode after user approval.                    |
| `AskUser`      | Asks focused multiple-choice questions or transfers a blocked decision to the user. |

`TodoWrite` acts as a lightweight state machine. Tasks move through pending, in-progress, and completed states. It improves continuity and final reporting.

`ExitSpecMode` and `AskUser` are handoff tools. They change who owns the next decision: the model, the user, or the implementation phase.

`AskUser` should be constrained to a small questionnaire, typically 1–4 focused questions. Each question should include enough context for the user to decide, and options should be explicit while still allowing an "own answer" path in the UI.

---

## Web and External Information Tools

| Tool        | Purpose                                             |
| ----------- | --------------------------------------------------- |
| `WebSearch` | Searches the web for external information.          |
| `FetchUrl`  | Retrieves content from a URL subject to validation. |

`FetchUrl` should reject localhost, private IP ranges, link-local addresses, cloud metadata endpoints, and internal corporate infrastructure unless explicitly allowed by policy.

External-content tools need stricter policy than local read tools because they can expose the model to prompt injection, untrusted instructions, private URLs, or sensitive documents.

The model should treat fetched content as data unless the user explicitly authorizes following instructions from that content.

---

## Deferred and Meta Tools

| Tool                              | Purpose                                  |
| --------------------------------- | ---------------------------------------- |
| `ToolSearch`                      | Loads additional tool schemas on demand. |
| `Skill`                           | Invokes named procedural skills.         |
| `GenerateDroid` / `GenerateAgent` | Creates custom Droid/agent definitions.  |

Deferred tooling reduces context pressure by avoiding full schema injection until needed. Skills package repeatable procedures and should be treated as prompt extensions with operational authority.

---

## Subagent and Mission Tools

| Tool            | Purpose                                                        |
| --------------- | -------------------------------------------------------------- |
| `Task`          | Launches a stateless subagent with a scoped assignment.        |
| `EndFeatureRun` | Returns structured mission-worker results to the orchestrator. |

`Task` is parent-to-worker delegation. `EndFeatureRun` is worker-to-orchestrator completion. Task calls should include a short description, a self-contained prompt, and a valid subagent type or custom Droid name selected from the currently available agents.

A worker handoff should include:

- what was implemented,
- what remains,
- files changed,
- tests/checks run with exit codes or observations,
- tests added and cases covered,
- verification evidence,
- blockers or follow-up work,
- whether control should return to the orchestrator.

---

## Tool Selection Model

The harness does not need to hard-code every tool choice. Selection emerges from:

1. base prompt instructions,
2. tool schema descriptions,
3. current event history,
4. tool results and errors,
5. mode-specific reminders.

The model usually operates at intent level:

```text
Need to understand code → Grep/Glob/LS/Read
Need to change code → ApplyPatch/Edit/Create
Need to verify → Execute
Need user decision → AskUser
Need approved plan → ExitSpecMode
Need parallel expertise → Task
```

---

## Custom Tool Channels

Some capabilities may be represented as custom tool calls instead of ordinary function calls. Patch application is a common example.

Message processors should support both:

- ordinary function-call events,
- custom-tool-call events.

They should also deduplicate calls by stable ids because later requests can replay earlier tool events.

---

## Design Guidance

- Keep tool schemas strict enough to reject malformed calls.
- Put operational rules in schema descriptions where the model sees them at call time.
- Enforce high-risk policies server-side when possible.
- Separate read-only tools from mutation tools in policy even if both are visible to the model.
- Treat handoff tools as state transitions, not ordinary utilities.
