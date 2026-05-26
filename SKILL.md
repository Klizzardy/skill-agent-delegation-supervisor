---
name: agent-delegation-supervisor
description: Use when a supervising agent delegates scoped project work to another agent while retaining boundaries, risk decisions, and final acceptance responsibility.
---

# Agent Delegation Supervisor

## When Not Worth Delegating

Skip delegation when:
- The change is a single-file, single-digit-line edit the main agent can apply directly.
- The task requires cross-cutting architectural judgment across 5+ subsystems that only the main agent holds.
- The task touches secrets, auth configuration, CI/CD pipelines, or deployment manifests that the user has NOT explicitly scoped. When config/deploy files are user-requested scope, delegation is allowed but must be constrained to the specified files or keys; unauthorized entries must be preserved untouched.
- Loading the delegate with sufficient context would cost more tokens than doing the work directly.

## Read Before Change (Risk-Tiered)

The delegate MUST read relevant existing code and report what it found before making any change. This is the single highest-leverage rule for avoiding rework. The stop-after-read requirement is risk-tiered, not blanket.

### Normal Scoped Work

For tasks within well-defined boundaries (clear SCOPE, no config/deploy/auth files, no shared-file conflicts, no scope ambiguity), the delegate reads the context files, implements the changes, runs tests, and reports findings with diff/test evidence — all in a single execution. Do not add a round-trip just for formality.

### Gated High-Risk Work

When the task touches any of the following, the delegate MUST stop after reading and wait for main-agent confirmation before modifying:

- Existing user configuration or deployment files
- Public API signatures or database schema
- Authentication, credential, or key boundaries
- Files potentially shared with other concurrent tasks
- Scope ambiguity that could lead to boundary violations

In gated mode, the delegate's first response reports only what it read (with file paths and line numbers) plus a brief implementation plan. The main agent confirms or adjusts, then the delegate proceeds.

### Guessing Is Still Rejected

Regardless of risk tier, if the delegate's first response names files or endpoints without citing observed code (line numbers, file paths it actually read), stop the delegation immediately. Require it to read first, then re-engage.

Do not state repository status, file existence, configuration shape, or runtime behavior unless it was provided by the user or observed through an allowed read-only step. If the relevant project root is unknown, ask the main agent for the permitted root instead of issuing a broad filesystem read scope.

Before dispatching any delegate, the main agent must know or discover the permitted project root(s). If roots or existing interface facts are missing, perform a bounded main-agent discovery step or request the missing location; do not invent an implementation scope, commit reference, or off-limits frontend/backend file list.

**Example of a failing response (reject it):**
> "The guestbook likely uses `/submit` and `/list` endpoints in `php/guestbook.py`."

**Example of an acceptable response:**
> "Read `src/api/routes.py:42-68` — the existing endpoints are `POST /api/guestbook` and `GET /api/guestbook?page=1`. The handler uses SQLite via `db.execute()` at line 55."

## Compact Task Packet

Every delegation must carry a self-contained task packet with these fields:

```
TASK: [One sentence — what to produce, not how to produce it]
SCOPE:
  - Allowed paths: [glob patterns, e.g. src/guestbook/**/*.py]
  - Forbidden paths: [e.g. config/**, deploy/**, **/*.env, **/credentials*]
  - Do NOT touch: [specific files or subsystems explicitly off-limits]
CONTEXT FILES: [2-5 file paths the delegate should read first]
DISCOVERY ROOTS: [Only if exact files are not yet known; narrow read-only directories supplied or approved by the main agent]
EXISTING BEHAVIOR: [1-2 sentences describing current relevant behavior]
ACCEPTANCE: [Concrete, verifiable outcome — e.g. "curl /api/guestbook returns 200 with JSON array"]
CONSTRAINTS:
  - Do NOT change API signatures of existing public functions.
  - Do NOT add dependencies without explicit approval.
  - Keep changes within the allowed paths.
```

The task packet is the contract. If the delegate violates any SCOPE or CONSTRAINT, the deliverable is rejected.

## Boundary Protection

