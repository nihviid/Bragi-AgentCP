# Manager-Control Tool — Implementation Plan

**Bragi Agent Control Plane (BACP)**  
**Date:** 2026-05-03  
**Branch:** `agent/bootstrap-discovery`

---

## 1. Current Tool Status

A single Python 3 script `scripts/bacp-bridge` (860 lines, stdlib only) manages the file bridge at `~/.bragi/agent-control-plane/`. Thirteen commands implemented. Manual JSON validation. Basic audit logging.

```
bacp-bridge [command]
```

No dependencies. No install step. Works on any system with Python 3.

---

## 2. Existing Implemented Commands

| Command | Phase | Status | Notes |
|---------|-------|--------|-------|
| `status` | P0 | Done | Reads all queue directories, prints summary + STOP state + latest log. Supports `--json`. |
| `list-instructions` | P0 | Done | Lists all pending manager instructions with project, command, age, ID. Supports `--json`. |
| `list-reports` | P0 | Done | Lists all pending executor reports with project, task description, age, ID. Supports `--json`. |
| `next-report` | P0 | Done | Shows oldest outbox file with metadata, validation warnings. Supports `--json`. |
| `write-instruction` | P0 | Done | Reads stdin, validates, writes to inbox with generated filename. Supports `--archive-source <id>`. |
| `write-report` | P2 | Done | Reads stdin, validates required project_state fields, auto-fills envelope (id, direction, type, created_at, from, to), writes to outbox. Supports `--ack-source <id>`. |
| `next-instruction` | P1 | Done | Shows oldest inbox file with metadata. Supports `--json`. |
| `decisions` | P1 | Done | Lists pending and answered decisions (last 10). Supports `--json`. |
| `stop` | P1 | Done | Writes STOP with interactive confirmation. |
| `resume` | P1 | Done | Removes STOP with interactive confirmation. |
| `archive` | P1 | Done | Moves message to archive/sent or archive/received by origin. |
| `ack` | P1 | Done | Alias for archive with ack semantics. |
| `archive-all` | P2 | Done | Bulk archive all messages in a queue (`reports`, `instructions`, `decisions`) with y/N confirmation. |

**13 commands implemented.** All P0, P1, and P2 complete.

---

## 3. Remaining Quality Gaps

| Gap | Priority | Status | Description |
|-----|----------|--------|-------------|
| `jsonschema` validation | Medium | Not started | Current validation is manual string checks. A proper `jsonschema` import would validate against the 4 schema files. |
| Bulk archive by age | Low | Not started | No command to archive messages older than N days. Current `archive-all` archives everything. |

---

## 4. Minimal Workflow to Support

The tool already supports this loop end-to-end:

```
CLI → bacp-bridge status          # show bridge health
CLI → bacp-bridge next-report     # show pending report
Human → copy to ChatGPT           # manager reads
Human → pipe to write-instruction # manager responds
CLI → bacp-bridge next-instruction  # read instruction
CLI → act on instruction          # do the work
CLI → bacp-bridge ack <id>        # archive consumed message
CLI → write report to outbox/     # report results
CLI → bacp-bridge archive <id>    # archive report
```

The loop works. The only missing piece is **automated polling** (Phase 3) and **direct API integration** (Phase 4) which are out of scope.

---

## 5. Exact Implementation Steps (Remaining)

### Step 1 — jsonschema Validation (not started)

Add proper schema validation using Python's built-in capabilities (no external dependency):

- Add a `--validate` flag to `write-instruction` that also validates against the schema file
- Add validation warnings to `next-report` and `next-instruction` display
- Keep manual validation as fallback when schema files are missing

Implementation:
- `scripts/bacp-bridge`: ~30 additional lines for schema file loading and validation
- No new files needed

### Step 2 — README Update (not started)

Replace the current README's discovery-phase focus with tool documentation:

- Installation: `chmod +x scripts/bacp-bridge`
- Quick start: the 7-step workflow above
- Command reference table
- Environment variables (BACP_ROOT)

Implementation:
- `README.md`: rewrite to focus on tool usage

### Step 3 — Consolidation Cleanup (not started)

Remove or mark discovery-phase docs that are out of scope:

- `CLAUDE.md`: Remove discovery pass references, watcher-health references
- Mark watcher-health doc as superseded by tool spec
- Remove cross-project-collaboration doc from active navigation

Implementation:
- Approx 3-5 file edits, all in docs/

### Step 4 — End-to-End Verification (not started)

### Step 5 — End-to-End Verification

Run the full loop once with ChatGPT:

1. Write a report to outbox
2. `bacp-bridge next-report` → copy to ChatGPT
3. Create a fake ChatGPT response
4. `echo '<response>' | bacp-bridge write-instruction`
5. `bacp-bridge next-instruction` → read it
6. `bacp-bridge ack <id>` → archive it
7. `bacp-bridge status` → verify

No new code needed. Validates the existing implementation works.

---

## 6. Test Plan

| Test | Method | Pass Criteria |
|------|--------|---------------|
| Status shows correct counts | Run on known queue state | Correct inbox/outbox/decision counts |
| Stop/resume | Run stop, then resume | STOP appears then disappears |
| Write-instruction validation | Pipe invalid JSON | Rejected with specific error message |
| Write-instruction creates file | Pipe valid instruction | File appears in inbox/ with correct name |
| Ack archives message | Ack an inbox file | File moves to archive/received/ |
| Archive cross-directory | Ack by filename substring | File found regardless of source directory |
| Empty queue handling | Run on empty directories | Prints "No pending messages", exits 0 |
| Guard against stopped bridge | Run while STOP exists | Prints "Bridge is HALTED", exits 1 |

All manual. No automated test framework.

---

## 7. Acceptance Criteria

The tool is complete when:

1. A human can complete the full manager↔CLI loop without touching directories or filenames
2. Every command produces clear output or actionable error messages
3. No message is ever deleted (archive only)
4. The kill switch reliably halts operations
5. The audit log records every command
6. Invalid JSON is rejected before it enters the message queue
7. The README documents the full workflow in under 5 minutes of reading

Criteria 1-6 are met now. Criterion 7 (README) is the only remaining item.

---

## 8. What Remains Out of Scope

| Area | Status | Rationale |
|------|--------|-----------|
| Automated polling daemon | OUT | Phase 3. Not needed until CLI runs unattended. |
| ChatGPT API integration | OUT | Phase 4. Not needed until human is fully eliminated from loop. |
| Dashboard / UI | OUT | The tool is a CLI. Dashboard reads the same filesystem. |
| GitHub integration | OUT | The bridge manages instructions, not code. |
| Watcher health monitoring | OUT | Removed per manager decision. Not a bridge concern. |
| Project discovery | OUT | No vault, Portal, AppKit, MSDK, or Router inspection. |
| Cross-project collaboration | OUT | The bridge controls the CLI, not external projects. |
| External project changes | OUT | Never modify managed projects. |
| Automated tests | OUT | The tool is simple enough for manual verification. |
| Package distribution | OUT | Single script, no install step needed. |
