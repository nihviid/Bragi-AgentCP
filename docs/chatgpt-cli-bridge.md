# ChatGPT-to-CLI Bridge

**Bridge Architecture Document**  
**Bragi Agent Control Plane (BACP)**  
**Date:** 2026-05-03  
**Branch:** `agent/bootstrap-discovery`  

---

## 1. Problem Statement

Currently, collaboration between the ChatGPT manager (high-level orchestrator) and the CLI executor (Claude Code) requires **manual copy/paste** of messages, status reports, and instructions. This is slow, error-prone, and defeats the purpose of having AI agents cooperate.

Every round-trip:
1. CLI finishes work → prints status to terminal
2. Human selects + copies terminal output
3. Human pastes into ChatGPT
4. Manager reads, thinks, writes response
5. Human selects + copies response
6. Human pastes into CLI terminal
7. CLI reads, continues

This is unsustainable for multi-project orchestration with frequent turn-taking.

---

## 2. Goals

- Eliminate manual copy/paste between ChatGPT manager and CLI executor
- Define a structured message format both sides can produce and consume
- Keep all bridge data on local filesystem (no external API calls from CLI)
- Support asynchronous operation (manager and CLI act independently)
- Enable phased automation (manual → assisted → automated)
- Provide a clear migration path to a dashboard UI
- Maintain an audit trail of all bridge communication
- Include a hard stop mechanism (kill switch)

---

## 3. Non-Goals

- **Not a real-time messaging system.** Polling is acceptable. Sub-second latency is not required.
- **Not a replacement for `bragi-vault/messages/`.** The vault message system handles cross-repo developer communication. The bridge handles manager↔CLI operational communication.
- **Not a CI/CD pipeline.** Bridge messages are about what to do and what was done, not about deployment artifacts.
- **Not a dashboard.** The bridge is the data layer. A future dashboard will consume bridge data.
- **Not an external API.** The bridge lives on the local filesystem. No outbound webhooks, no cloud sync from the CLI side. The CLI never calls out.
- **Not a security boundary.** The bridge assumes a trusted local environment. Messages are plain JSON on disk.

---

## 4. Why Terminal Scraping Is Avoided

Terminal scraping (parsing CLI stdout to extract structured data) was considered and rejected:

| Problem | Why It Fails |
|---------|-------------|
| **Fragile output** | CLI output varies by model, mode, and task. No consistent format to parse. |
| **No acknowledgment** | You cannot tell if the manager read the scraped output. |
| **Race conditions** | Scraping half-written output produces corrupted state. |
| **No history** | Terminal scrollback is ephemeral. No audit trail. |
| **Single channel** | Cannot distinguish status reports from error messages from human queries. |
| **No routing** | All output goes to one terminal. Cannot direct to different agents. |

**File-based structured messaging solves all of these:** deterministic schemas, explicit acknowledgments, complete history, multi-channel routing, and survives restarts.

---

## 5. File-Based Bridge Architecture

```
┌──────────────────────┐         ┌──────────────────────┐
│   ChatGPT Manager    │         │    CLI Executor      │
│   (orchestrates)     │         │   (Claude Code)      │
└──────┬───────────────┘         └──────────┬───────────┘
       │                                    │
       │  Writes instructions               │  Writes status
       │  to inbox/manager-to-cli/          │  to outbox/cli-to-manager/
       │                                    │
       │  Reads status from                  │  Reads instructions
       │  outbox/cli-to-manager/            │  from inbox/manager-to-cli/
       │                                    │
       │  Both read/write                   │  Both read/write
       │  decisions/                        │  decisions/
       │                                    │
       └──────────────┬─────────────────────┘
                      │
                      ▼
         ~/.bragi/agent-control-plane/
         (local filesystem, all agents read/write)
```

Both ChatGPT (via copy/paste in Phase 1, or a relay in later phases) and the CLI executor read from and write to the same directory structure on the local filesystem.

---

## 6. Directory Layout

