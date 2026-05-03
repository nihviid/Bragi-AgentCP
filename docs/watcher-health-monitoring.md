# Watcher Health Monitoring

**Date:** 2026-05-03
**Status:** Design (pre-implementation)
**Phase:** Pre-design (discovery gap analysis)

---

## 1. Problem Statement

Discovery Pass 5 found a structural gap in the existing cross-project collaboration protocol: there is no watcher health monitoring.

The current system assumes watcher agents are always running, always healthy, and always processing messages within seconds (the `/loop 60s` interval). When a watcher goes offline — whether due to session expiry, network issues, model errors, or human absence — the system produces no alert. Messages pile up in `messages/pending/` silently.

As of 2026-05-03, 4 pending messages from 2026-04-29/30 (PORTAL_TO_APPKIT) remain unprocessed. This is a **5-day backlog** in what is designed as a sub-minute loop. The sender (Portal) has no way to know the AppKit watcher hasn't seen the messages. The only detection mechanism is a human manually noticing the stale messages.

**The risk to BACP:** If BACP relies on the vault message protocol for cross-project awareness, undetected watcher failures mean BACP operates on stale data. The manager must know when a project's watcher is offline.

---

## 2. Current Watcher Model

### Agent Roster

| Watcher | Project | Focus | `/loop` Interval |
|---------|---------|-------|-----------------|
| `appkit` | AppKit | Runtime, Kits, Bragi API runtime | 60s |
| `portal` | Portal | Cloud control plane, Bragi Builder | 60s |
| `msdk` | mSDK | BLE transport, iOS native plugin | 60s |
| `cto` | All (advisory) | Architecture, technical debt | 60s |
| `ceo` | All (advisory) | Business, strategy | 60s |
| `cpo` | All (advisory) | Product, roadmap | 60s |
| `cro` | All (advisory) | Revenue, partnerships | 60s |
| `consultant` | All (advisory) | Consulting, domain expertise | 60s |

### Loop Behavior

Every watcher runs the same loop via `/loop 60s`:

```
1. Check if flag file exists: messages/.notify-<role>
2. If flag does NOT exist → report "No flag — idle" → yield
3. If flag EXISTS:
   a. Delete the flag file
   b. Scan messages/pending/ for matching messages
   c. For each: read → research → write response → set status → move to completed/
4. Yield → sleep 60s → repeat
```

### Key Characteristics

- **Flag-driven:** No flag = no work = near-zero cost. The watcher is idle between messages.
- **Single-threaded:** One watcher per project. No parallel processing.
- **Stateless per tick:** Each `/loop` iteration is independent. No accumulated state across ticks.
- **Silent on absence:** If the session expires or the `/loop` stops, no alert is raised. The flag stays intact. The messages remain in `pending/`.

---

## 3. Observed Risk

| Risk | Observed Evidence | Impact |
|------|------------------|--------|
| **Stale pending messages** | 4 PORTAL_TO_APPKIT messages from 2026-04-29/30 still in pending/ as of 2026-05-03 (5 days) | Portal sent responses/collaboration requests that were never received by AppKit |
| **Silent watcher failure** | AppKit watcher did not process messages for 5 days. No alert was raised by any system. | Backlog grew undetected. Human had to notice manually. |
| **No sender acknowledgement** | Portal has no way to know AppKit never read the messages | Portal operates assuming collaboration happened |
| **No timeout or retry** | Once a message is in pending/, it stays there forever unless a watcher processes it | No circuit breaker. No escalation path. |
| **Backlog compounds** | If AppKit watcher comes back online, it processes all 4 pending messages at once (burst) | Potential for missed messages in the burst, or ordering issues if messages are interdependent |

### Root Cause

The watcher model has no health feedback loop. It is a fire-and-forget architecture:
- Watcher runs → processes messages → reports "idle" → repeats
- Watcher stops → no one notices → messages accumulate

The only "health signal" is the absence of processing activity, which is invisible without monitoring.

---

## 4. BACP Monitoring Goals

1. **Detect stale pending messages** — messages in `pending/` longer than configurable thresholds
2. **Detect silent watcher failure** — project watchers with no recent processing activity
3. **Alert the manager** — create decision requests when health degrades
4. **Provide watcher health visibility** — through the bridge (`bacp-bridge watcher-health`)
5. **Integrate with manager control loop** — health state feeds into manager decisions
6. **Preserve existing vault protocol** — monitoring is read-only, completely non-interfering

---

## 5. BACP Non-Goals

