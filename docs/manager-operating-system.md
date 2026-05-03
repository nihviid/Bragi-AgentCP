# Manager Operating System (MOS)

**Bragi Agent Control Plane (BACP)**  
**Date:** 2026-05-03  
**Branch:** `agent/bootstrap-discovery`  
**Status:** Specification

---

## 1. Purpose

The Manager Operating System defines how the ChatGPT manager controls the CLI executor through the BACP bridge. The manager owns direction, verification, and quality. The executor owns bounded execution. MOS replaces improvisation with discipline — the manager follows defined rules for task decomposition, quality gates, failure handling, decision routing, and model cost.

Without MOS, the manager is reactive: it responds to executor reports without a framework for what to check, when to intervene, or how to decompose work. MOS makes the manager proactive and consistent.

---

## 2. Manager Role Definition

### The Manager Owns

| Area | Detail |
|------|--------|
| Objective management | Defines what each task achieves and why |
| Task decomposition | Splits large work into bounded, verifiable units |
| Instruction quality | Every instruction has scope, constraints, success criteria |
| Quality gates | Verifies every executor report before next instruction |
| Verification | Checks files changed, tests, build, lint, risks |
| Decision routing | Decides what executor handles, what manager decides, what escalates to human |
| Escalation | Routes to human when security, production, or business decisions are needed |
| Documentation enforcement | Requires doc updates when behavior or workflow changes |
| Cross-project awareness | Considers impact on contracts, APIs, schemas, integration points |
| Model/cost discipline | Routes tasks to appropriate model tier. Conserves Opus for high-value work. |
| Memory management | Instructs compact/clear when context pressure is high |

### The Manager Must Never

- Silently expand a task's scope beyond the original instruction
- Ignore failed tests or assume they pass without checking
- Repeat the same failed attempt without a new hypothesis
- Approve risky work (rollback, production deployment, destructive operations) without human confirmation
- Allow the executor to own architecture direction or project-level decisions
- Bypass human approval for high-risk decisions
- Issue the next task before verifying the previous one

---

## 3. Executor Role Definition

### The Executor Owns

| Area | Detail |
|------|--------|
| Reading instructions | Checks inbox/ via `bacp-bridge next-instruction` |
| Acknowledging | ACKs instructions after reading (`bacp-bridge ack <id>`) |
| Executing tasks | Implements within the scope, constraints, and files specified |
| Reporting | Writes structured reports via `bacp-bridge write-report` |
| Running checks | Executes tests, build, lint as requested |
| Surfacing blockers | Reports what failed and why, with reproduction steps |
| Recommending next steps | Suggests what the manager should do next |

### The Executor Must Never Own

- Project-level direction or roadmap decisions
- Merge or release approval
- Broad architecture decisions (file organization, framework choice, dependency selection)
- Silent scope expansion beyond the instruction's constraints
- Autonomous cross-project work
- Creating or deleting the STOP file

---

## 4. Task Decomposition Rules

### One Task, One Objective

A task should have one clear objective and produce one bounded outcome. If a task naturally splits, the manager should issue separate instructions.

### Split When

- The task affects multiple modules or files in different areas of the codebase
- Tests, documentation, and implementation would all be large in a single pass
- A refactor and a behavior change are combined (these must be separate unless explicitly approved)
- The task spans more than one project

### Every Task Must Include

- A clear objective (what success looks like)
- Scope boundaries (what files or areas are in bounds)
- Constraints (what the executor must not do)
- Success criteria (how the manager will verify completion)

### Examples

| Good | Bad |
|------|-----|
| "Add --json flag to status command. Only modify scripts/bacp-bridge." | "Improve the tool." |
| "Fix metadata display in next-instruction. Only modify cmd_next_instruction function." | "Make the output better." |
| "Create docs/example-instruction.json with the format below." | "Update documentation." |

---

## 5. Manager Instruction Format

Every manager instruction must include these fields. This is the canonical format — deviations should be exceptions with justification.

```
{
  "command": "<command>",
  "instruction": "<task title and description>",
  "project": "<project name>",
  "objective": "<what success looks like>",
  "scope": "<files or areas in bounds>",
  "constraints": ["constraint 1", "constraint 2"],
  "success_criteria": ["criterion 1", "criterion 2"],
  "allowed_files": ["file patterns or categories"],
  "required_checks": ["test", "build", "lint"],
  "report_back": ["what succeeded", "what failed", "files changed"],
  "model_selection": {
    "manager_model": "<model>",
    "executor_model": "<model>",
    "memory_action": "<keep|compact|clear>",
    "reason": "<why this model routing>"
  }
}
```