```
~/.bragi/agent-control-plane/
│
├── inbox/
│   ├── manager-to-cli/          # Instructions from ChatGPT → CLI
│   └── human-to-cli/            # Direct instructions from human → CLI
│
├── outbox/
│   ├── cli-to-manager/          # Status reports from CLI → ChatGPT
│   └── cli-to-human/            # Messages from CLI → human
│
├── archive/
│   ├── sent/                    # Processed outbound messages
│   └── received/                # Processed inbound messages
│
├── decisions/
│   ├── pending/                 # Open decision requests
│   └── answered/                # Resolved decisions
│
├── logs/
│   └── bridge.log               # Append-only activity log
│
└── STOP                         # Kill switch — if present, bridge is dead
```

### Why `~/.bragi/` instead of inside `bragi-vault/`

- The control plane is a separate concern from the knowledge vault
- The bridge must work even if the vault is unavailable
- Clean separation: vault = knowledge, `~/.bragi/` = operational control
- Survives vault restructuring without bridge disruption

---

## 7. CLI-to-Manager Message Schema

Status reports from the CLI executor back to the ChatGPT manager after completing work.

```json
{
  "id": "2026-05-03T120000-appkit-001",
  "project": "appkit",
  "direction": "cli_to_manager",
  "type": "status_report",
  "from": "executor",
  "to": "manager",
  "requires_response": true,
  "created_at": "2026-05-03T12:00:00+02:00",
  "payload": {
    "current_task": "Define ChatGPT-to-CLI bridge architecture",
    "action_taken": [
      "Explored vault structure and message system",
      "Reviewed existing watcher agent conventions",
      "Drafted bridge architecture design"
    ],
    "files_created": [
      "docs/chatgpt-cli-bridge.md"
    ],
    "files_modified": [],
    "commands_run": [
      "git init && git checkout -b agent/bootstrap-discovery",
      "mkdir -p docs && mkdir -p docs/discovery",
      "git add -A && git commit..."
    ],
    "git_status": "3 files staged, clean working tree",
    "open_questions": [
      "Should the bridge live in ~/.bragi/ or inside the BACP repo?",
      "What is the exact polling interval for Phase 0?"
    ],
    "risks": [
      "Multiple concurrent CLI sessions could write conflicting status reports"
    ],
    "next_recommended_action": "Write discovery plan for vault mapping",
    "manager_instruction_needed": false
  }
}
```

### Message Types (CLI → Manager)

| `type` | When | `requires_response` |
|--------|------|-------------------|
| `status_report` | Task completed or checkpoint reached | `true` |
| `blocked` | Cannot proceed without manager input | `true` |
| `error` | Non-recoverable failure | `true` |
| `information` | FYI, no action needed | `false` |
| `decision_request` | Needs human or manager decision | `true` |

---

## 8. Manager-to-CLI Instruction Schema

Instructions from the ChatGPT manager back to the CLI executor.

```json
{
  "id": "2026-05-03T120030-appkit-001-response",
  "reply_to": "2026-05-03T120000-appkit-001",
  "project": "appkit",
  "direction": "manager_to_cli",
  "type": "instruction",
  "from": "manager",
  "to": "executor",
  "created_at": "2026-05-03T12:00:30+02:00",
  "payload": {
    "decision": "continue",
    "instruction": "Create docs/chatgpt-cli-bridge.md. Design only. Do not implement yet.",
    "model_selection": {
      "manager_model": "sonnet",
      "executor_model": "sonnet",
      "memory_action": "keep",
      "reason": "bounded documentation task"
    },
    "constraints": [
      "Do not modify Router, Portal, MSDK, or AppKit",
      "Do not implement code yet",
      "Single design document only"
    ],
    "success_criteria": [
      "Problem statement is clear",
      "Message schemas are defined",
      "Directory layout is specified",
      "Phased implementation plan exists"
    ],
    "context_links": [
      "bragi-vault/CLAUDE.md",
      "bragi-vault/templates/message.md"
    ]
  }
}
```

### Message Types (Manager → CLI)

| `type` | When |
|--------|------|
| `instruction` | New task or work item |
| `acknowledgment` | Confirms receipt of a status report |
| `clarification` | Answers an open question |
| `decision` | Resolves a decision request |
| `override` | Changes priority or cancels current work |
| `kill` | Emergency stop (also writes STOP file) |

---

## 9. Decision Request Schema

When the CLI needs a decision from the manager or human.