- **Not a watcher replacement.** BACP does not process vault messages. Watchers stay unchanged.
- **Not a message processor.** BACP does not read, respond to, or forward vault messages.
- **Not a flag manager.** BACP does not create, delete, or modify flag files.
- **Not a message archiver.** BACP does not move messages between pending/completed.
- **Not a notification system.** BACP does not alert watchers or send cross-project messages.
- **Not a watcher supervisor.** BACP does not restart, re-deploy, or manage watcher sessions.
- **Not a real-time monitor.** BACP monitors on-demand (via `bacp-bridge watcher-health`) or at manager instruction boundaries.

---

## 6. Read-Only Monitoring Model

```
BACP Monitor (read-only)
    │
    ├── reads messages/pending/          ← file count, age per file
    ├── reads messages/completed/        ← freshness per project
    ├── reads messages/.notify-*         ← flag presence and age
    ├── does NOT modify any of the above
    │
    ▼
Health Assessment (derived from file metadata)
    │
    ├── per-watcher health state
    ├── pending message age by project
    ├── flag staleness
    │
    ▼
Decision Request (if threshold exceeded)
    │
    └──→ decisions/pending/ → manager reviews → decides action
```

### What BACP Reads

| Data | Source | How |
|------|--------|-----|
| Pending message count | `messages/pending/` | List `.md` files |
| Pending message age | File modification time | Compare to current time |
| Pending message sender/receiver | YAML frontmatter `from`/`to` | Parse frontmatter |
| Completed message freshness | `messages/completed/` files by modification time | Most recent per project |
| Flag presence | `messages/.notify-*` files | Check existence |
| Flag age | File modification time | Compare to current time |

### What BACP Does NOT Touch

- File contents (beyond frontmatter parsing for `from`/`to` fields)
- Flag files (never creates, deletes, or modifies)
- Message states (never changes `status` field)
- File locations (never moves between directories)

---

## 7. Health Signals to Track

### Signal 1: Pending Message Age

The most direct signal. A pending message means a watcher hasn't processed it.

| Metric | Source | Unit |
|--------|--------|------|
| Oldest pending message age | `messages/pending/` mtime | Minutes since modification |
| Per-project pending age | Filter pending files by `to` field frontmatter | Minutes per project |
| Pending message count | `messages/pending/` | Count |
| Per-project pending count | Filter by `to` field | Count per project |

### Signal 2: Pending Message Count

A growing backlog indicates the watcher isn't keeping up. For a healthy system with `/loop 60s`, the pending count should be 0 or very low.

### Signal 3: Flag Age

If a flag file exists and is old, the watcher hasn't polled recently. This is a stronger signal than pending message age because flags are deleted on first poll — even if no messages exist.

| Metric | Source | Unit |
|--------|--------|------|
| Flag presence | `messages/.notify-*` exist? | Boolean |
| Flag age | `messages/.notify-*` mtime | Minutes since creation |
| Any flag active? | Any `.notify-*` file exists? | Boolean |

### Signal 4: Completed Message Freshness

Even without pending messages, a healthy watcher should produce periodic activity. If no completed messages exist for a project in N days, the watcher may be offline.

| Metric | Source | Unit |
|--------|--------|------|
| Latest completed per project | `messages/completed/` mtime filtered by `from` field | Timestamp |
| Days since last completed | Current time − latest completed mtime | Days |

### Signal 5: Per-Project Watcher Activity

Aggregate view of all signals for a single project.

### Signal 6: Stale Relates-To Chains

If a pending message has a `relates-to` field pointing to a message that has been in `completed/` for a long time, it may be a follow-up that was never processed. This indicates a broken thread.

### Signal 7: Missing Response Patterns

If a project consistently sends messages but never receives responses (no completed messages from the target project), a watcher may be broken. This requires tracking sender/receiver pairs over time.

---

## 8. Watcher Health States

### State Definitions

| State | Definition | CLI Indicator | Manager Action |
|-------|-----------|---------------|----------------|
| **Healthy** | All pending messages < 2 hours old. Flags processed within 60s. Recent completed activity. | `✓ HEALTHY` | None — continue normal operation |
| **Delayed** | Pending messages 2–24 hours old. No critical threshold exceeded. | `⚠ DELAYED` | Observe. Flag if trend continues. |
| **Stale** | Pending messages 24–72 hours old. Or flag exists > 24 hours. Or no completed activity in 7 days. | `✗ STALE` | Create decision request. Manager decides whether to alert human. |
| **Blocked** | Pending messages > 72 hours old. Or flag exists > 72 hours. Or multilple stale pending messages per project. | `✗ BLOCKED` | Create decision request. Escalate to human. |
| **Unknown** | No data available for this watcher. Watcher identity not recognized. | `? UNKNOWN` | Log as unknown. Flag for manager review. |

