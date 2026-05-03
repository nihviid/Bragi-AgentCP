# Manager Control Loop

**Design Document**  
**Bragi Agent Control Plane (BACP)**  
**Date:** 2026-05-03  
**Branch:** `agent/bootstrap-discovery`  
**Status:** Design (pre-implementation)

---

## 1. Problem Statement

Currently, the ChatGPT manager has no direct control over CLI executor activity. The human is the intermediary:

```
Manager (ChatGPT) ──(human copy/paste)──► CLI Executor
CLI Executor ──(human copy/paste)──► Manager (ChatGPT)
```

This means:

- **No oversight.** The manager cannot see what the CLI is doing between status reports.
- **No intervention.** The manager cannot pause, redirect, or stop mid-execution.
- **No verification.** The manager cannot independently verify what the CLI reports.
- **No escalation.** Blocked tasks sit until the next human copy/paste cycle.
- **No memory control.** The manager cannot tell the CLI to compact, clear, or persist memory.
- **No model control.** The manager cannot route tasks to the appropriate model tier.

The bridge solved the format problem (structured JSON replaces paste). The manager control loop solves the authority problem: the manager must be able to control the executor, not just talk to it.

---

## 2. Why Manager Control Comes Before Discovery

Discovery Passes 1 and 2 mapped the landscape. The remaining passes (3–10) would map more detail, but each additional pass explores territory without establishing the command infrastructure needed to act on it.

**The risk of continuing discovery first:** the system would know everything and control nothing.

**The shift:** establish the control contract that makes the manager authoritative over the CLI. Then discovery resumes with the manager able to direct, inspect, and verify each pass.

This is analogous to establishing a chain of command before beginning reconnaissance. The commander must be able to task the scout, receive the report, and redirect based on findings — without walking alongside them.

---

## 3. Manager Authority Model

### Authority Principles

1. **The manager allocates tasks.** The CLI does not choose what to work on. The manager decides.
2. **The manager controls model routing.** The CLI does not decide which model to use. The manager specifies.
3. **The manager controls memory.** The CLI does not decide when to compact or clear context. The manager instructs.
4. **The manager can pause, redirect, or stop at any time.** The CLI accepts and acts on these commands immediately.
5. **The manager verifies results.** CLI status reports are input to verification, not the final word.
6. **The human overrides the manager.** The human can countermand any manager instruction via `inbox/human-to-cli/`.

### Authority Scope

| Domain | Manager Authority | CLI Autonomy |
|--------|------------------|--------------|
| Task selection | Decides what to work on | None |
| Task execution | Sets constraints, approves approach | Controls implementation within constraints |
| Model selection | Specifies model per task | None — uses model manager specifies |
| Memory management | Issues compact/clear commands | Reports memory pressure |
| File creation | Approves within project scope | Creates files per instruction |
| Git operations | Specifies branch, commit strategy | Executes per instruction |
| PR operations | Approves PR readiness, reviews | Opens PRs, responds to reviews |
| Test execution | Requests specific tests | Runs and reports results |
| Kill switch | Can trigger via manager instruction | Human-only for STOP file |

### Manager Cannot

- Write source code directly (CLI executes)
- Access project repos directly (CLI bridges)
- Modify the local filesystem outside the bridge directory (relay only)
- Override human STOP file (human-only authority)

---

## 4. Executor Responsibility Model

### Responsibilities

1. **Report accurately.** Every status report reflects actual state, not desired state.
2. **Follow instructions exactly.** Deviations must be documented in the status report.
3. **Surface blocking issues immediately.** Blocked is a separate message type from status.
4. **Produce machine-readable reports.** JSON follows the defined schema.
5. **Surface all relevant state.** Git status, test results, build output, file changes.
6. **Accept commands at any time.** Between tasks, on task boundaries, and when idle.
7. **Never exceed authorized scope.** If an instruction is ambiguous, ask — do not infer.
8. **Manage context responsibly.** Report memory pressure before it causes degradation.

### Executor Cannot

- Choose its own tasks (manager assigns)
- Select its own model (manager specifies)
- Create PRs without manager approval
- Merge PRs without manager approval
- Modify files outside the project scope of the current instruction
- Ignore or defer manager commands (pause, stop, redirect are immediate)

### Idle Behavior