```json
{
  "id": "2026-05-03T130000-portal-002",
  "project": "portal",
  "direction": "cli_to_manager",
  "type": "decision_request",
  "from": "executor",
  "to": "manager",
  "created_at": "2026-05-03T13:00:00+02:00",
  "payload": {
    "title": "Should discovery scan Portal test files?",
    "context": "Portal has 87 test files. Discovery Pass 6 must decide whether to read them all or only the CI workflow files.",
    "options": [
      {
        "label": "Read all test config + CI workflows",
        "effort": "30 min",
        "pros": ["Complete picture", "Understand test infrastructure"],
        "cons": ["Slower pass", "Many files irrelevant to discovery goals"]
      },
      {
        "label": "Only CI workflow files + test runner configs",
        "effort": "10 min",
        "pros": ["Faster", "Focus on CI pipeline topology"],
        "cons": ["Misses individual test conventions"]
      }
    ],
    "recommendation": "Option 2 — test runner patterns emerge from CI configs",
    "deadline": "2026-05-04",
    "blocked_tasks": ["Pass 6 — GitHub Workflows"]
  }
}
```

Decision requests go to `decisions/pending/`. When answered, the response goes to `decisions/answered/` with the manager's choice.

---

## 10. Message Lifecycle

```
CREATED → PENDING → ACKNOWLEDGED → ARCHIVED
              │                        │
              └──→ EXPIRED             └──→ DELETED
```

```
CLI writes status report:
  1. Write JSON to `outbox/cli-to-manager/20260503T120000Z~appkit~executor~p12345~a8f3.status.json`
  2. File is in PENDING state
  3. Human or relay sees the file
  4. File content is presented to ChatGPT manager
  5. Manager writes response to `inbox/manager-to-cli/20260503T120030Z~appkit~manager~p00000~b4d1.instruction.json`
  6. Original status file is ACKNOWLEDGED
  7. Both files move to archive/ (sent/ and received/)
  8. Archive is write-only — never rewritten, only appended

Manager writes instruction:
  1. Write JSON to `inbox/manager-to-cli/20260503T120030Z~appkit~manager~p00000~b4d1.instruction.json`
  2. CLI polls inbox, finds instruction
  3. CLI reads instruction, begins work
  4. CLI writes status report back (cycle continues)
  5. Original instruction moves to archive/received/
```

### State transitions

| State | Meaning | File Location |
|-------|---------|---------------|
| `pending` | Written but not yet read | `inbox/` or `outbox/` |
| `acknowledged` | Counterpart has responded | Stays in place until archive |
| `archived` | Processed, preserved for audit | `archive/sent/` or `archive/received/` |
| `expired` | Stale, no response within TTL | `archive/` with note |

---

## 11. Polling Behavior

### CLI Side

The CLI executor polls the bridge at natural execution boundaries:

| Trigger | What Happens |
|---------|-------------|
| **Session start** | Reads `inbox/manager-to-cli/`, `inbox/human-to-cli/`, `decisions/answered/`, checks for `STOP` |
| **Task complete** | Writes status report to `outbox/cli-to-manager/`, then polls for response |
| **Blocked** | Writes `type: blocked` to `outbox/cli-to-manager/`, sets `requires_response: true`, waits |
| **Before any new work** | Checks `inbox/` for higher-priority instructions, checks `STOP` |
| **Every N minutes** | (Phase 2+) Optional periodic polling via `/loop` |

### Manager Side

The ChatGPT manager processes bridge messages at interaction boundaries:

| Trigger | What Happens |
|---------|-------------|
| **Before each response** | Checks `outbox/cli-to-manager/` for new status reports, checks `decisions/pending/` |
| **After writing instruction** | Confirms file landed in `inbox/manager-to-cli/` |
| **User provides new input** | Processes input in context of latest bridge state |

### Phase 1 Polling

Phase 1 polling is manual (no bridge automation yet):
1. CLI writes status report to `outbox/cli-to-manager/20260503T120000Z~bacp~discovery~p12345~a8f3.status.json`
2. CLI prints: "BACP STATUS REPORT ready at ~/.bragi/agent-control-plane/outbox/cli-to-manager/"
3. Human opens file, copies content to ChatGPT
4. Manager responds, human writes response to `inbox/manager-to-cli/`
5. CLI detects new file on next check

---

## 11a. Concurrency Risk and Filename Uniqueness

### The Problem

