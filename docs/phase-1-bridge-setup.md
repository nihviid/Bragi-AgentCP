# Phase 1 Bridge Setup — Manual Structured Bridge

**Date:** 2026-05-03
**Status:** Operational
**Phase:** 1 of 4 (Manual Structured Bridge)

---

## 1. Purpose

Phase 1 makes the ChatGPT-to-CLI bridge operational at the filesystem level. All future CLI activity — including discovery passes and project work — reports through this bridge. Instead of unstructured terminal output that the human must parse and reformat, every status report, decision request, and instruction follows a structured JSON schema written to a standard directory layout.

The bridge does not automate anything yet. All message transport between ChatGPT and the CLI is human-mediated. But the structure is in place from day one.

---

## 2. Directory Layout

```
~/.bragi/agent-control-plane/
├── inbox/
│   ├── manager-to-cli/       # Instructions from ChatGPT → CLI
│   └── human-to-cli/         # Direct instructions from human → CLI
├── outbox/
│   ├── cli-to-manager/        # Status reports from CLI → ChatGPT
│   └── cli-to-human/          # Messages from CLI → human
├── archive/
│   ├── sent/                  # Processed outbound messages
│   └── received/              # Processed inbound messages
├── decisions/
│   ├── pending/               # Open decision requests
│   └── answered/              # Resolved decisions
├── logs/
│   └── bridge.log             # Append-only activity log
└── STOP.example              # Kill switch example (rename to STOP to activate)
```

### Purpose of Each Directory

| Directory | What Goes There | Who Writes | Who Reads |
|-----------|----------------|------------|-----------|
| `inbox/manager-to-cli/` | JSON instruction messages from ChatGPT | Human (pastes manager response) | CLI executor |
| `inbox/human-to-cli/` | Direct human override/input | Human | CLI executor |
| `outbox/cli-to-manager/` | JSON status reports from CLI | CLI executor | Human (copies to ChatGPT) |
| `outbox/cli-to-human/` | Messages or alerts directed at human | CLI executor | Human |
| `archive/sent/` | Processed outbound messages (read-only after archive) | Bridge/CLI after acknowledgement | Audit |
| `archive/received/` | Processed inbound messages (read-only after archive) | Bridge/CLI after acknowledgement | Audit |
| `decisions/pending/` | Open decision requests with options | CLI or manager | Human or manager |
| `decisions/answered/` | Resolved decisions | Manager or human | CLI |
| `logs/` | `bridge.log` — append-only audit trail | CLI | Human, audit |
| `STOP.example` | Template for kill switch | N/A (example) | N/A |

---

## 3. How the CLI Writes Status Reports

At every natural execution boundary (task complete, blocked, checkpoint reached), the CLI executor writes a JSON status report to `outbox/cli-to-manager/`.

### When to Write

- **Task complete** — always
- **Checkpoint reached** — for tasks longer than 30 minutes
- **Blocked** — immediately
- **Session start** — reports current state and any pending items
- **Session end** — final summary

### Schema

See `docs/chatgpt-cli-bridge.md` section 7 for the full schema.

### Filename Convention

```
<ISO8601-UTC>T<Z>~<project>~<role>~<pid>~<nonce>.<type>.json
```

Example components:
- Timestamp: `20260503T120000Z`
- Project: `appkit`, `portal`, `msdk`, `router`, `bacp`
- Role: `executor`, `discovery`, `manager`
- PID: `p12345` (process ID if available, otherwise `p00000`)
- Nonce: `a8f3` (4-hex random nonce for uniqueness)
- Type: `status`, `blocked`, `error`, `info`, `decision_request`

Full example:
```
20260503T120000Z~appkit~executor~p12345~a8f3.status.json
```

This convention guarantees uniqueness even with concurrent sessions on the same project.

### Required Fields in Every Status Report

| Field | Always Required? |
|-------|-----------------|
| `id` | Yes (must match filename without extension) |
| `project` | Yes |
| `direction` | Yes (`cli_to_manager`) |
| `type` | Yes |
| `from` | Yes |
| `to` | Yes |
| `created_at` | Yes (ISO 8601 with timezone) |
| `payload.current_task` | Yes |
| `payload.action_taken` | Yes (empty array is valid) |
| `payload.commands_run` | Yes (empty array is valid) |
| `payload.git_status` | Yes |
| `payload.next_recommended_action` | Yes |
| `payload.manager_instruction_needed` | Yes (boolean) |