When the executor has no pending instruction in `inbox/manager-to-cli/`:

1. Report idle status: `{ "current_task": "idle", "manager_instruction_needed": false }`
2. Poll `inbox/` every N seconds (configurable, default 30)
3. Poll `STOP` every cycle
4. If idle exceeds configurable threshold (default 5 minutes), log and yield

---

## 5. Required State Exposed by the CLI

The CLI must expose the following state in every status report. This is the minimum viable state — the manager needs every field to exercise control.

| Field | Type | Why Manager Needs It |
|-------|------|---------------------|
| `project` | string | Which project is this about |
| `current_task` | string | What is being worked on right now |
| `task_status` | enum | `in_progress`, `completed`, `blocked`, `failed`, `idle` |
| `action_taken` | string[] | What was done since last report |
| `files_created` | string[] | New files |
| `files_modified` | string[] | Changed files |
| `files_deleted` | string[] | Deleted files |
| `commands_run` | string[] | Shell commands executed |
| `git_branch` | string | Current branch |
| `git_status` | string | Clean/dirty summary |
| `git_commits` | string[] | Recent commit messages |
| `uncommitted_changes` | int | Count of uncommitted files |
| `test_results` | object | Pass/fail/skip counts |
| `test_output` | string | Relevant test output snippets |
| `build_results` | object | Pass/fail per target |
| `lint_results` | object | Errors/warnings counts |
| `memory_status` | string | Context utilization |
| `open_questions` | string[] | Things the executor is unsure about |
| `risks` | string[] | Things that could go wrong |
| `next_recommended_action` | string | What the executor thinks should happen next |
| `manager_instruction_needed` | boolean | Is the executor waiting on the manager |
| `model_in_use` | string | Current model being used |
| `session_duration` | string | How long the current session has been running |
| `blocked_reason` | string | If blocked, why (null otherwise) |

---

## 6. Required Manager Commands

The manager can issue the following commands. Each is a JSON message type written to `inbox/manager-to-cli/`.

### Command Table

| Command | `type` | Effect | Priority |
|---------|--------|--------|----------|
| **continue** | `instruction` | Proceed with current or next task | Normal |
| **pause** | `override` | Stop work immediately, preserve state, wait for resume | High |
| **stop** | `override` | Abort current task, do not resume. Return to idle. | High |
| **redirect** | `override` | Change project or task. Current work is abandoned. | High |
| **approve** | `decision` | Confirm a proposed action may proceed | Normal |
| **reject** | `decision` | A proposed action is denied. Include reason and alternative. | Normal |
| **request_status** | `instruction` | Produce a full status report immediately | Normal |
| **request_diff** | `instruction` | Produce a git diff summary for specified files | Normal |
| **request_tests** | `instruction` | Run specified tests and report results | Normal |
| **request_pr_status** | `instruction` | Check status of open PRs for the project | Normal |
| **compact_memory** | `instruction` | Summarize and compress context, discard low-value detail | Normal |
| **clear_memory** | `override` | Reset executor context entirely. Fresh session. | High |
| **escalate_to_human** | `instruction` | Manager cannot resolve — request human decision | Normal |

### Command Schema

```json
{
  "id": "20260503T120030Z~portal~manager~p00000~b4d1.instruction.json",
  "reply_to": "20260503T120000Z~portal~executor~p12345~a8f3",
  "project": "portal",
  "direction": "manager_to_cli",
  "type": "override",
  "command": "pause",
  "from": "manager",
  "to": "executor",
  "created_at": "2026-05-03T12:00:30+02:00",
  "payload": {
    "reason": "Found conflicting PR — need human decision before continuing",
    "instruction": "Save all in-progress work. Do not commit. Report current state and wait.",
    "model_selection": {
      "manager_model": "sonnet",
      "executor_model": "haiku",
      "memory_action": "preserve",
      "reason": "low-complexity pause instruction"
    },
    "constraints": [
      "Do not modify any files",
      "Do not run tests",
      "Report within 30 seconds"
    ]
  }
}
```

### Command Behavior Matrix