Multiple concurrent CLI sessions (e.g., one executor per project) may write to the bridge simultaneously. Without safeguards, two sessions could write files with identical names, causing overwrites, lost messages, and corrupted audit state.

### Filename Uniqueness Requirement

All message filenames must be globally unique. This is enforced by the naming convention:

```
<TIMESTAMP>~<PROJECT>~<ROLE>~<PID>~<NONCE>.<TYPE>.json
```

| Component | Guarantee |
|-----------|-----------|
| `TIMESTAMP` | UTC ISO 8601 to the second. Prevents collisions across time. |
| `PROJECT` | Scoped to one project (`appkit`, `portal`, `msdk`, `router`, `bacp`). |
| `ROLE` | Identifies agent type (`executor`, `discovery`, `manager`). |
| `PID` | Process ID when available. Different OS processes produce different PIDs. |
| `NONCE` | 4 random hex characters (16-bit space, 65536 values). Collision risk at high concurrency is negligible but non-zero. |

### Future Locking Options (Design, Not Implemented)

To eliminate the remaining collision risk and prevent race conditions, these options are documented for future phases:

| Option | How It Works | When to Use |
|--------|-------------|-------------|
| **Atomic write via temp file** | Write to a `.tmp` file, then `rename()` (atomic on same filesystem). A reader sees only complete files. | Phase 2+ |
| **Per-project outbox directories** | Each project gets its own outbox: `outbox/cli-to-manager/appkit/`, `outbox/cli-to-manager/portal/`, etc. Reduces write contention. | Phase 2+ |
| **Message acknowledgement files** | An `.ack` sidecar file created after the original is processed. Writers check for `.ack` before proceeding. Prevents double-processing. | Phase 2+ |
| **Single-writer scheduler** | A single scheduler process serializes all bridge writes. Agents submit message content to the scheduler, which writes sequentially. Eliminates all write contention. | Phase 3+ (dashboard) |

### Phase 1 Approach

Phase 1 accepts the residual risk. The human-mediated transport and unique filenames make collisions astronomically unlikely at current concurrency levels (one active CLI agent at a time). If collisions occur in practice, a reader can distinguish files by full UUID and the human can resolve manually.

---

## 12. Archive Behavior

### When Archival Happens

A message is archived when the counterpart has acknowledged it:

| Scenario | What Gets Archived |
|----------|-------------------|
| Manager sends instruction → CLI starts work | Instruction moves to `archive/received/`. CLI writes new status to `outbox/`. |
| CLI sends status → Manager responds | Status moves to `archive/sent/`. Response lands in `inbox/`. |
| Decision is answered | Decision moves from `decisions/pending/` to `decisions/answered/`. |

### Archive Rules

- **Never modify archived messages.** Archive is append-only, write-once.
- **Archive filename includes original message ID** for traceability.
- **Archive directory preserves the original directory structure** (sent vs received, date-sorted).
- **Old archives are safe to delete** after 90 days (no system depends on them).
- **The bridge.log is never archived** — it stays in place and rotates by size.

### Archive Naming Convention

Messages are archived with their original filename so they remain traceable:

```
archive/sent/20260503T120000Z~appkit~executor~p12345~a8f3.status.json
archive/received/20260503T120030Z~appkit~manager~p00000~b4d1.instruction.json
```

---

## 13. Kill Switch

### How It Works

The `STOP` file at `~/.bragi/agent-control-plane/STOP` is a single plain-text file.

- **If `STOP` exists** → The bridge is dead. No new messages are created. No existing messages are processed.
- **If `STOP` does not exist** → Normal operation.

The human writes the STOP file. No agent creates or removes the STOP file.

### Contents of STOP

```
Bridge halted by human at 2026-05-03T14:00:00+02:00
Reason: Emergency pause — all BACP operations suspended
Resume: Delete this file to restore bridge operation
```

### What Happens When STOP Is Present

| Component | Behavior |
|-----------|----------|
| **CLI executor** | On next poll: detects STOP, logs "BRIDGE HALTED", refuses to read/write any bridge messages, exits any active bridge loop |
| **ChatGPT manager** | On next interaction: checks for STOP via relay, reports "Bridge is halted", refuses new bridge operations |
| **Relay (Phase 2+)** | Detects STOP, exits immediately, does not poll or push |

### Recovery

