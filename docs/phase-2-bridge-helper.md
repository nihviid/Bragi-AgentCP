# Phase 2 — Bridge Helper Design

**Date:** 2026-05-03
**Status:** Design (pre-implementation)
**Phase:** 2 of 4 (Local Bridge Helper)

---

## 1. Purpose

Phase 1 established the bridge directory and structured message schemas, but all message transport between ChatGPT and the CLI executor depends on the human manually navigating `~/.bragi/agent-control-plane/` directories, remembering filenames, copying file contents between terminals, and running archive operations by hand.

Phase 2 introduces a bridge helper CLI tool (`bacp-bridge`) that automates all filesystem-level operations. The tool:

- Reads pending instructions from `inbox/manager-to-cli/`
- Reads pending reports from `outbox/cli-to-manager/`
- Archives processed messages
- Validates messages against JSON schemas
- Shows pending decisions
- Surfaces exactly what needs to be pasted to ChatGPT
- Handles the kill switch

**Phase 2 does not eliminate the human from the loop.** The human still transports messages between ChatGPT and the bridge. But the human no longer navigates directories, constructs filenames, or remembers file paths. The bridge helper surfaces the right thing at the right time.

---

## 2. Problem Solved

### Before Phase 2 (Pure Phase 1)

```
1. CLI writes status to outbox/cli-to-manager/<timestamp>~<project>~<role>~<pid>~<nonce>.status.json
2. Human must: find the file, open it, copy content, paste to ChatGPT
3. ChatGPT responds
4. Human must: create new file in inbox/manager-to-cli/ with correct filename
5. Human must: remember to archive processed messages
6. Human must: navigate directories manually
```

### After Phase 2

```
1. CLI writes status to outbox/ (unchanged)
2. Human runs: bacp-bridge next-report → tool prints report to terminal
3. Human copies terminal output → pastes to ChatGPT
4. ChatGPT responds
5. Human runs: bacp-bridge write-instruction <file> <manager-id> → tool writes file + archives old
6. Human runs: bacp-bridge status → sees current state
```

**The bridge helper eliminates:** directory navigation, filename construction, manual archival, manual validation, manual decision scanning.

---

## 3. Non-Goals

- **Not a ChatGPT API client.** Phase 2 does not call any external API. ChatGPT communication remains manually mediated.
- **Not a daemon or background service.** Phase 2 is a CLI tool invoked by the human or the executor. No polling, no cron.
- **Not a dashboard.** Phase 2 has no UI. It is a terminal CLI.
- **Not a watcher agent replacement.** Existing `/loop 60s` watchers are unchanged.
- **Not a cross-repo messaging tool.** The existing `bragi-vault/messages/` protocol is unchanged.
- **Not a CI/CD integration.** Phase 2 does not interact with GitHub Actions or any CI system.

---

## 4. Directory Inputs and Outputs

The bridge helper operates on the same directory structure as Phase 1. It is a tool that reads and writes to these directories, not a replacement for them.

```
~/.bragi/agent-control-plane/
│
├── inbox/
│   ├── manager-to-cli/          # READ: bridge helper reads pending instructions
│   └── human-to-cli/            # READ: bridge helper reads human overrides
│
├── outbox/
│   ├── cli-to-manager/          # READ: bridge helper surfaces pending reports
│   └── cli-to-human/            # READ: bridge helper surfaces human-bound messages
│
├── archive/
│   ├── sent/                    # WRITE: bridge helper moves processed outbox files here
│   └── received/                # WRITE: bridge helper moves processed inbox files here
│
├── decisions/
│   ├── pending/                 # READ: bridge helper lists pending decisions
│   └── answered/                # READ: bridge helper checks for answers
│
├── logs/
│   └── bridge.log               # WRITE: bridge helper appends audit entries
│
├── schemas/                     # READ: bridge helper validates messages against schemas
│   └── manager-control/
│       ├── project-state.schema.json
│       ├── executor-report.schema.json
│       ├── manager-instruction.schema.json
│       └── decision-request.schema.json
│
└── STOP                         # READ: bridge helper checks for kill switch
```