| Command | CLI in Task | CLI Idle | CLI Blocked |
|---------|------------|----------|-------------|
| **continue** | Acknowledge, continue | Check inbox for next task | Acknowledge, explain block, await resolution |
| **pause** | Stop work, save state, report | Already idle, report state | Stop blocking escalation, wait |
| **stop** | Abort, discard in-progress, report | Already idle, confirm | Abort escalation, return to idle |
| **redirect** | Abort current, switch project/task | Switch project context | Abort escalation, switch context |
| **approve** | Incorporate as go-ahead | N/A | Proceed with approved approach |
| **reject** | Adjust approach, do not proceed | N/A | Block stands, propose alternative |
| **request_status** | Produce full report | Produce full report | Produce full report |
| **request_diff** | Produce diff | Produce diff (no changes) | Produce diff |
| **request_tests** | Queue tests, report when done | Queue tests, report when done | Cannot run tests while blocked |
| **request_pr_status** | Check PRs, report | Check PRs, report | Check PRs, report |
| **compact_memory** | Compact, resume | Compact, stay idle | Compact, stay blocked |
| **clear_memory** | Save state, clear, fresh start | Clear, fresh start | Clear, fresh start |
| **escalate_to_human** | Save state, wait | Wait | Wait |

---

## 7. Manager-Readable Project State Schema

The manager needs a consolidated view of each project's current state. This is the schema for the project state file that the CLI writes after every instruction cycle.

```json
{
  "project": "portal",
  "report_id": "20260503T120000Z~portal~executor~p12345~a8f3",
  "generated_at": "2026-05-03T12:00:00Z",
  "task_state": {
    "current": "Implement device descriptor endpoint",
    "status": "in_progress",
    "progress_pct": 65,
    "started_at": "2026-05-03T10:00:00Z",
    "estimated_completion": "2026-05-03T14:00:00Z"
  },
  "repo_state": {
    "branch": "feature/device-descriptor",
    "status": "dirty",
    "uncommitted_files": 3,
    "ahead_of_remote": 2,
    "behind_remote": 0,
    "recent_commits": [
      "621769d Add BRAGI_DRIVER_SIGNING_KEY to e2e workflow",
      "5e893b1 Fix BASE_URL env-var bug in registry-events-contract"
    ]
  },
  "test_state": {
    "last_run": "2026-05-03T11:30:00Z",
    "total": 142,
    "passed": 140,
    "failed": 2,
    "skipped": 0,
    "failing_tests": [
      "registry-events-contract.test.ts:87 — expected pem but got spki",
      "device-descriptor.test.ts:34 — timeout"
    ]
  },
  "build_state": {
    "last_result": "passed",
    "duration_seconds": 45
  },
  "lint_state": {
    "errors": 0,
    "warnings": 3,
    "new_violations": 0
  },
  "pr_state": {
    "open_prs": [
      {
        "number": 23,
        "title": "Realign driver-registry contract with AF-09 v0.2",
        "status": "open",
        "checks_passing": true,
        "review_status": "changes_requested",
        "draft": false
      }
    ],
    "recently_merged": []
  },
  "memory_state": {
    "context_utilization_pct": 72,
    "session_age_minutes": 120,
    "recommended_action": "none"
  },
  "decision_state": {
    "pending_decisions": [
      {
        "id": "20260503T130000Z~portal~executor~p12345~c7e2",
        "title": "Should we use REST or gRPC for device descriptor endpoint",
        "created_at": "2026-05-03T13:00:00Z"
      }
    ],
    "open_questions": [
      "Is the AF-09 v0.2 contract final or still iterating?"
    ]
  },
  "executor_recommendation": {
    "next_action": "Fix failing registry-events-contract test",
    "needs_approval": true,
    "recommended_model": "sonnet"
  },
  "manager_decision_needed": true
}
```

---

## 8. Executor Report Schema

Every executor report follows this structure. It is the superset — individual reports may omit optional fields.