1. Human deletes `STOP`
2. Human appends a resume entry to `logs/bridge.log`
3. Next agent poll detects STOP is gone → resumes normal operation

### Safety Properties

- **Only the human can kill the bridge.** Agents cannot create or remove STOP.
- **STOP is checked at every poll cycle.** Maximum latency between writing STOP and bridge halting is one poll interval.
- **Existing in-flight messages are preserved.** When the bridge resumes, agents process pending messages normally.
- **STOP is unambiguous.** If the file exists, the bridge is dead. No edge cases.

---

## 14. Audit Log

### Format

`logs/bridge.log` is a plain-text, append-only file. One entry per bridge event.

```
2026-05-03T12:00:00+02:00 | OUT | cli-to-manager | 2026-05-03T120000-appkit-001 | status_report  | PENDING
2026-05-03T12:00:30+02:00 | IN  | manager-to-cli | 2026-05-03T120030-appkit-001-response | instruction | PENDING
2026-05-03T12:01:00+02:00 | ARC | sent           | 2026-05-03T120000-appkit-001 | status_report  | ARCHIVED
2026-05-03T12:01:00+02:00 | ARC | received       | 2026-05-03T120030-appkit-001-response | instruction | ARCHIVED
2026-05-03T13:00:00+02:00 | DEC | pending        | 2026-05-03T130000-portal-002 | decision_request | OPEN
2026-05-03T13:30:00+02:00 | DEC | answered       | 2026-05-03T130000-portal-002 | decision_request | RESOLVED
2026-05-03T14:00:00+02:00 | KILL | STOP          | -                          | kill_switch    | ENGAGED
```

### Log Event Types

| Prefix | Meaning |
|--------|---------|
| `OUT` | Message written to outbox |
| `IN` | Message written to inbox |
| `ARC` | Message archived |
| `DEC` | Decision request or response |
| `KILL` | Kill switch event |
| `ERR` | Bridge error |

### Log Rotation

- Log file grows unbounded. Rotate at 10MB.
- Old logs are safe to delete (archive contains the actual message content).

---

## 15. Project-Aware Routing

### Agent Identities

| Agent | Bridge Prefix | Can Read | Can Write |
|-------|-------------|----------|-----------|
| ChatGPT Manager | `manager` | `outbox/cli-to-manager/`, `decisions/pending/` | `inbox/manager-to-cli/`, `decisions/answered/` |
| CLI Executor (any) | `executor` | `inbox/manager-to-cli/`, `inbox/human-to-cli/`, `decisions/answered/` | `outbox/cli-to-manager/`, `decisions/pending/` |
| Human | `human` | `outbox/cli-to-human/`, `decisions/pending/` | `inbox/human-to-cli/`, `decisions/answered/` |

### Routing by Project

Each message's `project` field determines routing:

- `project: "all"` → Affects all projects. Processed by any available executor.
- `project: "portal"` → Only the executor working on Portal should respond.
- `project: "appkit"` → Only the AppKit executor.
- `project: "msdk"` → Only the mSDK executor.
- `project: "router"` → Only the Router executor.

### Routing by Message Type

| `type` | Routed To | Priority |
|--------|-----------|----------|
| `kill` | All agents | Critical |
| `override` | Target project executor | High |
| `instruction` | Target project executor | Normal |
| `decision` | Decision requester | Normal |
| `acknowledgment` | Message sender | Low |

### Conflict Resolution

If two messages in `inbox/` target the same executor:

1. `kill` and `override` types are processed first regardless of order
2. Remaining messages processed in chronological order by `created_at`
3. If `requires_response: false`, the message is informational and does not block the queue

---

## 16. Human Decision Handling

### When Human Decisions Are Needed

The bridge supports two decision request paths:

**Path A: CLI → Manager → Human**
```
CLI writes decision_request → outbox/cli-to-manager/
Manager sees it → decides whether to answer or escalate to human
If escalated: Manager formulates question for human
Human decides → Manager writes `decision` → inbox/manager-to-cli/
CLI reads decision → continues work
```

**Path B: CLI → Human (direct)**
```
CLI writes decision_request → outbox/cli-to-human/
Human sees file → writes decision → inbox/human-to-cli/
CLI reads decision → continues work
```

### Decision Request Lifecycle