---

## 4. How Manager Instructions Are Written Back

When ChatGPT (the manager) responds to a CLI status report, the human pastes the response into the bridge.

### Process

1. CLI writes status to `outbox/cli-to-manager/`
2. Status file includes a `BACP STATUS REPORT` section at the end for easy human reading
3. Human copies the status JSON from the file (or reads the summary section)
4. Human pastes into ChatGPT
5. Manager responds with an instruction or acknowledgment
6. Human copies manager's response
7. Human writes response to `inbox/manager-to-cli/`
8. CLI detects new file on next poll cycle
9. Original status file is moved to `archive/sent/`
10. Manager response is moved to `archive/received/` after processing

### What the CLI Checks in the Inbox

On each poll, the CLI:
1. Lists JSON files in `inbox/manager-to-cli/` sorted by creation time
2. Checks `inbox/human-to-cli/` for direct human overrides
3. Checks `decisions/answered/` for any new decisions
4. Processes the oldest unprocessed message first
5. After processing each message, moves it to `archive/received/`
6. Logs the cycle to `logs/bridge.log`

---

## 5. Message Filename Convention

All message files must follow this naming convention to guarantee uniqueness and enable future automation.

### Pattern

```
<TIMESTAMP>~<PROJECT>~<ROLE>~<PID>~<NONCE>.<TYPE>.json
```

### Fields

| Field | Format | Example | Required |
|-------|--------|---------|----------|
| Timestamp | `YYYYMMDDTHHmmssZ` (UTC) | `20260503T120000Z` | Yes |
| Project | lowercase project key | `appkit`, `portal`, `msdk`, `router`, `bacp` | Yes |
| Role | agent role identifier | `executor`, `discovery`, `manager` | Yes |
| PID | `p` + process ID or `p00000` | `p12345` | Yes |
| Nonce | 4 random hex characters | `a8f3` | Yes |
| Type | message type | `status`, `instruction`, `decision_request`, `blocked`, `error`, `info`, `ack`, `decision` | Yes |

### Examples

```
# CLI status report for AppKit work
20260503T120000Z~appkit~executor~p12345~a8f3.status.json

# Manager instruction in response
20260503T120030Z~appkit~manager~p00000~b4d1.instruction.json

# Decision request from discovery pass
20260503T130000Z~bacp~discovery~p12345~c7e2.decision_request.json
```

### Why This Format

- **Timestamp-first**: Files sort chronologically. Oldest first in directory listings.
- **Tilde separators**: No ambiguity with dots in types or dashes in timestamps.
- **Project+Role**: Enables per-project filtering for future automation and dashboards.
- **PID+Nonce**: Eliminates filename collisions even with concurrent agents.
- **File extension = message type**: Future directory scanners can filter by extension.

---

## 6. Manual Copy/Paste Fallback

The manual fallback is the default for Phase 1. It is not an emergency procedure — it is the procedure.

### How the Human Bridges CLI ↔ ChatGPT

```
CLI writes file → prints summary → human copies → ChatGPT reads → responds →
human copies response → writes file → CLI reads → continues
```

### The CLI MUST Print a BACP STATUS REPORT

At the end of every task or checkpoint, the CLI prints a structured status report that the human can copy directly. The report must include:

```
BACP STATUS REPORT — 2026-05-03
Project: appkit
Branch: feature/foo
Current Task: (one line)
Action Taken:
- bullet 1
- bullet 2
Files Created:
- path/to/file.md
Files Modified:
- path/to/other.py
Commands Run:
- git add -A
Git Status: clean / dirty (summary)
Open Questions: (list or "None")
Risks: (list or "None")
Next Recommended Action: (one line)
Needs Human Decision: yes / no
Manager Instruction Needed: yes / no
```

### What the Human Copies

The human copies the full BACP STATUS REPORT and pastes it into ChatGPT. The manager reads it and responds with an instruction or acknowledgment. The human copies the manager's response and writes it to a JSON file in `inbox/manager-to-cli/`.

---

## 7. Kill Switch Behavior

### How It Works

The file `~/.bragi/agent-control-plane/STOP` is the kill switch. If it exists, the bridge is dead.

- **STOP.example** is a template. It is pre-created but inert.
- **STOP** (without `.example`) is the active kill switch. It should NOT exist under normal operation.