```json
{
  "id": "timestamp~project~role~pid~nonce",
  "project": "appkit | portal | msdk | router | bacp",
  "direction": "cli_to_manager",
  "type": "status_report | blocked | error | information | decision_request",
  "from": "executor",
  "to": "manager",
  "requires_response": true | false,
  "created_at": "ISO8601 timestamp",
  "payload": {
    "current_task": "string",
    "task_status": "in_progress | completed | blocked | failed | idle",
    "action_taken": ["string"],
    "files_created": ["string"],
    "files_modified": ["string"],
    "files_deleted": ["string"],
    "commands_run": ["string"],
    "git_branch": "string",
    "git_status": "clean | dirty",
    "git_commits": ["string"],
    "uncommitted_changes": 0,
    "diff_summary": "string (short summary of what changed)",
    "test_results": {
      "ran": true | false,
      "total": 0,
      "passed": 0,
      "failed": 0,
      "skipped": 0,
      "failing_tests": ["string"],
      "output_snippet": "string (relevant lines only)"
    },
    "build_results": {
      "ran": true | false,
      "passed": true | false,
      "targets": {}
    },
    "lint_results": {
      "ran": true | false,
      "errors": 0,
      "warnings": 0,
      "new_violations": 0
    },
    "pr_status": {
      "checked": true | false,
      "open_count": 0,
      "prs": []
    },
    "memory_status": {
      "context_utilization_pct": 0,
      "session_age_minutes": 0,
      "recommended_action": "compact | clear | none"
    },
    "open_questions": ["string"],
    "risks": ["string"],
    "blocked_reason": "string | null",
    "next_recommended_action": "string",
    "recommended_model": "haiku | sonnet | opus",
    "manager_instruction_needed": true | false
  }
}
```

### Report Type Requirements

| `type` | Required Fields (in addition to id/project/timestamps) | `requires_response` |
|--------|-------------------------------------------------------|-------------------|
| `status_report` | All payload fields except `blocked_reason` | `true` if task complete or blocked |
| `blocked` | `blocked_reason`, `current_task`, `action_taken`, `risks`, `open_questions` | `true` |
| `error` | `current_task`, `action_taken`, `commands_run`, `risks` (with error details) | `true` |
| `information` | `current_task`, `action_taken` (minimum) | `false` |
| `decision_request` | `open_questions`, `risks`, `next_recommended_action` | `true` |

---

## 9. Manager Instruction Schema

Every instruction from the manager to the CLI follows this structure.

```json
{
  "id": "timestamp~project~role~pid~nonce",
  "reply_to": "previous message id or null",
  "project": "appkit | portal | msdk | router | bacp",
  "direction": "manager_to_cli",
  "type": "instruction | override | decision | acknowledgment | clarification | kill",
  "command": "continue | pause | stop | redirect | approve | reject | request_status | request_diff | request_tests | request_pr_status | compact_memory | clear_memory | escalate_to_human",
  "from": "manager",
  "to": "executor | human",
  "created_at": "ISO8601 timestamp",
  "payload": {
    "decision": "continue | stop | pause | redirect | approve | reject | escalate",
    "instruction": "string — what the executor should do",
    "instruction_detail": "string — additional context, rationale, or approach guidance",
    "success_criteria": ["string"],
    "constraints": ["string"],
    "model_selection": {
      "manager_model": "sonnet",
      "executor_model": "haiku | sonnet | opus",
      "memory_action": "keep | compact | clear",
      "reason": "string — why this model and memory action"
    },
    "task_timeout_minutes": 0
  }
}
```

### Instruction Type Constraints

| `type` | Allowed `command` Values | Requires `reply_to`? |
|--------|-------------------------|---------------------|
| `instruction` | `continue`, `request_status`, `request_diff`, `request_tests`, `request_pr_status`, `compact_memory` | No (but recommended) |
| `override` | `pause`, `stop`, `redirect`, `clear_memory` | Yes |
| `decision` | `approve`, `reject` | Yes |
| `acknowledgment` | none (informational only) | Yes |
| `clarification` | none (informational only) | Yes |
| `kill` | `escalate_to_human` | No |

---

## 10. Verification Loop

The core control loop. Every manager-initiated task follows this cycle.

```
┌─────────────────────────────────────────────────────────┐
│                     VERIFICATION LOOP                    │
│                                                          │
│  MANAGER           CLI              MANAGER              │
│  issues task ───► acknowledges ──►                       │
│                    │                                      │
│                    ▼                                      │
│                  acts                                     │
│                    │                                      │
│                    ▼                                      │
│                  reports ───────────► reads               │
│                                        │                  │
│                                        ▼                  │
│                                      verifies             │
│                                        │                  │
│                         ┌──────────────┼──────────────┐   │
│                         ▼              ▼              ▼   │
│                      PASSED        FAILED         BLOCKED │
│                         │              │              │   │
│                   decide next    diagnose and     escalate │
│                   task or end    re-instruct     or pause │
│                         │              │              │   │
│                         └──────────────┴──────────────┘   │
│                                        │                  │
│                                        ▼                  │
│                                  next cycle               │
└─────────────────────────────────────────────────────────┘
```

