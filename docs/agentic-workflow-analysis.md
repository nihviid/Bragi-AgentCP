# Agentic Workflow — UX Flow, Failure Modes, and Optimizations

**Date:** 2026-05-03
**System:** BACP Trio (Manager + Executor + Terminal)

---

## 1. Complete UX Flow

### Phase 0: Startup

```
User runs: bacp-trio portal
                  │
                  ▼
          iTerm2 window opens (3 panes)
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
    Manager    Executor   Terminal
   /tmp/bacp-  ~/portal/  ~/portal/
   manager-    claude     (shell)
   portal/
   claude

Each pane has CLAUDE.md defining its role.
BACP_PROJECT and BACP_ROOT set in environment.
```

### Phase 1: Task Request (Executor → Manager)

```
Executor writes report:
  bacp-bridge write-report
         │
         ▼
  Validates required fields:
    - project, task.{id,description,status}
    - repo.{branch,last_commit,working_tree}
    - next_recommended_action
         │
         ▼
  Writes JSON to: outbox/<ts>~<project>~executor~<pid>~<nonce>.status.json
         │
         ▼
  Creates flag: NOTIFY-MANAGER-<PROJECT> (zero bytes)
         │
         ▼
  Logs to bridge.log
```

### Phase 2: Flag Detection (Manager)

```
bacp-manager --watch --project portal
         │
  Every 5s: does NOTIFY-MANAGER-PORTAL exist?
         │
         ├── No → sleep 5s, repeat
         │
         └── Yes → delete flag (atomic consumption)
                    │
                    ▼
                    Read oldest report from outbox/cli-to-manager/
                    │
                    ├── No reports → log warning, continue
                    │
                    └── Report found → process
```

### Phase 3: Manager Processing

```
Manager reads report → calls LLM with:
  - MOS rules (one task = one objective)
  - Memory log (last 10 decisions)
  - Current report JSON
         │
         ▼
  LLM generates instruction JSON:
  {
    "command": "redirect",
    "instruction": "...",
    "project": "portal",
    "model_selection": { ... }
  }
         │
         ▼
  Manager validates instruction has required fields
         │
         ▼
  Writes instruction to: inbox/<ts>~<project>~manager~<pid>~<nonce>.redirect.json
         │
         ▼
  Creates flag: NOTIFY-EXECUTOR-<PROJECT>
         │
         ▼
  Writes to memory log: manager-log.jsonl
  Logs to bridge.log
```

### Phase 4: Flag Detection (Executor)

```
bacp-executor --watch --project portal
         │
  Every 5s: does NOTIFY-EXECUTOR-PORTAL exist?
         │
         ├── No → sleep 5s, repeat
         │
         └── Yes → delete flag (atomic consumption)
                    │
                    ▼
                    Read oldest instruction from inbox/manager-to-cli/
                    │
                    ├── No instructions → log warning, continue
                    │
                    └── Instruction found → process
```

### Phase 5: Execution

```
Executor reads instruction → calls LLM with:
  - Execution prompt (plan, commands, files)
  - Current instruction JSON
         │
         ▼
  LLM generates execution plan:
  {
    "commands": ["git status", "npm run build"],
    "files_to_modify": [{"path": "...", "content": "..."}],
    "explanation": "..."
  }
         │
         ▼
  Executor:
  1. Writes modified files to disk
  2. Runs commands via subprocess
  3. Captures stdout/stderr
         │
         ▼
  For each command:
  ├── Exit 0 → success, continue
  └── Exit ≠ 0 → log error
                  │
                  ▼
                  Call LLM with error context for auto-fix
                  │
                  ├── Fix suggested → apply fix, retry
                  └── No fix → record as failure
```

### Phase 6: Reporting (Executor → Manager)

```
Executor writes report:
  bacp-bridge write-report --ack-source <id>
         │
         ▼
  Includes:
  - what_was_attempted
  - what_succeeded (list)
  - what_failed (list)
  - project_state (branch, commit, working tree)
  - next_recommended_action

  Archives source instruction (--ack-source)
  Creates flag: NOTIFY-MANAGER-<PROJECT>
         │
         ▼
  Loop returns to Phase 2
```

### Phase 7: Termination

```
User presses Ctrl+C in Manager or Executor pane
         │
         ▼
  Agent stops. Bridge state preserved.
  Pending messages remain in queues.
  Flags remain (or already consumed).
         │
         ▼
  Next start picks up where it left off.
```

---

## 2. Failure Mode Analysis

### F1: LLM Generates Invalid JSON

| Stage | Probability | Impact | Detection |
|-------|-------------|--------|-----------|
| Manager | Medium | Instruction rejected, cycle broken | `bacp-manager` logs "LLM response was not valid JSON" |
| Executor | Medium | Execution plan rejected | `bacp-executor` logs "LLM response was not valid JSON" |

