# Manager-Control Tool Specification

**Bragi Agent Control Plane (BACP)**  
**Date:** 2026-05-03  
**Branch:** `agent/bootstrap-discovery`  
**Status:** Specification (implemented, consolidated)

---

## 1. Tool Purpose

A single CLI tool (`bacp-bridge`) that enables the ChatGPT manager to control the CLI executor through a local filesystem bridge. The manager decides what to do. The CLI executes. The tool handles all filesystem operations so the human never navigates directories or constructs filenames.

The entire system is one directory, one Python script, four JSON schemas, and a human clipboard.

---

## 2. Minimal User Workflow

```
1. CLI completes task → writes structured report to outbox/
2. Human runs:     bacp-bridge next-report
3. Human copies    terminal output → pastes to ChatGPT
4. Manager reads   report → writes instruction
5. Human copies    instruction → pipes to:
   echo '<json>' | bacp-bridge write-instruction
6. CLI runs:       bacp-bridge next-instruction
7. CLI acts on     instruction → reports back (goto 1)
```

That is the entire loop. Every command is one line. No directory navigation, no filename construction, no manual archiving.

---

## 3. Manager-to-CLI Instruction Flow

```
ChatGPT ──(human copy/paste)──► echo '<json>' | bacp-bridge write-instruction
                                   │
                                   ▼
                              ~/.bragi/agent-control-plane/
                              inbox/manager-to-cli/
                                   │
                              bacp-bridge next-instruction
                                   │
                                   ▼
                              CLI executor reads + acts
                                   │
                              bacp-bridge ack <id>
                                   │
                                   ▼
                              Archive/received/
```

The manager writes instructions. The CLI reads them. The CLI acknowledges when consumed. Instructions are never deleted — only moved to archive.

---

## 4. CLI-to-Manager Report Flow

```
CLI executor writes report to outbox/cli-to-manager/
         │
    bacp-bridge next-report
         │
         ▼
    Terminal output ──(human copy/paste)──► ChatGPT
         │
    bacp-bridge ack <id>
         │
         ▼
    Archive/sent/
```

Reports are never deleted. Archived reports remain as audit trail.

---

## 5. Command List

| Command | Reads | Writes | Purpose |
|---------|-------|--------|---------|
| `status [--json]` | All queues | nothing | Show bridge health and queue counts |
| `next-report [--json]` | outbox/ | nothing | Display oldest pending CLI report |
| `next-instruction [--json]` | inbox/ | nothing | Display oldest pending manager instruction |
| `write-instruction [--archive-source <id>]` | stdin | inbox/ | Write validated instruction from pipe. Optionally archives source report after write. |
| `write-report [--ack-source <id>]` | stdin | outbox/ | Write executor status report from pipe. Auto-fills id, direction, type, created_at, from, to. Validates required project_state fields. Optionally acks source instruction. |
| `decisions [--json]` | decisions/ | nothing | Show pending/answered decisions |
| `archive <id>` | any queue | archive/ | Move processed message to archive |
| `archive-all <queue>` | queue | archive/ | Bulk archive all pending messages in a queue (reports, instructions, or decisions) |
| `ack <id>` | any queue | archive/ | Archive + acknowledge consumed message |
| `stop` | nothing | STOP | Halt all bridge operations (confirmation) |
| `resume` | nothing | (remove STOP) | Reactivate bridge operations (confirmation) |

All commands are one line with optional flags. No config files needed.

---

## 6. File Queue Structure

```
~/.bragi/agent-control-plane/
├── inbox/manager-to-cli/     ← Manager instructions (write-only for CLI)
├── outbox/cli-to-manager/    ← Executor reports (write-only for manager)
├── archive/sent/             ← Processed outbound messages
├── archive/received/         ← Processed inbound messages
├── decisions/pending/        ← Open decision requests
├── decisions/answered/       ← Resolved decisions
├── logs/bridge.log           ← Append-only audit trail
└── STOP                      ← Kill switch file (present = halted)
```

Three flows share this tree:
- **Instructions**: inbox/ → CLI → archive/received/
- **Reports**: outbox/ → manager → archive/sent/
- **Decisions**: decisions/pending/ → manager → decisions/answered/

Files use the naming convention: `TIMESTAMP~PROJECT~ROLE~PID~NONCE.TYPE.json`

---

## 7. Message Schema

Four JSON schemas in `schemas/manager-control/`:

| Schema | File | Required Fields |
|--------|------|----------------|
| Manager Instruction | `manager-instruction.schema.json` | command, instruction, model_selection, project |
| Executor Report | `executor-report.schema.json` | what_was_attempted, what_succeeded, what_failed, project_state |
| Project State | `project-state.schema.json` | 13 fields: project, task, repo, tests, build, lint, PR, memory, decisions, risks, next |
| Decision Request | `decision-request.schema.json` | title, options, pros/cons, recommendation, deadline |