### Loop States

| State | Manager Action | CLI Action | Expected Output |
|-------|---------------|------------|-----------------|
| **ISSUE** | Write instruction to `inbox/manager-to-cli/` | Poll inbox | CLI reads instruction |
| **ACKNOWLEDGE** | Wait | Write acknowledgment to `outbox/cli-to-manager/` | Manager sees acknowledgment |
| **ACT** | Wait (no new instructions) | Execute task per instruction | Files changed, commands run |
| **REPORT** | Wait | Write status report to `outbox/cli-to-manager/` | Full status payload |
| **VERIFY** | Read report, check success criteria, inspect results | Wait | Decision: next, fix, escalate |
| **DECIDE** | Write next instruction | Wait for next instruction | New cycle or end |

### Verification Rules

- **Passed:** All success criteria met. Manager may approve, or request additional verification.
- **Failed:** Some criteria not met. Manager must diagnose and either re-instruct or mark as known deviation.
- **Blocked:** CLI cannot proceed. Manager must resolve the block, escalate to human, or abort.
- **Timeout:** If CLI does not report within `task_timeout_minutes`, manager should issue `request_status`.

### Timeout Behavior

| Duration | Action |
|----------|--------|
| Task timeout (configurable) | Manager issues `request_status` |
| 2× task timeout | Manager issues `pause` + `request_status` |
| 3× task timeout | Manager issues `stop`, escalates to human |

---

## 11. GitHub Visibility

The CLI must expose GitHub state in every status report. The manager needs this to verify work without direct GitHub access.

### Manager-Readable GitHub State

| Field | Source | Manager Use |
|-------|--------|-------------|
| `branch` | `git branch --show-current` | Know what branch is active |
| `git_status` | `git status --porcelain` | See uncommitted work |
| `recent_commits` | `git log --oneline -5` | See what was committed |
| `ahead/behind` | `git rev-list --count HEAD..@{u}` | Know sync state with remote |
| `open_prs` | `gh pr list --json number,title,state,headRefName,reviewDecision,statusCheckRollup` | See open work |
| `pr_checks` | `gh pr view <number> --json statusCheckRollup` | Check CI results |
| `pr_review` | `gh pr view <number> --json reviews` | Check review state |
| `changed_files` | `gh pr diff <number> --name-only` | See PR diff scope |
| `diff_summary` | `git diff --stat` | See local changes summary |

### CLI Actions for GitHub Visibility

| Action | Command | When |
|--------|---------|------|
| Report branch | `git branch --show-current` | Every status report |
| Report commits | `git log --oneline -5` | After any commit |
| Report PR state | `gh pr list ...` | Every status report |
| Report PR checks | `gh pr view <num> --json statusCheckRollup` | When PR is referenced |
| Diff summary | `git diff --stat` | When files are modified |
| Detailed diff | `git diff <files>` | On `request_diff` command |

---

## 12. Local Repo Visibility

The CLI exposes local repo state beyond what GitHub shows. The manager needs this to verify mid-task progress.

### Local State Exposed

| State | Source | Manager Use |
|-------|--------|-------------|
| Current branch | `git branch --show-current` | Confirm correct branch |
| Working tree status | `git status --porcelain` | See all changes |
| Unstaged changes | `git diff --stat` | See what's not yet staged |
| Staged changes | `git diff --cached --stat` | See what's ready to commit |
| Untracked files | `git status --porcelain | grep '^??'` | See new files |
| Test results | `pytest/vitest output` | Verify tests pass |
| Build output | Build tool output | Verify build succeeds |
| Lint output | Lint tool output | Verify no new violations |
| Memory pressure | Internal tracking | Know when to compact |

### Test Result Reporting

Tests are reported at three levels:

1. **Summary** — pass/fail/skip counts. Every status report.
2. **Failing tests** — test names and error messages. Every status report when tests ran.
3. **Full output** — `request_tests` response includes complete output for failing tests.

### Build Result Reporting

Builds are reported at two levels:

1. **Pass/fail** — per target. Every status report when build ran.
2. **Error output** — build errors. When build failed.

