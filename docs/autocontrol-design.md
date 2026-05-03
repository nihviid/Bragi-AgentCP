# Autocontrol Design

**Minimum Safe Autocontrol Layer for BACP**  
**Date:** 2026-05-03  
**Branch:** `agent/autocontrol-minimum`

---

## 1. Definition of Autocontrol

Autocontrol means automated transport between the AgentCP filesystem bridge, the manager interface, and the CLI executor, without the human acting as clipboard intermediary.

### Autocontrol Does Mean

- Reports are automatically surfaced to the manager
- Manager instructions are written back to the CLI inbox
- The CLI continues within explicit constraints
- The control loop is faster because the human is no longer the transport layer

### Autocontrol Does NOT Mean

- Autonomous merge or deployment
- Autonomous production changes
- Bypassing the STOP file
- Bypassing human approval for high-risk decisions
- Hidden execution or silent scope expansion
- The CLI deciding what to do next

---

## 2. Current Limitation

Today, the human is the transport layer:

```
Executor writes report → outbox/
Human opens file → copies text → pastes to ChatGPT
ChatGPT responds → human copies → pipes to write-instruction
```

The tool eliminated directory navigation and filename construction. But the clipboard handoff remains. Every round-trip requires:
1. Terminal → ChatGPT (copy)
2. ChatGPT → terminal (paste)

For a 10-task session, that is 20 manual clipboard operations. Autocontrol eliminates all of them while keeping the human as the decision authority.

---

## 3. Desired Loop

```
Executor writes report to outbox/
         │
    [autocontrol relay detects report]
         │
         ▼
    Manager receives report (via API or relay UI)
         │
    Manager produces instruction
         │
         ▼
    [relay writes instruction to inbox/]
         │
         ▼
    Executor reads instruction
         │
    Executor acts
         │
    Executor writes report
         │
         ▼
    [repeat]
```

The relay replaces the human clipboard. The human remains the decision authority for high-risk gates.

---

## 4. Minimum Viable Autocontrol

The P0 must support:

| Capability | Required |
|------------|----------|
| Watch outbox/ for new reports | Yes |
| Surface newest report to manager (stdout or relay) | Yes |
| Accept manager response (stdin or file) | Yes |
| Write response to inbox/ | Yes |
| Archive or ack source report | Yes, explicit |
| Obey STOP file | Yes |
| Write audit log | Yes |
| Respect human decision gates | Yes |

P0 does NOT include:
- API integration
- Dashboard UI
- Clipboard integration
- Background daemon (poll loop is fine)

---

## 5. Safety Boundaries

### Autocontrol Must Never

- Merge PRs or push to protected branches
- Deploy to any environment
- Delete repositories, branches, or files outside the bridge
- Modify external projects (Portal, AppKit, MSDK, Router) without an explicit task containing project name
- Bypass human-required decisions (merge, deploy, security, rollback)
- Run when STOP file exists
- Auto-continue after a high-risk report without confirmation
- Write to inbox/ without the manager's structured instruction

### Autocontrol is a Transport Layer

Autocontrol does not add intelligence. It does not decide what to do. It moves messages between the bridge and the manager. The existing MOS rules still govern what the manager decides and what the executor does.

---

## 6. Human Decision Gates

The following actions must always pause for human approval — autocontrol must not proceed past them without explicit confirmation:

| Gate | Trigger | Behavior |
|------|---------|----------|
| Merge | Report claims task is complete and recommends merge | Autocontrol prints merge request, waits for human `y/N` |
| Release | Any report mentioning release or deploy | Autocontrol stops, prints warning, waits for human resume |
| Production deploy | Any mention of production | Autocontrol stops, requires human to continue or abort |
| Security/permissions | Report mentions secrets, credentials, permissions | Autocontrol stops, escalates to human |
| Destructive rollback | Report mentions rollback, revert, data loss | Autocontrol stops, escalates |
| Cross-project conflict | Report mentions two or more external projects | Autocontrol prints conflict, waits for human |
| Opus budget | Opus model used or requested | Autocontrol logs, optionally pauses if configured |
| STOP present | STOP file detected | Autocontrol refuses to start or continue |

---

## 7. Autocontrol Modes

### Mode A — Manual Relay (Current State)

```
Human copies terminal output → pastes to ChatGPT
Human copies ChatGPT response → pipes to write-instruction
```

No automation. Full human visibility. Current implementation.

### Mode B — Local Relay (First Implementation)

```
bacp-bridge autocontrol [--once | --watch]
```

A local process that:
1. Reads the newest report from outbox/
2. Prints it to stdout with a manager prompt header
3. Waits for the manager's response on stdin (or from a file)
4. Validates the response as a valid instruction
5. Writes it to inbox/ using existing write-instruction logic
6. Optionally archives the source report
7. In `--watch` mode, loops until STOP or interrupt

The human still sees every message. The human still writes the instruction. But the human no longer copies, pastes, or runs separate commands — the relay handles transport.

### Mode C — API Relay (Future)

```
bacp-bridge autocontrol --api [--model <model>]
```

Same as Mode B, but instead of waiting for human stdin input, the relay sends the report to a model API (OpenAI, Anthropic) and writes the response directly to inbox/.

Human decision gates remain — if the response triggers a gate (merge, deploy, etc.), the relay pauses for human approval before proceeding.