Before sending the task packet, the main agent MUST:
1. Verify that SCOPE paths do not include configuration files, .env files, credential stores, or deployment scripts unless the user has explicitly requested them. When config/deploy files ARE in the user-requested scope, constrain the delegation to the specific files or keys the task requires; unauthorized entries must be preserved untouched.
2. Agents may read necessary configuration locally to verify structure (key names, types), but MUST NOT copy real keys, passwords, connection strings, internal hostnames, or email addresses into output, prompts, or documentation.
3. Never modify source configuration files for the purpose of sanitization. If a delegate needs to reference config structure, describe it abstractly without revealing actual values.

## Low-Token Interaction Rule

When the user explicitly requests direct execution (e.g., "just do it," "go ahead," "implement now"), the main agent still prepares a compact task packet internally, but sends only a short status update to the user before launching the delegate:

> "Delegating [task] to implementation agent. Will report back with results."

Do NOT expand the full task brief, context file list, or scope details in the chat unless:
- The task involves gated high-risk boundaries that require the user's decision
- The user specifically asks for the plan before execution

This keeps the user-facing conversation lean while maintaining internal rigor.

## Parallel Delegation Rules

When delegating multiple tasks in parallel:
- Each task packet must target disjoint file sets. If two tasks might touch the same file, run them sequentially.
- State the parallel intent explicitly: "These N tasks have no shared files and can run concurrently."
- Merge results in order: apply each delegate's output to a clean copy of the base branch, then verify no conflict.

## Four Verification Checkpoints (Main Agent)

| # | When | Action |
|---|------|--------|
| 1 | **Pre-delegation** | Confirm the task packet is complete, SCOPE boundaries are tight, and no sensitive values are copied into prompts or user-visible output. Classify the task as normal or gated high-risk (see Read Before Change). |
| 2 | **Post-read** | Delegate has reported what it read. Verify the report includes real file paths and line numbers. If it guessed, stop and require reading. For gated high-risk tasks, this is a mandatory stop — the main agent must explicitly confirm before the delegate proceeds to implementation. For normal scoped work, the delegate may proceed through implementation in the same execution. |
| 3 | **Pre-merge** | Review the diff. Check: only allowed paths modified, no unrelated refactors, no unauthorized config changes, no new dependencies snuck in. |
| 4 | **Acceptance** | Run the acceptance criteria from the task packet. If it fails, the deliverable is incomplete — do not merge. |

Checkpoint 3 is always a mandatory stop point. Checkpoint 2 is a mandatory stop only for gated high-risk tasks; for normal scoped work it is integrated into the delegate's single execution. Do not proceed past a mandatory checkpoint until it passes.

## Error Correction Loop

When the delegate produces an incorrect deliverable:

1. **Detect** — which checkpoint caught it? (Guessing at #2? Scope violation at #3? Failed acceptance at #4?)
2. **Diagnose** — was the task packet ambiguous? Did the delegate skip reading? Was the scope too loose?
3. **Correct** — choose the proportionate response:
   - **Small, clear-cut fix** (typo, missing null-check, wrong log level): the main agent applies the patch directly, verifies acceptance, and moves on.
   - **Significant or recurring issue** (wrong architecture, repeated scope violation, same failure pattern twice): tighten the task packet at the point of failure and re-delegate, or the main agent takes over the work directly.
4. **Record** — if re-delegating, note what was tightened so the next attempt starts from a stronger contract.

## Token Honesty

Do not claim specific token savings ("saved 40% tokens") unless you have measured both the delegated and non-delegated approaches on the same task. Without measurement, state qualitatively: "Delegation avoided loading [X large files] into the main context" or "The main agent processed only the diff review, not the full implementation."

Distinguish **main-agent quota/context savings** from **total combined token use**. Delegation can reduce expensive main-agent context while increasing total tokens across the supervisor and delegate sessions.

When the delegate runtime exposes structured usage or cost output, enable it for substantive delegated runs and record per-run input tokens, output tokens, cost, and acceptance result. Avoid launching a fresh delegate session for tiny corrections when its fixed context startup cost is likely larger than the patch; batch related work into one tightly scoped task packet instead.

## Language-Aware References

When the user's primary language is Chinese, read `references/guide.zh-CN.md` for detailed templates and incident patterns. For all other users, read `references/guide.en.md`. Do not load both — pick one based on the active conversation language.