---

## 13. Decision Routing

Not every situation requires the same decider. This section defines what routes where.

### Routing Table

| Situation | Default Router | Can Delegate To | Can Escalate To |
|-----------|---------------|-----------------|-----------------|
| Task complete, criteria met | **CLI autonomously** reports | Manager verifies | Human if manager disagrees |
| Task complete, criteria partially met | **Manager** decides next step | CLI can recommend | Human if unresolvable |
| Blocked on technical choice | **Manager** decides | CLI can recommend | Human if architectural |
| Blocked on business decision | **Manager** decides | N/A | **Human** always |
| Blocked on access/permissions | **Human** decides | N/A | N/A |
| New file needed outside scope | **Manager** must approve | CLI can propose | Human if manager unsure |
| Cross-repo impact detected | **Manager** decides | CLI can flag | Human if high impact |
| PR ready for review | **Manager** reviews | CLI creates PR | Human if manager rejects |
| PR checks failing | **CLI autonomously** investigates | Reports findings | Manager decides fix or abandon |
| Test failure | **CLI autonomously** fixes | Reports fix | Manager reviews |
| Security concern | **Human** decides | Manager can flag | N/A |
| Memory pressure (>80%) | **CLI autonomously** compacts | Reports compaction | Manager can skip |
| Model selection override | **Manager** specifies | CLI can request | Human if manager refuses |

### Autonomous vs. Manager-Required

**CLI can do without manager approval:**
- Run tests
- Fix test failures
- Create files within the instruction scope
- Stage changes
- Compact memory (when pressure > 80%)
- Report status
- Investigate PR check failures

**CLI must ask manager before:**
- Starting a new task
- Changing the task scope
- Creating a PR
- Switching branches
- Deleting files
- Modifying files outside instruction scope
- Changing approach when current approach fails
- Using Opus (requires manager authorization)

**CLI must escalate to human (cannot resolve):**
- Missing credentials or access
- Business decisions
- Security concerns
- Cross-repo contract changes
- Any situation where manager says "escalate"

---

## 14. Kill Switch Behavior

### Two-Layer Kill

| Layer | Mechanism | Who Activates | Effect |
|-------|-----------|--------------|--------|
| **Bridge STOP** | `STOP` file at `~/.bragi/agent-control-plane/STOP` | **Human only** | Complete bridge halt. No messages read or written. All agents exit loops. |
| **Manager kill** | `type: kill` instruction in `inbox/manager-to-cli/` | **Manager** | Current task aborted. Executor returns to idle. Bridge structure remains operational. Manager can re-task. |

### Bridge STOP Behavior

When `STOP` file is present:
- CLI checks `STOP` at every poll cycle
- If STOP found: log "BRIDGE HALTED", refuse all bridge operations, exit active loops
- If STOP found mid-task: save work-in-progress state, do not commit, do not push
- Manager informs ChatGPT: "Bridge is halted" — no further bridge operations
- Recovery: human deletes STOP, appends resume entry to `bridge.log`

### Manager Kill Behavior

When `type: kill` instruction arrives:
- CLI logs "MANAGER KILL: {reason}"
- Aborts current task
- Saves work-in-progress state (stash, do not commit)
- Returns to idle state
- Remains available for new instructions
- Reports status: `{ "task_status": "killed", "killed_by": "manager", "reason": "..." }`

### Kill vs. Stop vs. Pause

| Command | Completes Current Work? | Can Resume? | State Preserved? |
|---------|------------------------|-------------|------------------|
| **pause** | Yes (saves state) | Yes | Full state preserved |
| **stop** (manager) | No (aborts) | No — fresh start | Work saved to stash |
| **kill** (manager) | No (aborts) | No — fresh start | Work saved to stash |
| **STOP** (human) | No (hard halt) | Yes (after STOP removed) | Work saved to stash, bridge state preserved |

---

## 15. Audit Log Behavior

### Log Location
`~/.bragi/agent-control-plane/logs/bridge.log`

### Log Format
```
TIMESTAMP | DIRECTION | STAGE | MESSAGE_ID | TYPE | STATUS
```

### Logged Events

Every control-loop event is logged:

| Event | Log Line |
|-------|----------|
| Manager issues instruction | `2026-05-03T12:00:30Z | IN | manager-to-cli | <id> | instruction | PENDING` |
| CLI acknowledges | `2026-05-03T12:00:31Z | OUT | cli-to-manager | <id> | acknowledgment | PENDING` |
| CLI reports status | `2026-05-03T12:30:00Z | OUT | cli-to-manager | <id> | status_report | PENDING` |
| Manager verifies | `2026-05-03T12:30:05Z | IN | manager-to-cli | <id> | decision | PENDING` |
| Task verified passed | `2026-05-03T12:30:06Z | VERIFY | passed | <id> | task | COMPLETE` |
| Task verified failed | `2026-05-03T12:30:06Z | VERIFY | failed | <id> | task | REJECTED` |
| Task blocked | `2026-05-03T12:30:06Z | BLOCK | <id> | task | BLOCKED` |
| Manager pause | `2026-05-03T12:35:00Z | CTRL | pause | <id> | override | ACTIVE` |
| Manager stop | `2026-05-03T12:35:00Z | CTRL | stop | <id> | override | ACTIVE` |
| Manager kill | `2026-05-03T12:35:00Z | CTRL | kill | <id> | kill | ENGAGED` |
| Human STOP | `2026-05-03T12:40:00Z | KILL | STOP | - | kill_switch | ENGAGED` |
| Bridge error | `2026-05-03T12:45:00Z | ERR | <stage> | <id> | error | FAILED` |
| Memory compact | `2026-05-03T12:50:00Z | MEM | compact | <id> | memory | COMPLETE` |
| Memory clear | `2026-05-03T12:50:00Z | MEM | clear | <id> | memory | COMPLETE` |
| Model routing | `2026-05-03T12:55:00Z | MODEL | sonnet | <id> | routing | ASSIGNED` |

### Log Event Prefixes

| Prefix | Meaning |
|--------|---------|
| `IN` | Message to CLI inbox |
| `OUT` | Message written to CLI outbox |
| `VERIFY` | Verification result |
| `BLOCK` | Task blocked |
| `CTRL` | Manager control command (pause/stop) |
| `KILL` | Kill switch event |
| `MEM` | Memory operation |
| `MODEL` | Model routing decision |
| `ERR` | Bridge error |
| `ARC` | Archive operation |

---

## 16. Model Routing Control

The manager controls which model the executor uses. The executor never selects its own model.

### Model Tier Definitions

| Tier | Model | Use Case | Max Context | Cost Tier |
|------|-------|----------|-------------|-----------|
| **Routing** | Haiku | Summarization, routing decisions, simple greps, file listing, acknowledgment messages | Low | Cheap |
| **Execution** | Sonnet | Normal development work: code, tests, documentation, code review | Medium | Standard |
| **Deep** | Opus | Architecture decisions, complex debugging, cross-repo impact analysis, design documents | High | Expensive |

### Routing Rules

| Situation | Default Model | Manager Can Override? |
|-----------|--------------|---------------------|
| Read file, list directory | Haiku (routing) | Yes — to Sonnet if deep analysis needed |
| Write simple message | Haiku (routing) | Yes |
| Acknowledge instruction | Haiku (routing) | Yes |
| Write code | Sonnet (execution) | Yes — to Opus for complex logic |
| Run tests | Sonnet (execution) | Yes |
| Git operations | Sonnet (execution) | Yes |
| Code review | Sonnet (execution) | Yes — to Opus for security review |
| Architecture decision | Opus (deep) | Yes — to Sonnet if bounded decision |
| Design document | Opus (deep) | Yes |
| Cross-repo impact analysis | Opus (deep) | Yes |
| Security review | Opus (deep) | No — Opus required |

### Model Selection in Instructions

Every instruction includes a `model_selection` block:

```json
"model_selection": {
  "manager_model": "sonnet",
  "executor_model": "sonnet",
  "memory_action": "keep",
  "reason": "standard development task — implement device descriptor endpoint"
}
```

### Memory Actions

| Action | Effect | When |
|--------|--------|------|
| `keep` | Preserve full context | Normal operation |
| `compact` | Summarize and discard low-value detail | Before long tasks, after checkpoint |
| `clear` | Reset context, fresh session | After major milestone, on context pressure |

### Opus Authorization

Opus is restricted. The executor cannot use Opus without explicit manager authorization.