### Key Principle

The bridge helper never creates original content. It:
- **Reads** pending messages from inbox/outbox
- **Displays** them to stdout (for copy/paste to ChatGPT)
- **Writes** acknowledgment and archive entries
- **Validates** messages against schemas
- **Reports** directory state on request

The executor writes reports. The human (or future relay) writes instructions. The bridge helper manages the filesystem around them.

---

## 5. Bridge Helper Responsibilities

| Responsibility | Details | Priority |
|---------------|---------|----------|
| **Surface pending reports** | Show the next unprocessed executor report with full content | P0 — core loop |
| **Surface pending instructions** | Show the next unprocessed manager instruction | P0 — core loop |
| **Write instructions** | Accept manager instruction JSON, validate, write to inbox with correct filename | P0 — core loop |
| **Archive messages** | Move processed messages from inbox/outbox to archive/sent|received | P0 — housekeeping |
| **Validate schemas** | Check JSON messages against schema definitions before writing | P1 — quality |
| **List decisions** | Show pending and answered decisions | P1 — visibility |
| **Kill switch check** | Check STOP presence, report status, handle stop/resume commands | P1 — safety |
| **Audit logging** | Update bridge.log for every operation | P1 — traceability |
| **Project routing** | Filter messages by project when displaying | P2 — convenience |
| **Status summary** | Show aggregate state of all bridge directories | P2 — situational awareness |

---

## 6. Message Lifecycle (Phase 2)

```
CLI writes report:
  1. CLI executor writes status JSON to outbox/cli-to-manager/
  2. Human runs: bacp-bridge next-report
  3. Bridge helper reads newest file, prints full content to stdout
  4. Human copies output → pastes to ChatGPT
  5. ChatGPT responds with instruction JSON
  6. Human saves response to local file (e.g., /tmp/response.json)
  7. Human runs: bacp-bridge write-instruction /tmp/response.json
  8. Bridge helper validates JSON against schema
  9. Bridge helper constructs correct filename → writes to inbox/manager-to-cli/
  10. Bridge helper archives the original report (moves to archive/sent/)
  11. Bridge helper appends audit entry to logs/bridge.log

Manager writes instruction:
  1. (Same cycle but reversed — instruction → ack → execution → report)
```

### Archive Rules

- Messages are archived when their counterpart has been written
- Archive preserves original filename for traceability
- Archive is write-once — never modify archived messages
- Archived messages older than 90 days are safe to delete

---

## 7. CLI Command Design

### Command Overview

```
bacp-bridge status              # Show bridge directory state
bacp-bridge next-report         # Show the next pending executor report
bacp-bridge next-instruction    # Show the next pending manager instruction
bacp-bridge write-instruction   # Write a manager instruction to inbox
bacp-bridge archive <id>        # Archive a specific message by ID
bacp-bridge decisions           # List pending and answered decisions
bacp-bridge stop                # Activate kill switch (STOP file)
bacp-bridge resume              # Deactivate kill switch (remove STOP)
```

### Command Details

#### `bacp-bridge status`

Shows aggregate state of the bridge.

```
$ bacp-bridge status
Bridge: ACTIVE
STOP: not present
Pending reports: 1 (20260503T123000Z~bacp~executor~p00432~d7e1 → portal)
Pending instructions: 0
Pending decisions: 0
Archived today: 7 messages
Last audit: 2026-05-03T12:31:00Z
```

#### `bacp-bridge next-report`

Reads the newest file from `outbox/cli-to-manager/` and prints it to stdout with a header.

