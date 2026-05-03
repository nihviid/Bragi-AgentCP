# Connector Requirements — From Real Usage

**Bragi Agent Control Plane (BACP)**  
**Date:** 2026-05-03  
**Branch:** `agent/bootstrap-discovery`  
**Status:** Requirements (pre-implementation)

---

## Context

Three real control-loop tasks were executed using the tool:

| Task | Description | Status |
|------|-------------|--------|
| 1 | Fix metadata fallback in next-instruction/next-report | Done (f28c6ec) |
| 2 | Improve README quickstart with clear 5-step loop | Done (73e1fa6) |
| 3 | Add example instruction/report pair | Done (73e1fa6) |

Each task followed the full tool loop: write-instruction → next-instruction → ack → write-report → next-report.

This document captures what felt smooth, what did not, and what a connector should automate.

---

## What Felt Smooth

- **The 5-step loop works.** INSTRUCT → READ → ACK → REPORT → REVIEW is unambiguous. Each step is one command. No directory navigation.
- **Filename-based metadata fallback.** Never seeing "?" anymore. Project, created_at, and id always display correctly.
- **`write-report` auto-fill.** The envelope fields (direction, type, created_at, from, to) are filled automatically. No risk of forgetting them.
- **`ack` by nonce.** Finding a message by partial ID (e.g., `ack c073`) works reliably for unique nonces.
- **`--ack-source` on write-report.** Archiving the source instruction at report time is the right default behavior.

---

## What Was Still Manual

### 1. `archive-all` is too blunt

Every `archive-all reports` archived *everything* including fresh reports written seconds earlier. Had to manually restore reports from archive twice during these 3 tasks.

**Fix needed:** Archive only messages older than a threshold, not all messages.

### 2. Report content must be manually constructed

`write-report` auto-fills the envelope, but the `project_state` JSON still needs manual construction via `echo`. For simple reports this is fine. For complex reports (with what_was_attempted, what_succeeded, what_failed lists) it requires careful quoting in the shell.

**Fix needed:** Either (a) accept a file path argument, or (b) provide a template generator.

### 3. No queue inspection

`status` shows counts but not filenames. `next-report` shows content. There is no way to list all pending reports or instructions without consuming them.

**Fix needed:** A `list-reports` and `list-instructions` command that shows filenames and metadata without displaying full content.

### 4. No auto-commit integration

Commit messages were handwritten. The task description from the manager instruction could be the commit message body, but there is no bridge between the tool and git.

**Fix needed:** Optional `--commit` flag on write-report that creates a git commit with the task description as message.

---

## What a Connector Should Automate

### Core (Phase 3)

1. **Poll inbox/** at a configurable interval and surface new instructions to the CLI without manual `next-instruction`
2. **Auto-archive source** after write-report when `--ack-source` is used (already works, but could be default behavior)
3. **Queue inspection** — list filenames and metadata in all queues without consuming messages
4. **Instruction context injection** — include the source instruction in the report template for tighter feedback

### Future (Phase 4)

5. **Direct API relay** — ChatGPT writes directly to inbox/ and reads directly from outbox/
6. **Auto-commit** — optional git commit from write-report
7. **Dashboard** — web UI that reads the same filesystem

---

## What a Connector Must Not Bypass

| Rule | Rationale |
|------|-----------|
| STOP must halt the connector | Kill switch is the last line of defense. Connector must check STOP before every action. |
| Manual intervention must always be possible | The tool must continue to work independently. Connector is an enhancement, not a replacement. |
| Archive must remain append-only | No deletion. No compaction. No TTL on archive entries. |
| Human must be able to see all connector activity | Every connector action must be visible in bridge.log. No silent operations. |
| No outbound network from the bridge | The connector is local-only. API relay is a separate concern (Phase 4). |
| No file deletion | Archive moves only. The connector must never delete files. |

---

## Minimum Viable Connector Responsibilities

```
1. POLL    Check inbox/ every N seconds for new messages
2. NOTIFY  Print new instruction to terminal when detected
3. REPORT  (no change — write-report already handles this)
4. ARCHIVE Auto-ack source when write-report --ack-source is used
5. INSPECT List messages in all queues (new bacp-bridge list-* commands)
6. STOP    Check STOP before every poll cycle. Halt if present.
```

The connector does NOT:
- Replace the tool (tool continues to work independently)
- Make API calls or network requests
- Create or delete files (archive moves only)
- Bypass human oversight
- Modify external projects

---

## Implementation Priority

| Priority | Feature | Why Now |
|----------|---------|---------|
| P0 | Queue inspection commands (`list-reports`, `list-instructions`) | Needed daily. Solves the "what's pending?" gap. |
| P1 | Poll daemon | Reduces friction. No more manual `next-instruction`. |
| P1 | Archive-by-age for archive-all | Prevents accidental archival of fresh messages. |
| P2 | Template generator for write-report | Reduces manual echo quoting. |
| P3 | Auto-commit on write-report | Nice to have. Bridges tool and git. |
| P4 | Direct API relay | Eliminates human clipboard entirely. |

---

## Verdict

The tool is ready for daily use. The 5-step loop works without friction for small tasks.

The connector should focus on queue inspection and polling first — eliminating the need to manually check for new instructions. Everything else is additive.