- If the executor thinks Opus is needed, it must **request** it in the status report: `"recommended_model": "opus"`
- The manager must **approve** Opus in the next instruction: `"executor_model": "opus"`
- The manager can also **deny** Opus and specify a different model: `"executor_model": "sonnet"`
- The bridge log records every model routing decision with the `MODEL` prefix

---

## 17. Dashboard Relationship

The dashboard is an interface to the manager-control loop, not a replacement for it.

### Dashboard Role

- **Read-only interface** to bridge state. Never writes to inbox, outbox, or decisions.
- **Visualizes** the verification loop: current task, pending decisions, active PRs, test results.
- **Surfaces** bridge log as an activity timeline.
- **Shows** project state from the manager-readable schema.
- **Exposes** kill switch status.

### Dashboard Does NOT

- Issue manager commands (continue, pause, stop, redirect, etc.)
- Replace the ChatGPT manager as the decision authority
- Write instructions to the CLI
- Modify bridge state
- Create or archive messages

### Authority Chain

```
ChatGPT Manager (authority)
    │
    ├── controls CLI Executor via bridge instructions
    │
    ├── controls model routing via model_selection in instructions
    │
    ├── controls memory via compact/clear commands
    │
    └── reports to Dashboard (read-only visualization)

Dashboard (visualization)
    │
    └── consumed by human for awareness, not control

Human (overriding authority)
    │
    ├── can kill bridge via STOP file
    │
    ├── can override manager via inbox/human-to-cli/
    │
    └── can instruct manager via ChatGPT interface
```

---

## 18. Minimal Implementation Plan

### Phase 0 — Schema Definition (NOW)

| Step | What | Deliverable |
|------|------|-------------|
| 0.1 | Finalize manager instruction schema | This document |
| 0.2 | Finalize executor report schema | This document |
| 0.3 | Finalize project state schema | This document |
| 0.4 | Finalize command definitions | This document |

### Phase 1 — Manual Verification Loop

| Step | What | Who |
|------|------|-----|
| 1.1 | CLI writes full-schema status reports to bridge outbox (already doing this) | Executor |
| 1.2 | Manager reads report, issues structured instruction | Manager |
| 1.3 | CLI acknowledges instruction and acts | Executor |
| 1.4 | Manager verifies report against success criteria | Manager |
| 1.5 | Manager decides next step (continue/approve/reject/pause) | Manager |
| 1.6 | All loop iterations logged to bridge.log | Executor |

**Phase 1 is the existing Phase 1 bridge, augmented with the control-loop schema.** No new automation. Human still transports messages. But the messages themselves carry full control data.

### Phase 2 — Assisted Verification

| Step | What |
|------|------|
| 2.1 | CLI automatically checks inbox before starting new work |
| 2.2 | CLI automatically reports memory pressure and recommends action |
| 2.3 | CLI automatically runs tests after code changes and includes results |
| 2.4 | CLI automatically checks PR status and includes in report |
| 2.5 | Manager can issue `request_status` mid-task (via human relay) |

### Phase 3 — Automated Verification Loop

| Step | What |
|------|------|
| 3.1 | Bridge helper monitors outbox and routes to manager |
| 3.2 | Manager responses routed back to CLI inbox |
| 3.3 | Verification loop runs without manual transport |
| 3.4 | Dashboard visualizes loop state |

---

## 19. Open Questions

| Question | Impact | Needs Answer From |
|----------|--------|-------------------|
| Should the executor auto-compact at memory > 80% or always wait for manager instruction? | Autonomy vs. control balance | Manager |
| Should `request_tests` specify exact test paths or can CLI decide what to run? | Latitude vs. precision | Manager |
| Is the `model_selection` block mandatory in every instruction or can it default? | Verbosity vs. control | Manager |
| Should PR creation be fully manager-gated or can CLI create draft PRs autonomously? | Workflow efficiency | Manager + Human |
| Should the CLI stash work on `pause` or leave working tree intact? | Safety vs. convenience | Human |
| How long should the CLI wait for a manager response before timing out? | Reliability | Manager |
| Should the dashboard be a separate process or a Tauri app consuming bridge files? | Implementation choice | Human (post-discovery) |
| Does the executor need a separate `model_in_use` field beyond `recommended_model`? | Traceability | Manager |