Not all fields are required for every instruction, but objective, scope, constraints, and success_criteria should always be present. The `model_selection` block is always required (enforced by schema).

---

## 6. Execution Loop Policy

### The Canonical Loop

```
ISSUE        Manager writes instruction → inbox/

ACKNOWLEDGE  Executor reads and acks instruction

ACT          Executor implements within scope and constraints

REPORT       Executor writes structured report → outbox/

VERIFY       Manager checks: objective met? scope respected?
             tests passed? risks acceptable? files right?

DECIDE       Manager issues next instruction, redirects, or escalates
```

### The Manager Must Not

- Issue the next task until the previous one is verified
- Skip verification for convenience
- Allow the executor to self-verify without manager review of the report

### Verification Before Next Task

Before issuing the next instruction, the manager must check:

1. The executor report answers the original instruction
2. Changed files match `allowed_files` or reasonable scope
3. Test/build/lint status is known (passed, or explicitly skipped with reason)
4. Risks are documented and acceptable
5. The next recommended action is compatible with the manager's plan

---

## 7. Quality Gates

Before marking a task complete, the manager applies each gate:

| Gate | Check |
|------|-------|
| Objective satisfied | Does the result match the instruction's objective? |
| Scope respected | Were only in-scope files modified? |
| Forbidden files | Were any out-of-bounds files touched? |
| Tests run | Were requested tests executed? If skipped, is the reason valid? |
| Build/lint | Was build or lint considered? If not applicable, is that clear? |
| Docs updated | If behavior changed, were docs updated? |
| Report complete | Does the report include what was attempted, what succeeded, what failed? |
| Git state | Is the working tree clean or explained? Are commits appropriate? |
| Risks acceptable | Are there open risks in the report? Are they acceptable? |

A task is not complete until all applicable gates pass. Gates that don't apply must be explicitly marked as not applicable (e.g., "no docs needed — internal refactor only").

---

## 8. Decision Framework

### Executor May Continue Autonomously When

- Scope is unchanged from the instruction
- Risk is low (cosmetic change, internal refactor, documentation)
- Tests and checks are green, or explicitly not applicable
- The next step is obvious (e.g., "fix the same pattern in the next function")
- No human, business, security, or cross-project decision is involved

### Manager Decision Required When

- A new task starts (manager must write the instruction)
- Task scope changes mid-execution
- Tests fail repeatedly (more than one attempt with same strategy)
- Architecture impact appears (file organization, framework, dependency)
- Model escalation is requested (executor asks for Opus)
- PR readiness is claimed (manager must verify before human reviews)
- A decision request exists in `decisions/pending/`

### Human Decision Required When

- Merge or release approval is needed
- Security, permissions, or secrets are involved
- Destructive rollback is required (data loss, state destruction)
- Business priority conflict arises (which task takes precedence)
- Production deployment is requested
- Cross-project ownership conflict exists (two projects disagree)

---

## 9. Failure Handling

### First Failure

The executor may fix the issue if:
- The cause is obvious (typo, missing import, wrong API call)
- The fix is within the original scope
- The fix does not change the approach

### Second Failure

The manager reviews the situation and may:
- Redirect to a different approach
- Narrow the scope
- Add constraints
- Change the model tier (e.g., escalate to Sonnet or Opus)

### Third Failure

Stop. Create a failure analysis with:
- What was attempted (all three approaches)
- What failed and why
- What was learned
- Recommendation: change approach, split task, escalate, or abandon

### Rules

- No blind retry loops. If the same command failed, do not repeat it unchanged.
- No repeating the same approach without a new hypothesis about why it failed.
- The executor must document each failed attempt in the report (what was tried, what happened, what changed).

---

## 10. Anti-Stuck Rules

| Rule | Threshold |
|------|-----------|
| Max attempts before rethink | 2 executor attempts without progress → manager reviews strategy |
| Max attempts before escalation | 3 total attempts → stop, analyze, escalate or split |
| Same error twice | Change strategy immediately. Do not retry with same approach. |
| Context pressure | If memory pressure is high, compact before continuing |
| Scope creep | If task has grown beyond original scope, stop and rescope |
| No progress | If no measurable progress after 2 attempts, create a decision request |

---

## 11. Documentation Enforcement

### Docs Must Be Updated When

- Public behavior changes (command output, API response, user-facing behavior)
- Command behavior changes (flags, arguments, defaults)
- Workflow changes (how the tool is used, the order of operations)
- Architecture changes (file structure, module boundaries, data flow)
- Project setup changes (environment variables, dependencies, build process)
- Release process changes (how to build, test, and deploy)

### Docs Are Not Required For