**Root cause:** LLM adds markdown code fences, commentary, or extra fields.

**Current mitigation:** `bacp-manager` strips markdown fences, validates with `json.loads()`. Falls back to defaults for missing `model_selection`.

**Failure cascade:** Instruction is not written → no NOTIFY flag → Executor stays idle → Manager has no report to respond to → loop dead.

**Optimization:** Retry LLM call with "respond with ONLY valid JSON, no commentary" error message. Up to 2 retries.

### F2: API Unavailable (502, 401, Timeout)

| Stage | Probability | Impact | Detection |
|-------|-------------|--------|-----------|
| All | Medium-High | LLM call fails, no decision made | `bacp-manager`/`bacp-executor` logs "API error {code}" |

**Root cause:** LiteLLM proxy down, network issue, auth token expired.

**Current mitigation:** None. Process exits on API error.

**Failure cascade:** API fails → no LLM response → no instruction/report written → no NOTIFY flag → other agent stays idle → system dead.

**Optimization:** Retry with exponential backoff (1s, 2s, 4s, 8s). Log and continue waiting instead of exiting.

### F3: Cross-Project Contamination

| Stage | Probability | Impact | Detection |
|-------|-------------|--------|-----------|
| Manager | Low (after per-project flags) | Executor works on wrong project | Executor runs commands in wrong repo |

**Root cause:** Manager writes instruction without `project` field. Generic flag not cleaned up. Old code path still used.

**Current mitigation:** Per-project NOTIFY flags (f758171). Each agent only watches its own project's flag.

**Failure cascade:** AppKit executor modifies Portal repo → wrong CI triggers → merge conflicts.

**Optimization:** Add `project` field validation in `bacp-bridge write-instruction` (require it to be non-empty). Executor should verify the instruction's `project` field matches its own `BACP_PROJECT` before executing.

### F4: Message Desync (Flag Without Message / Message Without Flag)

| Type | Probability | Impact | Detection |
|------|-------------|--------|-----------|
| Orphan flag | Low | Extra poll cycle, harmless | Flag exists, no message found, flag already deleted |
| Orphan message | Low | Message sits in queue forever | Queue inspection shows stale messages |

**Root cause (orphan flag):** Flag created but process crashes before message is written. Or flag created but agent deletes it before the other agent reads the message (race condition).

**Current mitigation:** Both agents read the queue after flag deletion (not before). Even if the flag is wrong, the queue read catches it.

**Optimization:** Periodic queue cleanup (archive messages older than 7 days). Monitor queue depth and alert on stale messages.

### F5: Agent Falls Out of Sync (Both Waiting)

| Stage | Probability | Impact | Detection |
|-------|-------------|--------|-----------|
| All | Low-Medium | System idles, no work done | No activity in any pane for extended period |

**Root cause:** Manager finishes processing and writes instruction. Executor's flag callback misses the creation (race on filesystem). Or both agents think the other should start.

**Current mitigation:** Flag-based polling catches missed signals on next tick (5s max delay).

**Optimization:** Minimum viable fix — poll interval is already 5s. Worst case: 5s delay. Acceptable.

### F6: Command Failure Cascade

| Stage | Probability | Impact | Detection |
|-------|-------------|--------|-----------|
| Executor | High (npm install, git merge, typecheck) | Executor reports failure, loop continues | `bacp-executor` logs "Command failed ({code})" |

**Root cause:** npm install fails → build fails → typecheck fails → tests fail → cascade. Each failure poisons the next step.

**Current mitigation:** `bacp-executor` attempts one auto-fix via LLM after each failure. Reports failures in structured format.

**Optimization:** Stop on first failure. Do not cascade. Write partial report with what succeeded + what failed. Let Manager decide whether to retry, redirect, or skip.

### F7: Context Exhaustion in Claude Sessions

| Stage | Probability | Impact | Detection |
|-------|-------------|--------|-----------|
| Executor | High (long sessions) | Claude loses track of task, generates poor code, or refuses to continue | Claude warns "Context at X%" |

**Root cause:** Long-running Claude Code session accumulates conversation history. Each `/loop` tick adds context.

**Current mitigation:** None in automated mode. Manual `/compact` or `/clear` in interactive mode.

**Optimization:** Add context pressure check to `bacp-executor`. If the session has processed >10 instructions, recommend session restart. The NOTIFY flag ensures no messages are lost during restart.

### F8: Flag Race Condition (Two Agents, Same Flag)

| Stage | Probability | Impact | Detection |
|-------|-------------|--------|-----------|
| All | Low (per-project flags) | Both agents process same message | Duplicate work |

**Root cause:** Two Manager processes watching the same project's flag. Both see flag, both delete it (second delete silently fails), both process the same report.

**Current mitigation:** Per-project flags prevent this for different projects. `atomics` (file deletion is atomic on macOS). Second deletion fails silently.

