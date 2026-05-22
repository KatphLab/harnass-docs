# 08 — Implementation Plan

---

## Goal

Implement the harness architecture as a reusable pi extension package. The extension should provide mode-aware prompting, hardened tools, spec approval, mission/subagent orchestration, structured handoffs, dynamic reminders, and safe conversation compaction.

Target platform: `@earendil-works/pi-coding-agent` TypeScript extension API.

---

## Architecture Mapping

| Harness capability | Pi mechanism | Implementation strategy |
|---|---|---|
| Exec Mode | Agent-start hook / prompt extension | Append autonomous execution contract and hide interactive tools. |
| Interactive Mode | Default CLI + custom tools | Add `AskUser`, `Task`, skills, and UI helpers. |
| Spec Mode | Dynamic reminder + tool gate | Inject spec reminder; block mutation until `ExitSpecMode` approval. |
| Mission workers | Subagent process or SDK session | Spawn scoped workers and require `EndFeatureRun`. |
| Hardened shell | Custom/replaced bash tool | Require summary, timeout, risk level, and risk reason. |
| File/search tools | Built-in wrappers or replacements | Add richer descriptions, path policy, and consistent schemas. |
| Todo tracking | Stateful custom tool | Persist task state in session/tool metadata. |
| Skills | Pi skill system | Bridge `Skill` tool to SKILL.md loading. |
| Dynamic reminders | Context injection hook | Add harness-authored `<system-reminder>` blocks. |
| Compaction | Session compaction hook | Produce structured summaries preserving active state. |

---

## Package Structure

```text
pi-harness-extension/
├── package.json
├── README.md
├── src/
│   ├── index.ts
│   ├── config.ts
│   ├── modes/
│   │   ├── exec.ts
│   │   ├── interactive.ts
│   │   ├── spec.ts
│   │   └── mission.ts
│   ├── prompts/
│   │   ├── base.ts
│   │   ├── exec.ts
│   │   ├── interactive.ts
│   │   ├── summaries.ts
│   │   ├── titles.ts
│   │   └── reminders.ts
│   ├── tools/
│   │   ├── read.ts
│   │   ├── ls.ts
│   │   ├── grep.ts
│   │   ├── glob.ts
│   │   ├── edit.ts
│   │   ├── write.ts
│   │   ├── apply-patch.ts
│   │   ├── bash.ts
│   │   ├── todo.ts
│   │   ├── ask-user.ts
│   │   ├── exit-spec-mode.ts
│   │   ├── task.ts
│   │   ├── end-feature-run.ts
│   │   ├── skill.ts
│   │   ├── generate-agent.ts
│   │   ├── web-search.ts
│   │   ├── fetch-url.ts
│   │   └── policy.ts
│   ├── subagents/
│   │   ├── worker.md
│   │   ├── scout.md
│   │   ├── planner.md
│   │   └── reviewer.md
│   ├── compaction/
│   │   └── summary.ts
│   └── ui/
│       ├── ask-user.ts
│       ├── spec-approval.ts
│       └── task-monitor.ts
├── skills/
│   ├── mission-planning/
│   │   └── SKILL.md
│   └── define-mission-skills/
│       └── SKILL.md
└── tests/
    ├── tool-schemas.test.ts
    ├── mode-policy.test.ts
    ├── spec-mode.test.ts
    ├── task.test.ts
    └── compaction.test.ts
```

---

## Phase 1 — Extension Foundation

Deliverables:

- package manifest,
- extension entry point,
- configuration loader,
- mode state storage,
- startup validation,
- base prompt assembly.

Implementation tasks:

1. Define extension config shape.
2. Register lifecycle hooks.
3. Add mode selector state.
4. Add command or settings interface for mode switching.
5. Add test harness for prompt/tool assembly.

---

## Phase 2 — Prompt and Mode Layer

Deliverables:

- Exec prompt module,
- Interactive prompt module,
- Spec reminder module,
- mission-worker suffix,
- summary/title prompts,
- dynamic reminder injector.

Mode policy:

| Mode | Prompt behavior | Tool behavior |
|---|---|---|
| Exec | Autonomous, no user prompts. | Hide or block `AskUser`. |
| Interactive | Human-in-loop allowed. | Enable `AskUser` and `Task`. |
| Spec | Interactive + read-only planning reminder. | Block mutation until approval. |
| Mission worker | Scoped autonomous work. | Require `EndFeatureRun`. |
| Summary/title | Narrow metadata generation. | Disable tools unless needed. |

