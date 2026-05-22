# 06 — Security Model

---

## Purpose

The harness security model protects the repository, local system, user data, external services, and model context from unsafe tool use, prompt injection, accidental disclosure, and uncontrolled persistence.

Security is layered. Some controls are hard harness policies; others are model-facing instructions that should be backed by enforcement when the risk is high.

---

## Trust Boundaries

```text
User
  ↓
Harness CLI / API
  ↓
Prompt assembler ─────── Tool router / policy engine
  ↓                         ↓
Model provider          Filesystem / Shell / Web / External APIs
  ↓                         ↓
Model output  ───────── Tool results
```

Primary trust boundaries:

- user ↔ harness,
- harness ↔ model provider,
- model ↔ tool router,
- tool router ↔ filesystem/shell/network,
- parent agent ↔ subagent,
- local context ↔ persisted logs,
- harness ↔ proxy/model-routing layer.

---

## Tool Risk Classes

| Class            | Examples                       | Risk                                             |
| ---------------- | ------------------------------ | ------------------------------------------------ |
| Read-only local  | `Read`, `LS`, `Grep`, `Glob`   | Data exposure, prompt injection from files.      |
| Local mutation   | `Edit`, `Create`, `ApplyPatch` | Code corruption, unauthorized changes.           |
| Shell execution  | `Execute`                      | Data loss, exfiltration, long-running processes. |
| External content | `WebSearch`, `FetchUrl`        | Prompt injection, SSRF/private URL access.       |
| User handoff     | `AskUser`, `ExitSpecMode`      | Misleading prompts, approval confusion.          |
| Subagent         | `Task`, `EndFeatureRun`        | Context leakage, scope drift.                    |
| Skill/meta       | `Skill`, `GenerateAgent`       | Untrusted procedure injection.                   |

---

## Risk Classification

Shell commands and other risky actions may carry model-generated risk labels. These labels are useful for visibility but should not be the only enforcement mechanism.

Recommended policy:

| Risk     |                  Allowed automatically? | Notes                                                                    |
| -------- | --------------------------------------: | ------------------------------------------------------------------------ |
| Low      |                                 Usually | Read-only commands, harmless inspection.                                 |
| Medium   |                    Usually with logging | Tests, builds, reversible local changes.                                 |
| High     | Require explicit policy or confirmation | Deletes, migrations, network writes, credential access.                  |
| Critical |                        Block by default | Destructive broad commands, secret exfiltration, unsafe external writes. |

The harness should enforce policy server-side for high-risk operations rather than trusting model self-classification.

---

## Spec-Mode Mutation Boundary

Spec Mode forbids mutation before approval. If this boundary matters for security, the harness should enforce it by policy:

- block edit/create/patch tools before approval,
- block mutating shell commands before approval,
- block external write APIs before approval,
- allow read/search/fetch only within configured policy.

The model-facing reminder is valuable but should be treated as defense-in-depth, not the hard control.

---

## Data Persistence

Agent sessions can contain sensitive material:

- source code,
- secrets in files or logs,
- command outputs,
- reasoning summaries,
- user instructions,
- tool results,
- external artifact contents.

Persistence policy should define:

- what is logged,
- retention duration,
- who can access logs,
- whether prompts/responses are stored,
- redaction rules,
- deletion procedures.

Prompt and response logs are high-sensitivity artifacts because they may contain everything the model saw or produced. A proxy or spend-log database that stores full prompts becomes a sensitive data repository, not just an observability backend.

---

## Prompt Injection

Prompt injection can enter through:

- repository files,
- fetched URLs,
- issue trackers,
- logs/traces,
- Slack/Linear/GitHub content,
- generated skills or agent definitions.

Model instruction hierarchy should be explicit:

```text
System/developer/harness instructions > user instructions > external content
```

External content should be treated as data unless the user explicitly authorizes following it.

---

## Subagent Context Leakage

Subagents should receive scoped context, not the entire parent history by default.

Controls:

- minimize worker prompts,
- redact secrets,
- pass only relevant files/requirements,
- define non-goals,
- require structured handoff,
- avoid exposing unrelated user data.

---

## Provider Trust

The model provider and any proxy/router may receive prompts, code snippets, command output, and reasoning context. Provider and proxy policy should be reviewed for:

- data retention,
- training use,
- encryption,
- region/compliance requirements,
- incident response,
- access controls.

Sensitive deployments should support local or private-model fallback where appropriate. A single external provider is both an availability dependency and a trust boundary.

---

## Network and URL Safety

`FetchUrl` and similar tools should block or require approval for:

- localhost,
- private IP ranges,
- cloud metadata endpoints,
- internal corporate hostnames,
- file URLs,
- credential-bearing URLs,
- redirects into blocked ranges.

Fetched pages should not be allowed to override harness instructions.

---

## Threat Summary

| Threat                          | Impact                  | Mitigation                                         |
| ------------------------------- | ----------------------- | -------------------------------------------------- |
| Destructive command             | Data loss               | Command policy, risk gates, sandboxing.            |
| Secret exposure in logs         | Credential compromise   | Redaction, limited logging, retention controls.    |
| Prompt injection from files/web | Wrong or unsafe actions | Instruction hierarchy, content isolation.          |
| Unauthorized pre-spec edit      | Scope violation         | Enforce mutation gate in tool router.              |
| Subagent overexposure           | Data leakage            | Scoped prompts, reduced privileges, and redaction. |
| External API write              | Unintended side effects | Approval gates and allowlists.                     |

---

## Design Guidance

- Prefer hard policy for high-impact actions.
- Keep logs minimal and redact aggressively.
- Treat external text as untrusted data.
- Sandbox shell execution and subagent workers where possible.
- Keep approval states explicit and durable.
- Audit tool calls, not just assistant messages.