```
$ bacp-bridge next-report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PENDING REPORT: 20260503T123000Z~bacp~executor~p00432~d7e1
Project: bacp | Direction: cli_to_manager | Type: status_report
Created: 2026-05-03T12:30:00+02:00 | Requires response: yes
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

=== PROJECT_STATE_SNAPSHOT ===
{
  "project": "bacp",
  "timestamp": "2026-05-03T12:30:00Z",
  ...
}

=== FULL REPORT ===
{ ... }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 COPY THIS TO CHATGPT → paste the FULL REPORT section
 After response: save to file → bacp-bridge write-instruction <file>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### `bacp-bridge next-instruction`

Reads the newest file from `inbox/manager-to-cli/` and prints it.

```
$ bacp-bridge next-instruction
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PENDING INSTRUCTION: 20260503T120030Z~appkit~manager~p00000~b4d1
Project: appkit | Command: continue
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

{ ... full instruction JSON ... }
```

#### `bacp-bridge write-instruction <path>`

Takes a file path containing a manager instruction JSON, validates it, writes it to `inbox/manager-to-cli/` with the correct filename, and archives the report it responds to.

```
$ bacp-bridge write-instruction /tmp/manager-response.json
✓ Validated against schema (manager-instruction.schema.json)
✓ Written to: ~/.bragi/agent-control-plane/inbox/manager-to-cli/20260503T120030Z~appkit~manager~p00000~b4d1.instruction.json
✓ Archived: outbox/cli-to-manager/20260503T120000Z~appkit~executor~p12345~a8f3.status.json → archive/sent/
✓ Logged: logs/bridge.log
```

**Validation performed:**
1. JSON is valid (parseable)
2. Required fields present (`id`, `project`, `direction`, `type`, `command`, `from`, `to`, `created_at`, `payload`)
3. `payload.model_selection` is present with all required sub-fields
4. `command` is a valid command from the enum
5. Filename is constructed from the `id` field + `.json`

**On validation failure:**
```
$ bacp-bridge write-instruction /tmp/bad-response.json
✗ Validation failed: missing required field "payload.model_selection"
Error at line 42: model_selection must include manager_model, executor_model, memory_action, reason
File not written. Fix the JSON and try again.
```

#### `bacp-bridge archive <id>`

Archives a specific message by its ID (partial match supported — enough to uniquely identify).

```
$ bacp-bridge archive 20260503T123000Z~bacp~executor~p00432~d7e1
✓ Found: outbox/cli-to-manager/20260503T123000Z~bacp~executor~p00432~d7e1.status.json
✓ Archived: → archive/sent/
✓ Logged: logs/bridge.log
```

#### `bacp-bridge decisions`

Lists pending and answered decisions.

```
$ bacp-bridge decisions
PENDING:
  (none)

ANSWERED:
  20260503T130000Z~portal~executor~p12345~c7e2 — "Should we use REST or gRPC" — resolved 2026-05-03
```

#### `bacp-bridge stop`

Activates the kill switch by writing the STOP file.

```
$ bacp-bridge stop "Pausing bridge for system maintenance"
⚠ WARNING: This will halt all bridge operations.
  Are you sure? [y/N]: y