---

## Phase 3 — Core Tools

Deliverables:

- hardened file/search tools,
- hardened shell tool,
- todo tool,
- web/fetch tools,
- shared policy module.

Tool requirements:

- strict schema validation,
- rich descriptions,
- path normalization,
- clear error messages,
- stable call ids,
- redacted logs,
- mode-aware permission checks.

Shell policy should include:

- required command summary,
- timeout default,
- risk level and reason,
- blocked command patterns,
- sandbox setup,
- output truncation/redaction.

---

## Phase 4 — Spec Mode

Deliverables:

- `ExitSpecMode` tool,
- spec approval UI or callback,
- saved spec artifact writer,
- approval state persistence,
- mutation gate enforcement,
- tests for approved/rejected/edited plans.

Approval state:

```ts
type SpecApproval = {
  approved: boolean
  message: string
  filePath?: string
  isEdited?: boolean
}
```

Behavior:

1. Inject spec reminder.
2. Allow read/search/inspection tools.
3. Block mutation tools before approval.
4. Accept one concrete plan through `ExitSpecMode`.
5. Present plan for user review.
6. Save final spec.
7. Return structured approval result.
8. Enable implementation tools only after approval.

---

## Phase 5 — Subagents and Missions

Deliverables:

- `Task` tool,
- worker prompt templates,
- subagent process/session runner,
- `EndFeatureRun` schema,
- mission metadata storage,
- concurrency limits.

Subagent runner behavior:

1. Build scoped worker prompt.
2. Start isolated worker session/process.
3. Stream or collect worker events.
4. Enforce worker timeout and tool policy.
5. Require structured handoff.
6. Return summary to parent.

Concurrency should default to a small limit, such as four workers, to avoid resource exhaustion and edit conflicts.

---

## Phase 6 — Skills and Agent Generation

Deliverables:

- `Skill` bridge tool,
- skill lookup and loading,
- missing-skill error behavior,
- optional `GenerateAgent` tool,
- project vs user agent scope policy.

Security requirements:

- project-local agents require explicit trust policy,
- generated agents should be reviewed before use,
- skills should be treated as prompt extensions,
- skill content should not override higher-priority harness instructions.

---

## Phase 7 — Compaction and Summaries

Deliverables:

- summary generator,
- summary updater,
- compaction hook,
- state-preservation tests.

The summary must preserve:

- current objective,
- mode and approval state,
- todos,
- files changed,
- validation status,
- blockers,
- next action,
- active subagent results.

---

## Phase 8 — UI Components

Deliverables:

- AskUser UI,
- spec approval/edit UI,
- task monitor,
- JSON or markdown render helpers.

UI should make state transitions clear:

- waiting for user answer,
- waiting for spec approval,
- worker running,
- validation failed,
- final complete.

---

## Phase 9 — Testing and Release

Test coverage:

- tool schema validity,
- mode-specific active tools,
- Spec Mode mutation blocking,
- approval result replay/persistence,
- shell risk policy,
- URL validation,
- subagent handoff schema,
- compaction state preservation,
- final response validation reporting.

Release checklist:

- [ ] Extension loads cleanly.
- [ ] Exec Mode does not expose user-question tools.
- [ ] Interactive Mode supports `AskUser`.
- [ ] Spec Mode blocks mutation before approval.
- [ ] `ExitSpecMode` saves and returns approval state.
- [ ] Shell tool enforces risk policy.
- [ ] `Task` launches isolated workers.
- [ ] `EndFeatureRun` validates structured handoff.
- [ ] Compaction preserves active state.
- [ ] Logs redact sensitive output.

---

## Key Design Decisions

1. **Mode policy belongs in the harness, not only in prompts.** Prompts guide the model; the tool router enforces critical constraints.
2. **Tool descriptions are behavior-shaping prompts.** Rich schemas improve tool selection and safety.
3. **Handoffs should be structured.** Approval, user questions, and worker completion need machine-readable results.
4. **Subagents need scoped context.** Workers should not inherit the full parent conversation by default.
5. **Compaction must preserve state, not just summarize prose.** Approval, todos, and blockers are active control data.

---

## Security Requirements

- Redact prompt/tool logs by default.
- Enforce path boundaries for file tools.
- Block private-network URL fetches.
- Require approval for high-risk shell commands.
- Disable mutation during Spec Mode before approval.
- Keep project-local skills/agents behind trust policy.
- Sandbox worker sessions where possible.