### State Transition Diagram

```
UNKNOWN
   │
   ▼
HEALTHY ←── DELAYED ──→ STALE ──→ BLOCKED
   │          (2h)       (24h)      (72h)
   │
   └──→ Auto-recovery when pending messages cleared
```

Watchers self-recover when they come back online and process their backlog. BACP observes and reports. BACP does not perform recovery actions.

---

## 9. Decision Request Triggers

When health monitoring crosses a threshold, BACP creates a decision request in `decisions/pending/`. The manager reviews and decides.

### Trigger: Pending Message Older Than Threshold

```
Condition: Any pending message in messages/pending/ with:
  age > threshold AND
  to field matches a known watcher role

Decision request to: manager
Title: "Watcher {role} has {count} pending message(s), oldest is {age} hours"
Options:
  - Monitor: No action, continue observing (current state)
  - Alert human: Manager notifies human that watcher may be offline
  Ignore: Dismiss this alert (adds watcher to ignore list)
```

### Trigger: Watcher Appears Inactive

```
Condition: For a known watcher role, both:
  no completed messages from this watcher in last 7 days AND
  pending messages exist targeting this watcher

Decision request to: manager
Title: "Watcher {role} appears inactive — no activity in {days} days"
```

### Trigger: Repeated Messages Without Response

```
Condition: Same sender sends 3+ messages to the same target with no response
  AND at least one pending message from this sender->target pair is > 24 hours

Decision request to: manager
Title: "Repeated messages from {sender} to {target} without response ({count} messages)"
```

### Trigger: Orphaned Flag

```
Condition: A flag file exists for > 24 hours
  AND no messages exist for that flag's target
  (flag was created but message was lost)

Decision request to: manager
Title: "Orphaned flag .notify-{role} — no matching messages found"
```

### Trigger: Project-Specific Backlog

```
Condition: A single project has 5+ pending messages

Decision request to: manager
Title: "Backlog detected for {project}: {count} pending messages"
```

---

## 10. Dashboard Implications

### Watcher Health Panel

A future dashboard would display:

```
WATCHER HEALTH
┌──────────┬──────────┬────────┬────────┬──────────┐
│ Watcher  │ Pending  │ Oldest │ Last   │ State    │
│          │ Messages │ Age    │ Active │          │
├──────────┼──────────┼────────┼────────┼──────────┤
│ appkit   │ 4        │ 5d     │ 5d ago │ ✗ STALE  │
│ portal   │ 0        │ —      │ 1d ago │ ✓ HEALTHY│
│ msdk     │ 0        │ —      │ 2d ago │ ✓ HEALTHY│
│ cto      │ 0        │ —      │ 3d ago │ ✓ HEALTHY│
│ ceo      │ 0        │ —      │ 7d ago │ ⚠ DELAYED│
└──────────┴──────────┴────────┴────────┴──────────┘
```

### Health State Color Coding

| State | Color | Dashboard Display |
|-------|-------|-------------------|
| Healthy | Green | Normal operation |
| Delayed | Yellow | May need attention |
| Stale | Orange | Needs review |
| Blocked | Red | Requires human intervention |
| Unknown | Gray | No data available |

### What the Dashboard Does NOT Show

The dashboard does not show individual message content. It shows aggregate health metrics. Individual messages remain in the vault for those who need to read them.

---

## 11. Manager-Control Implications

### How Health State Affects Manager Decisions

| Health State | Manager Trust in Project Data | Manager Action |
|--------------|------------------------------|----------------|
| **Healthy** | Full trust | Issue instructions normally |
| **Delayed** | Reduced trust — data may be slightly stale | Add "verify" step to next instruction |
| **Stale** | Low trust — project data is likely stale | Issue `request_status` to project CLI executor. Do not rely on watcher-processed data. |
| **Blocked** | No trust — watcher is likely offline | Escalate to human. Do not proceed with cross-project decisions. |
| **Unknown** | No data | Investigate before issuing cross-project instructions |

### Manager Control Loop Integration

Health state feeds into the manager control loop at the VERIFY step:

```
ISSUE → ACKNOWLEDGE → ACT → REPORT → VERIFY → DECIDE
                                          │
                                    Check watcher health
                                      │
                                    If STALE or BLOCKED:
                                      Flag in report
                                      Suggest human escalation
                                      Recommend pause on cross-project work
```

### Availability Instruction

The manager can issue a new command to request watcher health data:

```json
{
  "command": "request_status",
  "instruction": "Check watcher health for all projects and report.",
  ...
}
```

The executor runs `bacp-bridge watcher-health` (when implemented) and includes the results in the status report.

---

## 12. Non-Interference Rules

| Rule | Rationale | Enforcement |
|------|-----------|-------------|
| **Never delete flag files** | Flags are watcher control signals. Only the target watcher deletes its flag. | Design constraint + code review |
| **Never move messages between directories** | Only the processing watcher moves from pending/ to completed/. | Design constraint |
| **Never change message status** | The `status` field tracks lifecycle. Only the processing watcher changes it. | Design constraint |
| **Never create flag files** | BACP does not signal to watchers. BACP observes only. | Design constraint |
| **Never respond to vault messages** | BACP is not a vault participant. It uses its own bridge. | Architecture boundary |
| **Never modify message content** | Reading frontmatter for `from`/`to` is permitted. Modifying is not. | Read-only policy |
| **Never notify watchers** | BACP does not alert, restart, or interact with watchers. | Manager authority boundary |
| **Never auto-archive or auto-clean** | Stale messages remain. BACP reports but does not clean. | Data integrity |

### Enforcement

The monitoring implementation will:
- Open files in read-only mode
- Parse only frontmatter fields (`from`, `to`, `status`, `date`)
- Never open files for writing
- Never call rename/move/delete on vault paths
- Never write to `messages/` or `decisions/` directly (BACP uses own bridge paths)

---

## 13. Proposed Future Command

### `bacp-bridge watcher-health`

**Purpose:** Check health of all vault watcher agents. Read-only. No interference.

**Output:**

```
$ bacp-bridge watcher-health
Watcher Health Report — 2026-05-03T12:00:00Z
Vault path: ~/Desktop/Antigravity/bragi-vault

WATCHER         PENDING   OLDEST   FLAG   LAST COMPLETED    STATE
appkit          4         5d       no     5d ago            ✗ STALE
portal          0         —        no     1d ago            ✓ HEALTHY
msdk            0         —        no     2d ago            ✓ HEALTHY
cto             0         —        no     3d ago            ✓ HEALTHY
ceo             0         —        no     7d ago            ⚠ DELAYED
cpo             0         —        no     —                 ? UNKNOWN
cro             0         —        no     —                 ? UNKNOWN
consultant      0         —        no     —                 ? UNKNOWN

Pending messages by project:
  appkit (target): 4
    PORTAL_TO_APPKIT_2026-04-29_e2e-driver-registry-contract-realignment (5d)
    PORTAL_TO_APPKIT_2026-04-29_af09-followups-merged (5d)
    PORTAL_TO_APPKIT_2026-04-30_phase41-device-descriptor-confirmed (4d)
    PORTAL_TO_APPKIT_2026-04-29_af09-followups-merged (5d)

Active flags:
  .notify-all — created 2026-04-30, age 3d
  .notify-appkit — created 2026-04-30, age 3d
```

**Implementation sequencing:** This command is designed for Phase 2+ after the current P0+P1 commands are stable. It requires:
- Access to `bragi-vault/messages/` (read-only)
- Frontmatter parsing for `from`/`to` fields
- File mtime comparison
- Threshold configuration

### Command Dependencies

| Dependency | Source | Phase |
|-----------|--------|-------|
| Vault path | Hardcoded or env var (`BRAGI_VAULT_PATH`) | Design |
| Thresholds | Configurable defaults (see section 15) | Design |
| Frontmatter parsing | Python standard library (no YAML parser — simple line-based) | Implementation |
| File timestamps | `os.path.getmtime()` | Implementation |

---

## 14. Proposed Health Report Schema