### Creating STOP

To halt the bridge:
1. Copy `STOP.example` to `STOP`
2. Edit `STOP` to include the reason and initiator
3. Append a KILL entry to `logs/bridge.log`

### What Happens When STOP Exists

| Component | Behavior |
|-----------|----------|
| CLI executor | On next poll: detects STOP, logs "BRIDGE HALTED", refuses to read/write bridge messages |
| ChatGPT manager | On next interaction: informed STOP exists, refuses new bridge operations |
| Bridge (Phase 2+) | Relays detect STOP, exit immediately |

### Recovery

1. Delete `STOP`
2. Append a resume entry to `logs/bridge.log`
3. Next poll cycle resumes normal operation

### Safety Rule

**Only the human may create or delete STOP.** No agent creates or removes the kill switch.

---

## 8. Audit Log Behavior

### Log Location

`~/.bragi/agent-control-plane/logs/bridge.log`

### Log Format

```
TIMESTAMP | DIRECTION | STAGE | MESSAGE_ID | TYPE | STATUS
```

### Log Entry Types

| Prefix | Meaning | Example |
|--------|---------|---------|
| `OUT` | Message written to outbox | `2026-05-03T12:00:00Z | OUT | cli-to-manager | 20260503T120000Z~appkit~executor~p12345~a8f3 | status | PENDING` |
| `IN` | Message written to inbox | `2026-05-03T12:00:30Z | IN | manager-to-cli | 20260503T120030Z~appkit~manager~p00000~b4d1 | instruction | PENDING` |
| `ARC` | Message archived | `2026-05-03T12:01:00Z | ARC | sent | 20260503T120000Z~appkit~executor~p12345~a8f3 | status | ARCHIVED` |
| `DEC` | Decision lifecycle | `2026-05-03T13:00:00Z | DEC | pending | 20260503T130000Z~bacp~discovery~p12345~c7e2 | decision_request | OPEN` |
| `KILL` | Kill switch event | `2026-05-03T14:00:00Z | KILL | STOP | - | kill_switch | ENGAGED` |
| `ERR` | Bridge error | `2026-05-03T14:05:00Z | ERR | outbox | - | write | FAILED: disk full` |
| `INIT` | Bridge initialization | `2026-05-03T12:00:00Z | INIT | bridge-setup | - | directory-structure | CREATED` |

### Logging Rules

- Always append. Never overwrite or edit existing entries.
- One entry per bridge event (write, read, archive, kill, error).
- Entries are plain text, one per line.
- Rotate log at 10MB (old logs are safe to delete; archive contains full messages).
- The CLI executor appends to the log after every bridge operation.

---

## 9. Known Limitations

| Limitation | Impact | Planned Resolution |
|-----------|--------|-------------------|
| **Human-mediated transport** | Human must copy/paste between CLI and ChatGPT | Phase 2 bridge helper reduces friction; Phase 4 eliminates it |
| **No locking** | Concurrent CLI sessions could write conflicting status reports | Future: atomic temp-file writes, per-project outboxes, single-writer scheduler |
| **No periodic polling** | CLI only checks bridge at task boundaries | Phase 2+ optional `/loop` polling |
| **Single log file** | `bridge.log` grows unbounded | Rotate at 10MB |
| **No dashboard** | No visual overview of bridge state | Phase 3 |
| **No automated expiry** | Stale pending messages stay forever | Future: TTL field + cleanup script |
| **No encryption** | Plain JSON on disk | Out of scope (trusted local environment) |
| **Filename length** | Convention produces long filenames | Acceptable for a machine-readable directory |

---

## 10. Next Step After Setup

Phase 1 is now operational. The bridge directory exists at `~/.bragi/agent-control-plane/` with all subdirectories, a log file, and a kill switch example.

The immediate next step is to **use the bridge during discovery**. Every discovery pass produces a BACP STATUS REPORT written to `outbox/cli-to-manager/`. The discovery plan has been updated to mandate this.

**What happens next:**

| Step | When |
|------|------|
| Discovery passes execute, each reporting through the bridge | After this setup |
| Phase 2 bridge helper (reduces paste friction) | After discovery completes (optional) |
| Phase 3 dashboard (visual bridge state) | After bridge helper proves insufficient |
| Phase 4 API-connected manager | Only if approved; dashboard remains human control layer |
