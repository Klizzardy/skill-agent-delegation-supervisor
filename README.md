English | [简体中文](./README.zh-CN.md)


# Agent Delegation Supervisor — Reference Guide

## What This Skill Does

This skill is for a **supervising agent**. When a user asks it to hand off scoped research, code search, implementation, testing, debugging, documentation, or bounded operations checks to another execution agent, the supervising agent owns boundaries, high-risk decisions, and acceptance. This skill provides rules and templates to make delegation low-token, low-rework, and boundary-safe.

Core principle: **The main agent owns boundaries and acceptance. The delegate owns implementation. The delegate handles the primary implementation; the main agent may directly complete small, clear-cut acceptance fixes. The delegate does not make architectural decisions for the main agent.**

## When Delegation Makes Sense

- Medium-scope implementation tasks (2-5 files, contained within one subsystem)
- Search-heavy investigation tasks that require reading multiple files to answer a question
- Writing or updating tests after the main agent has confirmed the interface signatures
- The user explicitly asks to have another agent execute the work

## When to Skip Delegation

- Single-file, single-digit-line changes (the main agent edits faster directly)
- Cross-cutting changes across 5+ subsystems (the main agent must hold the full picture)
- Tasks touching secrets, auth configuration, CI/CD, or deployment scripts that the user has NOT explicitly scoped. When the user explicitly requests those files, delegation is allowed but must be constrained to the specified files or keys; unauthorized entries must be preserved untouched.
- The token cost of loading context into the delegate exceeds doing the work directly

## Why This Reduces Token Usage and Rework

1. **Main agent context stays lean**: The delegate's file reading and searching happens in an isolated session. The main agent only processes the diff review and acceptance report.
2. **Task packet as contract**: The SCOPE field uses precise glob patterns. The delegate has no excuse for touching files outside the boundary.
3. **Read Before Change (Risk-Tiered)**: This rule blocks the #1 source of rework — the delegate guessing filenames and endpoints without reading code. For normal scoped work, the delegate reads context, implements, and tests in a single execution, reporting findings with diff/test evidence. For high-risk tasks touching config/deploy, public API/db schema, auth/credential boundaries, shared file conflicts, or scope ambiguity, the delegate must stop after reading — report only, wait for main-agent confirmation before modifying.
4. **Four checkpoints**: Checkpoint 3 (Pre-merge) is a universal hard stop. Checkpoint 2 (Post-read) is a hard stop for high-risk tasks; for normal work it is integrated into the delegate's single execution. Problems surface at the earliest possible stage.

## Compact Task Brief Template

Copy and fill in this template for every delegation:

```
TASK: [One sentence — what to produce, not how to produce it]
SCOPE:
  - Allowed paths: [glob patterns, e.g. src/guestbook/**/*.py]
  - Forbidden paths: [e.g. config/**, deploy/**, **/*.env, **/credentials*]
  - Do NOT touch: [specific files or subsystems explicitly off-limits]
CONTEXT FILES: [2-5 file paths the delegate should read first]
DISCOVERY ROOTS: [Only when exact files are not yet known; narrow read-only directories supplied by the main agent]
EXISTING BEHAVIOR: [1-2 sentences describing current relevant behavior]
ACCEPTANCE: [Concrete, verifiable outcome — e.g. "curl /api/guestbook returns 200 with JSON array"]
CONSTRAINTS:
  - Do NOT change API signatures of existing public functions.
  - Do NOT add dependencies without explicit approval.
  - Keep changes within the allowed paths.
```

### Filling It Out Correctly

- **SCOPE** that is too broad is useless. `src/**` is not a valid scope — it means the main agent hasn't thought through the boundaries.
- **CONTEXT FILES**: Pick 2-5 critical files. Too many and the delegate gets lost; too few and the delegate starts guessing.
- **ACCEPTANCE** must be executable. "The code looks correct" is not an acceptance criterion. Use a curl command, a test command, or a specific UI behavior description.

## Delegation Prompt Template

```
You are receiving a scoped coding task. Follow these rules in order:

1. First, read every file listed in CONTEXT FILES. Report what you found with filenames and line numbers.
2. Propose changes based only on what you actually read — do not guess.
3. Make changes only within the SCOPE allowed paths.
4. After completing, self-check against ACCEPTANCE and report results.
5. If you need to modify files outside SCOPE or add a dependency, stop and report first. Wait for approval.
6. Risk gating: If the task touches existing user config/deploy files, public API or database schema changes, auth/credential boundaries, shared files that may conflict with concurrent tasks, or if the SCOPE is ambiguous — you MUST stop after reading, report findings and plan only, and wait for main-agent confirmation before implementing. For normal scoped work with clear boundaries and none of the above risks, you may complete read, implement, and test in a single execution.

Here is the task packet:
[Paste task packet]
```

## Acceptance Report Template