**Optimization:** Add PID file locking in flag directory. Agent checks if another process is watching its flag before starting.

### F9: Partial Write (Crash Mid-Write)

| Stage | Probability | Impact | Detection |
|-------|-------------|--------|-----------|
| All | Low | Corrupt JSON in queue | Next read fails with JSONDecodeError |

**Root cause:** Power loss, OS crash, or OOM killer during `json.dump()` to inbox/outbox.

**Current mitigation:** `write-instruction` and `write-report` write the entire file atomically (Python `open()` + write + close).

**Optimization:** Write to temp file, then `os.rename()` (atomic on same filesystem). Add periodic queue validation to detect and remove corrupt files.

### F10: STOP File Ignored

| Stage | Probability | Impact | Detection |
|-------|-------------|--------|-----------|
| All | Low | Kill switch bypassed | STOP exists but agents continue processing |

**Root cause:** Agent checks STOP only at cycle start, not mid-cycle. If STOP is written during LLM call or command execution, it is not detected until next cycle.

**Current mitigation:** STOP checked at the beginning of each poll cycle. Maximum delay: 5s + processing time.

**Optimization:** Check STOP before and after each major step (before LLM call, before command execution). Mid-cycle stop would cancel the current operation.

---

## 3. Optimization Priority Matrix

| Priority | Optimization | Effort | Impact | Risk |
|----------|-------------|--------|--------|------|
| **P0** | Retry API calls with backoff (F2) | Low | Prevents system dead on transient API errors | None |
| **P0** | Stop on first command failure, report partial results (F6) | Low | Prevents cascade failures | None |
| **P1** | Require `project` field in write-instruction validation (F3) | Low | Hardens project isolation | None |
| **P1** | Retry LLM on invalid JSON with corrective prompt (F1) | Low | Recovers from most LLM format errors | None |
| **P2** | Atomic file writes (write+rename) for messages (F9) | Low | Prevents corrupt messages | None |
| **P2** | PID file locking for watch mode (F8) | Low | Prevents duplicate agents | None |
| **P3** | Mid-cycle STOP check (F10) | Medium | Faster kill switch response | None |
| **P3** | Automatic context pressure reporting (F7) | Medium | Prevents degraded execution | Low |
| **P4** | Periodic queue validation / stale message cleanup (F4) | Medium | Keeps queues clean | Low |
| **P4** | Session restart at high context pressure (F7) | High | Maintains execution quality | Medium |

---

## 4. State Machine: Preventing Desync

The core design problem is preventing both agents from acting on the same message, or neither acting. A formal state machine per message:

```
CREATED (file written, no flag)
    │
    ▼
SIGNALED (flag created) ────── Orphan risk: flag without file
    │
    ▼
DETECTED (flag deleted, file read) ──── Double-processing risk
    │                                     (mitigated by atomic flag deletion)
    ▼
PROCESSING (LLM called, action taken)
    │
    ├── Success → RESPONDED (response written, flag for other agent)
    │
    └── Failure → FAILED (partial report written, flag for other agent)
                      │
                      ▼
                  (Manager decides: retry, redirect, or abandon)
```

**Key invariant:** Only one agent can delete the flag. The other agent's deletion is a no-op (file already gone). This is the vault watcher pattern and is the correct solution.

**Second invariant:** After deleting the flag, the agent MUST read the queue. The flag says "check the queue" — it does not carry the message itself. This prevents flag/message desync.

---

## 5. Key Metrics to Track

| Metric | What It Measures | Target |
|--------|-----------------|--------|
| Report-to-instruction latency | Time from report → LLM → instruction | <30s |
| Instruction-to-report latency | Time from instruction → execution → report | <5min for simple tasks |
| API error rate | LLM calls that fail | <1% |
| Invalid JSON rate | LLM responses that fail validation | <5% |
| Command failure rate | Shell commands that exit non-zero | <10% |
| Auto-fix success rate | Failed commands that LLM fixes | >50% |
| Orphan flag count | Flags without matching messages | 0 |
| Stale message count | Messages in queue >24h | 0 |

---

## 6. Summary

### What works well

- **Flag-based signaling**: Atomic, zero-cost idle, per-project isolation
- **Dual mode**: Interactive Claude Code OR automated scripts
- **Validation at every step**: JSON validated before entering queue
- **Audit trail**: Every action logged to bridge.log

### What breaks most often

1. **LLM generates bad JSON** (highest frequency failure)
2. **API transient errors** (highest impact — dead system)
3. **Command cascades** (one failure leads to more)

### Quickest wins

- Retry API calls (P0, <20 lines of code)
- Stop on first command failure (P0, <10 lines)
- Validate project field on write-instruction (P1, <10 lines)
- Retry LLM on bad JSON with corrective prompt (P1, <10 lines)