Validation is built into `bacp-bridge write-instruction` and `next-report`. Schema files are the source of truth — this document summarizes them.

---

## 8. Ack/Archive Behavior

- **No message is ever deleted.** All consumed messages move to archive.
- `ack` and `archive` are synonyms that move files to `archive/sent/` or `archive/received/` based on origin.
- Original filenames are preserved. On collision, an incrementing numeric suffix is appended.
- Cross-directory resolution: `ack` accepts a filename substring and searches inbox, outbox, and decisions/pending/.
- The archive is append-only. No compaction. No purge. Oldest entries never expire.

---

## 9. Kill Switch

A single file `~/.bragi/agent-control-plane/STOP` halts all bridge operations.

- **Create**: `bacp-bridge stop` (requires interactive `[y/N]` confirmation)
- **Remove**: `bacp-bridge resume` (requires interactive `[y/N]` confirmation)
- **Effect**: Most commands refuse to run when STOP is present
- **File content**: Human-readable reason, timestamp, resume instructions
- **Only the human may activate or deactivate the kill switch.** The CLI must never create or delete STOP.

---

## 10. Audit Log

All tool operations append to `~/.bragi/agent-control-plane/logs/bridge.log`:

```
2026-05-03T12:00:00Z status ~/.bragi/agent-control-plane ok
2026-05-03T12:01:00Z next-report ~/.bragi/agent-control-plane/outbox/... ok
2026-05-03T12:02:00Z write-instruction ~/.bragi/agent-control-plane/inbox/... ok
```

Each line: ISO timestamp, action, path (or root), result. No rotation, no truncation, no TTL.

---

## 11. Error Handling

| Scenario | Behavior |
|----------|----------|
| Invalid JSON on stdin | Reject with parse error, print validation messages |
| Missing required fields | Reject with list of missing fields |
| File not found | Print error, exit 1 |
| STOP file present | Most commands fail with "Bridge is HALTED" |
| Cannot write to directory | Print OS error, exit 1 |
| Unknown command | Print usage, exit 1 |
| Empty queue | Print "No pending messages", exit 0 |

All errors go to stderr. Normal output goes to stdout. Exit code 1 on failure, 0 on success.

---

## 12. Security Boundaries

| Boundary | Rule |
|----------|------|
| Network | The bridge makes zero outbound API calls. No HTTP, no webhooks, no cloud sync. |
| GitHub | The bridge never reads or writes GitHub. No `gh`, no API, no PR creation. |
| Filesystem | The bridge only touches `~/.bragi/agent-control-plane/`. No other paths. |
| Deletion | The bridge never deletes files. Archive moves only. |
| External projects | The bridge never reads or modifies Router, Portal, AppKit, MSDK, or bragi-vault. |
| Model access | The bridge is not a model. It does not call OpenAI, Anthropic, or any LLM. |
| Human control | The kill switch is human-only. The CLI may never create or destroy STOP. |

---

## 13. Minimal Implementation Roadmap

### Phase 1 — Bridge Setup ✓
- Directory structure, filename convention, documentation

### Phase 2 — Bridge Helper ✓ (P0 + P1 + P2 implemented)
- 10 CLI commands: status, next-report, next-instruction, write-instruction, decisions, archive, archive-all, ack, stop, resume
- `--json` output mode for scriptable consumption
- `--archive-source` flag for write-instruction
- `archive-all` for bulk queue archival
- Manual JSON validation
- Basic error handling
- Audit log

### Phase 2.5 — Consolidation (not started)
- Compact/remove discovery artifacts that drift from tool focus
- Update README for tool usage focus
- Ensure all existing docs reference tool spec, not discovery

### Phase 3 — Automated Polling
- Background daemon polls inbox/ at configurable interval
- CLI automatically picks up instructions without manual `next-instruction`
- Timer-based, not event-driven (no file watchers)

### Phase 4 — Direct API Integration
- ChatGPT manager writes directly to inbox/ (remote write capability)
- CLI reports directly to manager (remote read capability)
- Eliminates human clipboard entirely
- Requires careful auth boundary (local read, authenticated write)

---

## 14. What Is Explicitly Out of Scope

| Area | Status |
|------|--------|
| Project discovery | STOPPED. Never resume without explicit manager instruction. |
| Vault inspection | STOPPED. The bridge does not interact with bragi-vault. |
| Watcher health monitoring | REMOVED from active scope. Not a bridge concern. |
| Cross-project collaboration | STOPPED. The bridge controls the CLI, not external projects. |
| GitHub integration | OUT. The bridge does not push, PR, or review. |
| Dashboard / UI | OUT. The bridge is a CLI tool. Dashboard is a future concern. |
| API integration | OUT until Phase 4. No external API calls from the bridge. |
| External project changes | OUT. Never modify Router, Portal, AppKit, MSDK. |
| Model routing logic | OUT. The manager specifies model selection. The CLI obeys. |
| Automated decision-making | OUT. The CLI executes. The manager decides. |