After completing the task, the delegate produces a short acceptance report. The main agent uses this at checkpoints 3 and 4:

```
### Files Modified
- src/api/routes.py:42-68 — Added POST /api/guestbook endpoint
- src/models/guestbook.py:15-30 — Added GuestbookEntry dataclass

### Files Read But Not Modified (confirmed no changes needed)
- src/db/connection.py — Reuses existing connection pool, no changes needed

### Self-Check Results
- [x] curl -X POST /api/guestbook -d '{"msg":"test"}' returns 201
- [x] curl /api/guestbook returns JSON array containing the posted message
- [x] Existing tests in src/tests/test_routes.py all pass

### Unmet Acceptance Items
- None
```

## Low-Token Interaction Rule

When the user explicitly requests direct execution (e.g., "just do it," "go ahead," "implement now"), the main agent still prepares a compact task packet internally, but sends only a short status update to the user before launching the delegate:

> "Delegating [task] to implementation agent. Will report back with results."

Do NOT expand the full task brief, context file list, or scope details in the chat unless:
- The task involves gated high-risk boundaries that require the user's decision
- The user specifically asks for the plan before execution

This keeps the user-facing conversation lean while maintaining internal rigor.

## Common Failure Patterns and How to Fix Them

### Failure 1: Guessing File Names and Endpoints

**What it looks like**: The delegate, without reading any code, claims "The guestbook feature lives in `php/guestbook.py` with `/submit` and `/list` endpoints."

**Root cause**: The task was not classified for risk; the delegate was not instructed to report findings before implementing. The delegate relied on pattern-matching from past experience rather than reading actual code.

**Fix**: Classify the task's risk tier before delegation. Intercept at Checkpoint 2 (Post-read) — require the delegate to read the context files first and report line numbers. Reject all output until this is done. For high-risk tasks, enforce the gated stop; for normal tasks that still guess, pull the task back — the main agent handles it directly.

The same rule applies to repository status, configuration shape, and file existence: do not assert them without user-provided or read-only evidence. If exact files are not known, the main agent supplies narrow discovery roots; do not authorize an unbounded filesystem search.

If even the permitted project roots are unknown, the main agent performs bounded discovery or obtains the location first. Do not dispatch invented modification scope, commit references, or off-limits file lists.

### Failure 2: Modifying Unauthorized Config Files

**What it looks like**: The delegate "helpfully" updates `package.json`, `tsconfig.json`, or a formatter config while implementing the actual feature.

**Root cause**: The SCOPE "forbidden paths" didn't cover these files, or the delegate's auto-formatter triggered the changes.

**Fix**: At Checkpoint 3 (Pre-merge), review every file in the diff against the SCOPE. Reject changes outside SCOPE. Explicitly list common off-limits files in the "Do NOT touch" field: `package.json`, `.prettierrc`, `.eslintrc.*`, `tsconfig.json`, `.editorconfig`.

### Failure 3: Leaking Secrets or Sensitive Data

**What it looks like**: The task packet, prompt, or delegate output contains a database connection string, API key, or internal hostname.

**Root cause**: The main agent did not check for sensitive data at Checkpoint 1 (Pre-delegation), or wrote real values into the task packet.

**Fix**: Add a sensitive-data check at Checkpoint 1. Never write real keys, passwords, or connection strings into task packets, prompts, or documentation. Delegates may read configuration files locally to verify structure (key names, types) but must not reproduce real values in their output. Never modify source configuration files for the purpose of sanitization — describe config structure abstractly instead.

### Failure 4: Claiming Token Savings Without Evidence

**What it looks like**: After delegation, claiming "saved 50% token usage" with no measurement.

**Root cause**: Subjective impression substituted for measurement.

**Fix**: Follow the Token Honesty rule. Without measurement, describe qualitatively: "This delegation kept 3 large files (~2000 lines) out of the main agent's context. The main agent only processed a ~150-line diff review."

Keep two accounting questions separate: delegation may save main-agent quota and context, while increasing total combined tokens across supervisor and delegate sessions. Never claim overall savings unless both sides were measured.

When an execution agent runtime supports structured usage output such as JSON, record input tokens, output tokens, cost, and acceptance result for each substantive delegation. Do not start a fresh high-overhead delegate session for tiny wording fixes or one-point patches; batch related work into one compact, bounded task instead.

## How to Publish and Use

1. Place this directory (`agent-delegation-supervisor/`) in the skills directory supported by the agent runtime you use, for example `.codex/skills/` or `.claude/skills/`.
2. The main agent loads this skill when it receives a qualifying delegation request.
3. Follow the four-checkpoint workflow in SKILL.md.
4. When the user's language is not Chinese, the main agent loads this file (`references/guide.en.md`) for templates and failure patterns.
5. This skill contains no project-specific data and is safe to publish directly to GitHub.