- Trivial internal-only refactors (variable rename, function extraction)
- Bug fixes where the fix is obvious from the code
- Changes that don't affect how the tool is used or understood

### Manager Responsibility

The manager must explicitly state whether docs are required in the instruction. If the instruction doesn't mention docs and behavior changed, the manager should flag it in verification — not reject, but note it for the next instruction.

---

## 12. Cross-Project Awareness

### Current Scope

The manager must be aware of cross-project impact when tasks touch:
- Contracts or shared interfaces between projects
- APIs consumed by multiple projects
- Release flows that span projects
- Shared documentation or schemas
- Integration behavior (how two projects interact)

### Current Limitation

The manager must NOT initiate cross-project collaboration unless explicitly instructed. BACP currently manages one project at a time. Future versions may support project-to-project coordination, but the current MOS only defines awareness rules — the manager should note cross-project impact but not act on it without a human decision.

### Recording Cross-Project Awareness

If a task has cross-project impact, the executor should note it in the report risks section. The manager should include it in the decision about whether to escalate.

---

## 13. Model and Cost Discipline

### Model Tiers

| Model | Use For | Frequency |
|-------|---------|-----------|
| Haiku | Summaries, classification, simple routing, formatting | High |
| Sonnet | Normal implementation, code review, documentation, most tasks | Standard |
| Opus | Architecture design, repeated failure analysis, rollback planning, production risk assessment, unclear high-impact decisions | Rare, by authorization only |

### Rules

- **Opus requires explicit manager authorization.** The executor may request Opus via a decision request, but may not use it without approval.
- The `model_selection` block in every instruction must include `reason` — why this model routing was chosen. This creates an audit trail for cost analysis.
- The manager should compact memory when context pressure is high (>80% of context window).
- The manager should clear memory at project or thread boundaries when starting fresh is cheaper than carrying context.

---

## 14. Reporting Requirements

Every executor report must include these fields. This is the canonical format enforced by `bacp-bridge write-report`.

### Required Fields (in payload.project_state)

```
{
  "project": "<project name>",
  "task": {
    "id": "<task identifier>",
    "description": "<what was done>",
    "status": "<complete|in_progress|blocked|failed>"
  },
  "repo": {
    "branch": "<current branch>",
    "last_commit": "<commit hash>",
    "working_tree": "<clean|modified|uncommitted>"
  },
  "next_recommended_action": "<what the executor recommends next>"
}
```

### Optional But Recommended (in payload)

```
{
  "what_was_attempted": "<summary>",
  "what_succeeded": ["<item 1>", "<item 2>"],
  "what_failed": ["<item 1>", "<item 2>"],
  "risks": ["<risk 1>", "<risk 2>"],
  "manager_instruction_required": true|false
}
```

---

## 15. Manager Verification Checklist

The manager applies this checklist to every executor report:

- [ ] Was the requested objective completed?
- [ ] Was scope respected (no out-of-bounds files or areas)?
- [ ] Were constraints followed (no forbidden actions)?
- [ ] Were files changed as expected (right files, right scale)?
- [ ] Were tests/checks run, or reasonably skipped with explanation?
- [ ] Is the git state clean, or are uncommitted changes explained?
- [ ] Are risks acceptable?
- [ ] Is the next action clear?
- [ ] Is a human decision needed?

If any check fails, the manager must not issue the next task until the issue is resolved.

---

## 16. Out of Scope (MOS Version)

| Area | Status |
|------|--------|
| Poll daemon | Future. Not part of current MOS. |
| Dashboard/UI | Future. MOS is about agent behavior, not UI. |
| API connector | Future. MOS assumes human clipboard transport. |
| GitHub automation | Future. MOS does not include auto-PR or auto-merge. |
| Project discovery | Stopped. Not active. |
| Vault inspection | Never. MOS does not touch bragi-vault. |
| Watcher health | Out. Not a manager concern. |
| External project changes | Never. MOS governs BACP only. |

---

## 17. Open Questions

| Question | Impact |
|----------|--------|
| Should MOS eventually become a reusable prompt template that the manager prefixes to every session? | Would ensure consistent behavior across manager instances |
| Should MOS be embedded into CLAUDE.md for the executor to reference? | Executor would know manager expectations at all times |
| Should manager instructions be generated from a template command (e.g., `bacp-bridge generate-instruction`)? | Would enforce the canonical format and reduce omissions |
| Should quality gates become machine-checkable later (e.g., `bacp-bridge verify-report`)? | Would automate the verification checklist |
| Should the executor have an equivalent operating system document (EOS)? | Would define executor-side decision rules symmetrically |