✓ STOP written to: ~/.bragi/agent-control-plane/STOP
✓ Logged: logs/bridge.log
Bridge is now HALTED. Run 'bacp-bridge resume' to restore.
```

#### `bacp-bridge resume`

Deactivates the kill switch by removing the STOP file.

```
$ bacp-bridge resume
✓ STOP removed
✓ Logged: logs/bridge.log
Bridge is now ACTIVE.
```

---

## 8. Validation Behavior

### What Gets Validated

| Command | Validates | Schema Used |
|---------|-----------|-------------|
| `write-instruction` | Input JSON is a valid manager instruction | `manager-instruction.schema.json` |
| `next-report` | Displayed file matches executor report schema (read-only validation) | `executor-report.schema.json` |
| `next-instruction` | Displayed file matches manager instruction schema (read-only validation) | `manager-instruction.schema.json` |
| `decisions` | Decision request and response files match schemas | `decision-request.schema.json` |

### Validation Level

- **Phase 2 validates structure only** — required fields exist, types match, enums are valid.
- **Phase 2 does NOT validate semantic correctness** — whether the instruction is appropriate for the current state is the manager's responsibility.
- **Validation errors are warnings, not blocks** — the tool reports the issue but does not prevent reading. For `write-instruction`, validation failure prevents writing.

### Error Handling

| Error Type | Behavior |
|-----------|----------|
| File not found | Print error, exit code 1 |
| Invalid JSON | Print parse error with line number, exit code 1 |
| Schema violation | Print field-level violations, exit code 1 |
| Schema file missing | Print warning, proceed without validation, exit code 0 |
| STOP exists (non-stop command) | Print "Bridge is HALTED", exit code 1 |

---

## 9. Archive Behavior

### When Archival Happens

| Trigger | What Gets Archived |
|---------|-------------------|
| `write-instruction` processes a report | The report moves from `outbox/` to `archive/sent/`. The instruction goes to `inbox/`. |
| CLI acknowledges an instruction | The instruction moves from `inbox/` to `archive/received/`. |
| `archive <id>` called manually | The matching message moves to archive. |
| Decision is answered | The decision moves from `decisions/pending/` to `decisions/answered/`. |

### Archive Rules

- Archive filenames preserve the original message ID
- Archives are never modified, only created
- The `archive` command refuses to overwrite existing files
- Archive directories can be safely cleaned for messages older than 90 days
- The bridge log records every archive operation with direction and message ID

### Archive Naming

Messages keep their original filename in the archive:

```
archive/sent/20260503T120000Z~appkit~executor~p12345~a8f3.status.json
archive/received/20260503T120030Z~appkit~manager~p00000~b4d1.instruction.json
```

---

## 10. Kill Switch Behavior

### Command: `bacp-bridge stop`

1. The `stop` command requires confirmation (`y/N`)
2. On confirmation, writes a `STOP` file to `~/.bragi/agent-control-plane/STOP`
3. The STOP file contains timestamp, reason, and invocation command
4. Logs the event to `logs/bridge.log`
5. All subsequent `bacp-bridge` commands (except `resume` and `status`) refuse to operate
6. Exit code: 1 for any blocked operation

### Command: `bacp-bridge resume`

1. Checks if `STOP` file exists. If not, prints "Bridge is already active."
2. Deletes the STOP file
3. Appends resume entry to `logs/bridge.log`
4. Bridge is now active — all commands resume normal operation

### Context: `bacp-bridge status`

When STOP is present:
```
$ bacp-bridge status
Bridge: HALTED (stopped by human at 2026-05-03T12:35:00Z)
Reason: Pausing bridge for system maintenance
Run 'bacp-bridge resume' to restore.
Pending reports: 0
Pending instructions: 1
Pending decisions: 1
```

### Context: Non-stop Commands with STOP Present

Any command other than `status`, `stop`, or `resume` when STOP exists:
```
$ bacp-bridge next-report
Bridge is HALTED. No operations permitted.
Run 'bacp-bridge status' for details.
Run 'bacp-bridge resume' to restore.
```

---

## 11. Human/Manual ChatGPT Handoff Behavior

### The Handoff Flow

Phase 2 does not eliminate the human. Instead, it makes the handoff explicit and structured.

```
1. CLI writes report
2. Human runs: bacp-bridge next-report
3. Tool prints: "COPY THIS TO CHATGPT" banner + full report JSON
4. Human copies terminal output → opens ChatGPT → pastes
5. ChatGPT processes and responds with instruction JSON
6. Human saves response: cat > /tmp/bridge-response.json
7. Human runs: bacp-bridge write-instruction /tmp/bridge-response.json
8. Tool validates, writes, archives, logs
9. Human runs: bacp-bridge status → confirms message delivered
```

### How the Tool Makes the Handoff Obvious

Every `next-report` and `next-instruction` output includes a clear visual banner:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 📋 COPY TO CHATGPT
   Copy the FULL REPORT section above.
   After ChatGPT responds, save to a file and run:
   bacp-bridge write-instruction <filename>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### ChatGPT Side

The ChatGPT manager receives structured JSON. It processes it using its system prompt's bridge instructions and responds with a structured instruction JSON that the human saves and the tool validates.

### Reducing, Not Eliminating

Phase 2 reduces the handoff from 6 steps (Phase 1) to 4 steps:
1. Copy from tool output
2. Paste to ChatGPT
3. Copy from ChatGPT
4. Run `write-instruction`

The tool eliminates directory navigation, filename construction, schema validation, archival, and audit logging from the human's responsibility. Only the copy/paste remains.

---

## 12. Future API-Connected Behavior

Phase 4 (not Phase 2) will add ChatGPT API integration. The bridge helper design accounts for this transition.

### How Phase 2 Prepares for Phase 4

| Phase 2 Feature | Phase 4 Use |
|----------------|-------------|
| Message validation | Validated messages are safe to forward to API |
| Structured JSON output | API-ready payload format |
| Archive system | Complete history available for context injection |
| Audit log | Usage tracking, debugging, billing |
| Kill switch | Safety mechanism for API-connected operation |
| Decision tracking | Context for autonomous decision-making |

### Phase 4 Extension Points

The `write-instruction` command is the natural insertion point for API integration:

- **Phase 2:** `bacp-bridge write-instruction /tmp/response.json` (validates, writes file)
- **Phase 4:** `bacp-bridge write-instruction /tmp/response.json --relay` (validates, writes file, forwards to API)

The `--relay` flag would:
1. Validate the instruction (same as Phase 2)
2. Write it to inbox (same as Phase 2)
3. Forward it to ChatGPT API for direct processing (new)
4. Receive the response and write it back to outbox (new)

This keeps Phase 2's interface stable while adding the API layer later.

---

## 13. Future Dashboard Behavior

### How Phase 2 Prepares for Phase 3

| Phase 2 Feature | Phase 3 Dashboard Use |
|----------------|----------------------|
| `bacp-bridge status` output | Dashboard status panel (reads same data) |
| Directory structure | Dashboard reads directly from bridge filesystem |
| Archive system | Dashboard timeline view |
| Audit log | Dashboard activity feed |
| Decision tracking | Dashboard decision inbox |
| Schema validation | Dashboard knows data format |

### Dashboard Integration Point

The dashboard reads the same directories the bridge helper reads. No API needed:

```
Dashboard reads:
  outbox/cli-to-manager/       → Active tasks panel
  inbox/manager-to-cli/        → Pending instructions panel
  decisions/pending/           → Decision inbox
  decisions/answered/          → Decision history
  logs/bridge.log              → Activity timeline
  STOP                         → Kill switch status