1. CLI writes to `decisions/pending/` with `type: decision_request`
2. File sits in `pending/` until answered
3. Decision is written to `decisions/answered/` with the same `id` + `resolved_at` timestamp
4. CLI polls `decisions/answered/` at each cycle
5. Original `pending/` file is moved to `archive/`

### Human Decision Response

```json
{
  "id": "2026-05-03T130000-portal-002-response",
  "reply_to": "2026-05-03T130000-portal-002",
  "project": "portal",
  "direction": "manager_to_cli",
  "type": "decision",
  "from": "human",
  "to": "executor",
  "created_at": "2026-05-03T13:30:00+02:00",
  "payload": {
    "decision": "Option 2",
    "rationale": "Speed matters more than completeness for discovery",
    "additional_guidance": "Focus on CI workflow topology. Skip individual test file analysis."
  }
}
```

---

## 17. How This Later Connects to the Dashboard

The bridge provides the data foundation for a future dashboard:

### Phase 3 Ready Data

```
~/.bragi/agent-control-plane/
├── archive/          # Complete message history → dashboard timeline view
├── decisions/        # Pending and resolved decisions → decision tracker
├── logs/bridge.log   # Event stream → real-time activity feed
└── inbox/outbox/     # Current state → active task panel
```

### Dashboard Concepts That Use Bridge Data

| Dashboard Feature | Bridge Data Source |
|------------------|-------------------|
| Project status cards | Latest `status_report` per project in `archive/sent/` |
| Active tasks | Files in `inbox/manager-to-cli/` not yet archived |
| Pending decisions | Files in `decisions/pending/` |
| Activity timeline | `logs/bridge.log` parsed and rendered |
| Kill switch status | Presence/absence of `STOP` file |
| Decision history | `decisions/answered/` |
| Message volume per day | Archive file count by date |

### Dashboard Does NOT Replace the Bridge

The dashboard is a read-only visualization of bridge data. It never writes to `inbox/`, `outbox/`, or `decisions/`. The ChatGPT manager and CLI executor remain the only writers.

---

## 18. How This Later Connects to GitHub

The bridge connects to GitHub in three ways, all optional and phased:

### GitHub Integration Paths

| Integration | Phase | Purpose | CLI Side |
|-------------|-------|---------|----------|
| **Status PR linking** | Phase 1 (manual) | `payload.git_status` includes PR URLs | CLI reports PR links in status. Human copies them. |
| **Bridge repo relay** | Phase 2 (assisted) | ChatGPT reads/writes bridge via GitHub API | Relay script syncs bridge dir ↔ GitHub repo. CLI unchanged. |
| **GitHub Actions trigger** | Phase 2 (assisted) | Manager dispatches → triggers workflow | CLI reads instruction, may trigger a workflow via `gh workflow run`. |

### The Bridge Repo Pattern (Phase 2)

A dedicated `bragi-bridge/` GitHub repo acts as the relay:

```
ChatGPT (GitHub API) → bragi-bridge/ repo → relay script → ~/.bragi/agent-control-plane/
```

The relay:
- Is a simple shell script (bash or Python)
- Runs via cron or launchd every 30 seconds
- Syncs bidirectionally: bridge dir ↔ bridge repo
- Never modifies anything outside the bridge directory
- Checks for `STOP` first — if present, refuses to sync

This is the *only* external connection from either the bridge or the CLI. The CLI itself never calls any external API for bridge purposes.

---

## 19. Phased Implementation Plan

### Phase 1 — Manual Structured Bridge (ACTIVE)

**Status:** Directory structure created, bridge operational at filesystem level.
**Goal:** Eliminate unstructured copy/paste. Messages are structured JSON.

| Step | Status |
|------|--------|
| 1.1 | Create `~/.bragi/agent-control-plane/` with full directory structure |
| 1.2 | CLI writes structured JSON status reports to `outbox/cli-to-manager/` |
| 1.3 | Human copies JSON from file → pastes to ChatGPT |
| 1.4 | Manager responds with structured JSON |
| 1.5 | Human copies JSON → writes to `inbox/manager-to-cli/` |
| 1.6 | Messages archived to `archive/` after acknowledgement |
| 1.7 | All bridge activity logged to `logs/bridge.log` |
| 1.8 | `STOP` file created and understood by all agents |