```json
{
  "generated_at": "2026-05-03T12:00:00Z",
  "vault_path": "~/Desktop/Antigravity/bragi-vault",
  "overall_status": "degraded",
  "watchers": [
    {
      "role": "appkit",
      "project": "AppKit",
      "state": "stale",
      "pending_count": 4,
      "oldest_pending_age_hours": 120,
      "flag_exists": false,
      "flag_age_hours": null,
      "last_completed": "2026-05-01T12:00:00Z",
      "last_completed_age_hours": 48,
      "thresholds_exceeded": ["pending_age", "last_completed_age"]
    },
    {
      "role": "portal",
      "project": "Portal",
      "state": "healthy",
      "pending_count": 0,
      "oldest_pending_age_hours": null,
      "flag_exists": false,
      "flag_age_hours": null,
      "last_completed": "2026-05-02T12:00:00Z",
      "last_completed_age_hours": 24,
      "thresholds_exceeded": []
    }
  ],
  "pending_messages": [
    {
      "filename": "PORTAL_TO_APPKIT_2026-04-29_e2e-driver-registry-contract-realignment.md",
      "from": "portal",
      "to": "appkit",
      "age_hours": 120,
      "created": "2026-04-29",
      "subject": "Portal e2e suite — driver-registry contract test realigned"
    }
  ],
  "flags": [
    {
      "filename": ".notify-appkit",
      "target": "appkit",
      "age_hours": 72,
      "created": "2026-04-30"
    }
  ],
  "thresholds": {
    "delayed_hours": 2,
    "stale_hours": 24,
    "blocked_hours": 72,
    "unknown_days": 7
  }
}
```

---

## 15. Threshold Defaults

| Threshold | Default | Meaning | State |
|-----------|---------|---------|-------|
| `delayed_hours` | 2 | Pending message older than 2 hours | DELAYED |
| `stale_hours` | 24 | Pending message older than 24 hours | STALE |
| `blocked_hours` | 72 | Pending message older than 72 hours | BLOCKED |
| `unknown_days` | 7 | Watcher has no completed messages in 7 days | UNKNOWN |
| `backlog_count` | 5 | Project has 5+ pending messages | STALE |
| `flag_delayed_hours` | 1 | Flag exists for > 1 hour | DELAYED |
| `flag_stale_hours` | 12 | Flag exists for > 12 hours | STALE |

### Design Notes on Thresholds

- Thresholds should be configurable, not hardcoded. The user or manager may adjust based on observed cycle times.
- The ``delayed` threshold (2 hours) accounts for normal overnight gaps. A 2-hour pending message during business hours is different from a 2-hour pending message overnight. Phase 2 monitoring may add time-of-day awareness.
- The `blocked` threshold (72 hours) represents a weekend. If a message is still pending after a weekend, the watcher is likely offline, not just slow.
- These defaults are starting points. Real thresholds should be tuned based on observed watcher cycle times.

---

## 16. Integration with BACP Decisions

### Where Health Data Feeds In

| BACP Component | How It Consumes Health Data |
|---------------|---------------------------|
| **Manager control loop** | At VERIFY step, checks health state before deciding next action |
| **Decision requests** | Stale/Blocked watchers trigger decision requests to manager |
| **Executor reports** | `bacp-bridge watcher-health` output included in status reports |
| **Dashboard (future)** | Watcher health panel displays aggregate states |
| **Audit log** | Health state changes logged to `logs/bridge.log` |

### What Happens When BACP Detects an Unhealthy Watcher

The BACP manager may:

1. **Request status from the project CLI executor** — to determine if the project is actively being worked on
2. **Create a decision request** — asking whether to alert the human about the offline watcher
3. **Pause cross-project collaboration** — until watcher health is restored
4. **Escalate to human** — if the watcher has been blocked for > 72 hours

### What BACP Never Does

- Does not restart or re-deploy the watcher
- Does not send messages on behalf of the watcher
- Does not process the watcher's pending messages
- Does not delete or archive pending messages

---

## 17. Open Questions

| Question | Impact | Needs Answer From |
|----------|--------|-------------------|
| Should thresholds be configurable per project or global? | A busy project may have longer acceptable delays than an idle one. | Manager |
| Should BACP track watcher health history or only current state? | History enables trend detection (e.g., "watcher health degrading"). | Manager |
| Should the health report include the vault path as a parameter or always use the known default? | Multiple vault locations would require parameterization. | Manager |
| Should `bacp-bridge watcher-health` be a separate command or a flag on `bacp-bridge status`? | Separate command keeps `status` focused on bridge health. Combined reduces commands. | Manager |
| Should health alerts (STALE/BLOCKED) auto-create decision requests, or only report in dashboard? | Auto-creation improves responsiveness. But may create noise if thresholds are too tight. | Manager |
| Should health monitoring be aware of business hours? | Pending messages overnight are different from pending messages during work hours. | Human |
| Should there be a way to acknowledge known-backlog scenarios? | If human knows watcher is offline, they should be able to silence alerts for that watcher. | Human |