```

The bridge helper's `status` command outputs the same data the dashboard would display. The dashboard is a visual wrapper around `bacp-bridge status` + file reads.

---

## 14. Security Constraints

| Constraint | Reason | Phase 2 Compliance |
|-----------|--------|-------------------|
| No external API calls | Bridge stays local. No data exfiltration risk. | Compliant |
| No network access required | Works offline. Does not call GitHub, ChatGPT, or any remote. | Compliant |
| No credentials stored | No API keys, tokens, or passwords in bridge helper config. | Compliant |
| Kill switch is a file | Human-readable. Survives process death. No ambiguity. | Compliant |
| Write-once archive | Archived messages are never modified. Tamper evidence. | Compliant |
| Schema validation | Malformed JSON is rejected before writing to inbox. | Compliant |
| STOP overrides all | If STOP exists, no bridge operations permitted. | Compliant |
| No auto-delete | The tool never deletes files. Only moves to archive. | Compliant |

### What Phase 2 Does NOT Secure

- **Message content** is plain JSON on disk. No encryption. (Trusted local environment assumption.)
- **Authentication** — the tool does not authenticate CLI users. (Single-user system.)
- **Rate limiting** — no throttling on commands. (Human-paced interaction.)

---

## 15. Minimal Implementation Plan

### Phase 2 Steps

| Step | What | Dependencies |
|------|------|--------------|
| 2.1 | Implement `bacp-bridge status` — read bridge directories, report state | Directory structure exists |
| 2.2 | Implement `bacp-bridge next-report` — read newest outbox file, display with banner | Outbox directory exists |
| 2.3 | Implement `bacp-bridge next-instruction` — read newest inbox file, display | Inbox directory exists |
| 2.4 | Implement `bacp-bridge write-instruction` — validate JSON, write to inbox with correct filename, auto-archive source | Next-report exists |
| 2.5 | Implement `bacp-bridge archive` — move message to archive/sent|received | Archive directories exist |
| 2.6 | Implement `bacp-bridge decisions` — read decisions/pending/ and decisions/answered/ | Decision directories exist |
| 2.7 | Implement `bacp-bridge stop` / `bacp-bridge resume` — write/remove STOP file | n/a |
| 2.8 | Add schema validation to `write-instruction` | Schema files exist |
| 2.9 | Add schema validation to `next-report` and `next-instruction` | Schema files exist |
| 2.10 | Add audit logging to all commands | bridge.log exists |
| 2.11 | End-to-end testing — full loop with manual ChatGPT handoff | Steps 2.1–2.10 |
| 2.12 | Documentation — usage guide in `docs/phase-2-bridge-helper.md` | This document |

### Implementation Priority

P0 (must have for Phase 2 to be useful):
- `status`, `next-report`, `write-instruction`

P1 (should have):
- `next-instruction`, `archive`, `stop`, `resume`

P2 (nice to have):
- `decisions`, schema validation on display commands

### Implementation Language Options

| Option | Pros | Cons | Recommended For |
|--------|------|------|-----------------|
| **Bash script** | Zero dependencies. Ships with macOS. Simple file operations. | JSON parsing is painful. Schema validation is hard. | P0 commands only |
| **Python script** | `json` module. `jsonschema` for validation. Cross-platform. Fast file ops. | Requires Python 3. | P0–P2 with validation |
| **Go binary** | Single binary. Fast. JSON support built-in. | Requires Go toolchain. | Future if distribution needed |

**Recommendation:** Python 3 script. The `jsonschema` library makes schema validation straightforward. JSON handling is native. No compilation step needed. Building in `scripts/bacp-bridge/`.

---

## 16. Open Questions

| Question | Impact | Needs Answer From |
|----------|--------|-------------------|
| Should `bacp-bridge` be a single Python script or multiple commands? | Complexity vs. simplicity | Developer preference |
| Should `next-report` auto-archive after display, or require explicit `archive` call? | Convenience vs. control | Manager |
| Should `write-instruction` require the `reply_to` field to match a known report ID? | Data integrity vs. workflow speed | Manager |
| Should the tool validate that the instruction's `project` matches the report it's responding to? | Routing correctness | Manager |
| Should there be a `--format markdown` option for prettier ChatGPT pasting? | Readability vs. machine-friendliness | Human |
| Should the tool be installed globally (PATH) or run from the BACP repo? | Convenience vs. discoverability | Human |
| Should bulk archive (archive all messages older than N days) be in scope for Phase 2? | Housekeeping vs. scope creep | Manager |

---

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Validate schemas on write-instruction but not on display | Prevention of bad data vs. unnecessary friction | Validation prevents corrupt instructions from entering the inbox. Display validation is informational only — the reader can handle schema issues. |
| P0: status, next-report, write-instruction | Minimum viable loop | Without these three, the bridge helper cannot complete the control loop. Everything else is additive. |
| Kill switch requires confirmation | Safety | Accidental STOP activation would halt all operations. Confirmation prevents mistakes. |
| Archive is explicit, not automatic | Data safety | Auto-archiving on read could lose messages if the reader hasn't fully processed them. Explicit `archive` or implicit archive-on-write-instruction is safer. |