**Phase 1 is entirely human-mediated.** The CLI writes files. The human moves data between CLI and ChatGPT. This is still an improvement over unstructured copy/paste because:
- All messages have a consistent schema
- Full audit trail exists
- Archive preserves history
- Decision requests are tracked
- Kill switch works from day one

**Phase 1 is the reporting path for all BACP activity.** Every discovery pass, every task — all report through this bridge. See `docs/discovery-plan.md` and `docs/phase-1-bridge-setup.md` for bridge usage.

### Phase 2 — Local Bridge Helper (OPTIONAL)

**Decision:** Phase 2 bridge relay is optional. It is NOT a prerequisite for Phase 3 (dashboard). Do not build until Phase 1 has proven insufficient or high enough volume to justify automation.

**Goal:** Reduce human paste friction. No more manual file copying.

| Step | When |
|------|------|
| 2.1 | Write a small bridge helper script (bash/Python) |
| 2.2 | Helper watches `outbox/cli-to-manager/` for new files |
| 2.3 | Helper writes manager responses into `inbox/manager-to-cli/` |
| 2.4 | Helper archives acknowledged messages |
| 2.5 | Helper checks for `STOP` at every cycle |
| 2.6 | Bridge repo relay for ChatGPT direct access (only if human approval is no longer needed for transport) |

**Phase 2 still requires the human to approve outbound messages.** The helper does not send anything externally. It watches local files and moves them around. The human is the gatekeeper for any ChatGPT interaction.

### Phase 3 — Dashboard Bridge

**Decision:** Phase 3 does NOT depend on the Phase 2 bridge relay. The dashboard reads directly from the local filesystem at `~/.bragi/agent-control-plane/`. It works immediately after Phase 1 creates the directory structure.

**Goal:** A visual dashboard that consumes bridge data.

| Step | What |
|------|------|
| 3.1 | Read-only dashboard reads bridge directory |
| 3.2 | Shows: project status, active tasks, pending decisions, activity timeline |
| 3.3 | Dashboard is local-only, no server |
| 3.4 | Dashboard never writes to bridge directories |
| 3.5 | Decision inbox visible in dashboard |
| 3.6 | Kill switch status visible in dashboard |

**Phase 3 is read-only.** The dashboard visualizes bridge state. All writing still happens through the CLI and ChatGPT.

### Phase 4 — API-Connected Manager (Optional)

**Goal:** ChatGPT directly reads/writes the bridge without human copy/paste.

| Step | What |
|------|------|
| 4.1 | ChatGPT reads `outbox/cli-to-manager/` via bridge repo relay |
| 4.2 | ChatGPT writes instructions to `inbox/manager-to-cli/` via bridge repo relay |
| 4.3 | Human reviews and approves outbound ChatGPT messages before relay |
| 4.4 | Human maintains ability to halt bridge at any time via `STOP` |

**Phase 4 is optional.** Only implement if Phase 2 + Phase 3 prove insufficient. Phase 4 requires explicit approval because it introduces external API calls into the bridge flow.

**Phase 4 is never fully autonomous.** The dashboard remains the human control layer. The human can:
- Review all pending outbound messages before relay
- Pause the relay at any time
- Kill the bridge at any time
- Override any automated decision

---

## Appendix: Bridge vs. Vault Messages

| Aspect | Bridge (`~/.bragi/`) | Vault Messages (`bragi-vault/messages/`) |
|--------|---------------------|------------------------------------------|
| **Purpose** | Manager↔CLI operational communication | Cross-repo developer communication |
| **Format** | JSON | Markdown with YAML frontmatter |
| **Actors** | manager, executor, human | appkit, portal, msdk, all |
| **When** | During orchestrated execution | Async, ongoing |
| **Persistence** | Archived for 90 days | Permanent (git-tracked knowledge) |
| **Schema** | Defined in this document | Template at `templates/message.md` |
| **Automation** | Bridge helper watches directories | Watcher agents poll via flags |
| **Routing** | `project` field + agent identity | `to` field + `.notify-*` flags |
| **Kill switch** | `STOP` file | None (permanent system) |

The two systems are complementary. Bridge messages about work-in-progress. Vault messages about decisions, knowledge, and cross-repo contracts. A status report in the bridge may result in a vault message being created to record a decision.