---

## 8. Recommended First Implementation

**Implement Mode B first.**

### Why Mode B Before C

| Factor | Mode B | Mode C |
|--------|--------|--------|
| Risk | Low — human reviews every instruction | Higher — model writes instructions directly |
| Complexity | Low — stdin/stdout, no API | Higher — API client, auth, error handling |
| Value | Eliminates clipboard | Eliminates clipboard + thinking time |
| Dependencies | None | API key, model access, cost tracking |
| Human visibility | Full | Reduced (human reviews gates only) |

Mode B is the minimum viable improvement. It eliminates the clipboard without introducing API dependencies, cost, or reduced oversight.

---

## 9. Mode B Behavior

### Command

```
bacp-bridge autocontrol [--once]
```

### Flags

| Flag | Behavior |
|------|----------|
| (no flag) | Single pass: read oldest report, print manager prompt, wait for response, write instruction |
| `--once` | Same as no flag — explicit single-pass alias |
| `--watch` | Loop: poll every N seconds, process new reports as they appear, stop on interrupt or STOP |

### Flow (Single Pass)

```
1. Check STOP → refuse if present
2. Read oldest report from outbox/cli-to-manager/
3. Print report to stdout with MANAGER PROMPT header
4. Print instructions: "Paste your response below, then press Ctrl+D:"
5. Read stdin until EOF
6. Validate input as valid instruction JSON
7. Write instruction to inbox/ (reuses write-instruction logic)
8. Optionally archive source report
9. Log to bridge.log
10. Exit
```

### Manager Prompt Format

When a report is found, autocontrol prints:

```
━━━ MANAGER PROMPT ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  A new executor report is ready for review.

  Report: <filename>
  Project: <project>
  Created: <timestamp>

  Instructions:
  1. Review the report JSON below
  2. Write your instruction
  3. Paste it below and press Ctrl+D

━━━ REPORT ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
<report JSON>
━━━ END OF REPORT ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Paste your instruction JSON below, then press Ctrl+D:
```

### Watch Mode

In `--watch` mode, after processing a report, the relay sleeps for a configurable interval (default: 30 seconds) and checks for new reports. It loops until:
- STOP file is detected
- User presses Ctrl+C
- A report triggers a human decision gate that requires stop

### Default Poll Interval

30 seconds. Configurable via `--interval <seconds>`.

---

## 10. Mode C Behavior (Future Only)

### Command

```
bacp-bridge autocontrol --api [--model sonnet] [--interval 30]
```

### Additional Behavior

1. Same as Mode B, but instead of waiting for human stdin input:
2. Relay sends report JSON to model API (Anthropic or OpenAI)
3. Model response is validated as a valid instruction
4. Instruction is written to inbox/
5. Human decision gates are checked before proceeding
6. Audit log records: report ID, model used, latency, token count, cost estimate

### Out of Scope for Phase 0/1

- API key management
- Model selection UI
- Cost tracking dashboard
- Multi-model routing
- Streaming responses

---

## 11. Required Implementation Phases

### Phase 0 — Design (Current)

This document. Decisions:
- Mode B first
- Single-pass + watch mode
- No API dependency
- Human decision gates manual (not automated)
- Reuse existing write-instruction logic

### Phase 1 — Local Relay

Implement `bacp-bridge autocontrol --once`:
- Read oldest report
- Print manager prompt
- Read stdin response
- Validate and write instruction
- Archive source
- Log

Files modified:
- `scripts/bacp-bridge` (add ~120 lines)

### Phase 2 — Watch Mode

Implement `bacp-bridge autocontrol --watch`:
- Poll loop with configurable interval
- STOP detection
- Graceful interrupt handling
- Skip already-processed reports (track by filename)

### Phase 3 — Clipboard Helper (Optional)

- Auto-copy report to system clipboard (macOS: `pbcopy`, Linux: `xclip`)
- Print confirmation: "Report copied to clipboard"

### Phase 4 — API Relay

- Model API client
- Instruction validation from model response
- Human decision gates
- Cost tracking
- Audit log enrichment

---

## 12. Open Questions

| Question | Impact |
|----------|--------|
| Should API relay support OpenAI, Anthropic, or both? | Determines client library and key management. Both is ideal but adds complexity. Start with one. |
| Should the manager prompt be stored as a template file? | Would allow customization without code changes. Low priority. |
| Should autocontrol require an explicit allowlist of projects it can relay for? | Safety measure. Prevents autocontrol from processing unexpected project reports. |
| Should autocontrol run one report at a time (serial) or can it batch? | Serial is safer. Batching could cause conflicts. Start serial. |
| Should high-risk reports force an automatic STOP? | Currently they pause + prompt. For unattended Mode C, an auto-STOP on high risk may be appropriate. |

---

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Mode B first | Local relay without API | Lowest risk, immediate value, no new dependencies |
| Single-pass + watch | Both supported | Single-pass for manual use, watch for automated sessions |
| Reuse write-instruction logic | No duplicate code | Existing validation, filename generation, archive behavior applies |
| Human decision gates manual | Not automated | Too risky to automate gate detection in P0. Future: keyword-based auto-gate. |
| Serial processing | One report at a time | Prevents conflicts, simpler implementation, sufficient for current volume |
