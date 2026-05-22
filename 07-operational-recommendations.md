# 07 — Operational Recommendations

---

## Purpose

This document gives practical guidance for running, building, and auditing the harness in production or security-sensitive environments.

---

## For Harness Operators

### Minimize Prompt/Response Logging

Store the least conversation data required for debugging and compliance.

Recommended controls:

- disable full prompt logging by default,
- redact secrets before persistence,
- define retention windows,
- restrict log access,
- separate operational metrics from sensitive content.

### Enforce Tool Policy Server-Side

Do not rely only on model-provided risk labels or reminders.

Enforce:

- blocked destructive shell patterns,
- approval for high-risk commands,
- no mutation during Spec Mode before approval,
- URL/network allowlists and denylists,
- external write restrictions,
- repository path boundaries.

### Use Sandboxing

Run shell commands with isolation:

- dedicated working directory,
- resource limits,
- timeout limits,
- no ambient credentials unless required,
- restricted network where possible,
- clear process cleanup.

### Protect Secrets

- Scope API keys to minimum permissions.
- Rotate keys regularly.
- Avoid injecting secrets into model-visible context.
- Redact secrets from tool output.
- Block commands that print known secret env vars.

### Monitor Long Sessions

Long conversations increase risk and cost.

Recommended controls:

- token budget warnings,
- compaction thresholds,
- maximum tool-call depth,
- runaway command detection,
- final validation requirements.

---

## For Harness Builders

### Separate Control Plane from User Input

Represent harness reminders distinctly from human messages in internal state, even if the model API uses the same role field.

### Keep Handoffs Structured

Use explicit tools for state transitions:

- `ExitSpecMode` for plan approval,
- `AskUser` for user decisions,
- `Task` for subagent delegation,
- `EndFeatureRun` for worker completion.

Avoid relying on free-form assistant prose for critical state changes.

### Design Tool Schemas Carefully

Tool schemas should include:

- required fields,
- strict enums where possible,
- clear descriptions,
- preflight requirements,
- safety warnings,
- examples for ambiguous parameters.

Schema descriptions are part of the prompt surface and should be versioned like code.

### Implement Context Compaction

Compaction should preserve:

- current objective,
- approval/handoff state,
- todos,
- files changed,
- validation status,
- unresolved blockers.

Do not compact away the fact that a spec was approved or that a user decision is pending.

### Support Custom Tool Channels

The message layer should support both ordinary function calls and custom tool calls. Patch application, UI interactions, and provider-specific features may not fit a single function-call representation.

### Build for Mode-Specific Policy

Tool visibility and permission should depend on mode:

| Mode | Recommended policy |
|---|---|
| Exec | Allow mutation with risk controls. |
| Interactive | Allow mutation with user/harness policy. |
| Spec | Read-only until approval. |
| Mission worker | Scope to assigned repo/task. |
| Summary/title | No tools or only read-only context. |

---

## For Security Auditors

Audit the actual tool stream, not only the final answer.

Checklist:

- [ ] Are high-risk commands blocked or approved?
- [ ] Are mutation tools disabled before Spec Mode approval?
- [ ] Are prompts/responses logged? If yes, where and for how long?
- [ ] Are secrets redacted from tool outputs?
- [ ] Are external URLs validated against SSRF/private-network rules?
- [ ] Are subagent prompts scoped and redacted?
- [ ] Are handoff results durable and unambiguous?
- [ ] Are final answers consistent with validation results?
- [ ] Can long conversations be compacted safely?
- [ ] Are tool schemas versioned and reviewed?

---

## Production Defaults

Recommended baseline:

1. prompt logging off or redacted,
2. high-risk shell commands blocked by policy,
3. Spec Mode mutation gate enforced by router,
4. URL fetching denies private/local addresses,
5. subagents receive minimal scoped context,
6. summaries preserve active state,
7. all tool calls are audit logged with redaction,
8. final responses include validation status.
