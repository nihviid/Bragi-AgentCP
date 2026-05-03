# BACP — Bragi Agent Control Plane

A local CLI tool that enables a ChatGPT manager to control a CLI executor through a structured filesystem bridge.

```
Manager (ChatGPT) ←──(human clipboard)──→ CLI executor (bacp-bridge)
```

The tool is `scripts/bacp-bridge`. The bridge lives at `~/.bragi/agent-control-plane/`.

---

## What Problem It Solves

Without the bridge, manager↔CLI collaboration requires manual directory navigation, filename construction, JSON copying, and housekeeping. Every round-trip is error-prone and slow.

The tool reduces the loop to:

```
1. CLI reports  →  bacp-bridge next-report
2. Human copies  →  pastes to ChatGPT
3. ChatGPT responds  →  pipes to:  echo '<json>' | bacp-bridge write-instruction
4. CLI reads     →  bacp-bridge next-instruction
5. CLI acts      →  bacp-bridge ack <id>
6. Repeat
```

No directory navigation. No filename construction. No manual archiving.

---

## Available Commands

| Command | Purpose |
|---------|---------|
| `status [--json]` | Show bridge health and queue counts |
| `next-report [--json]` | Show oldest pending CLI-to-manager report |
| `next-instruction [--json]` | Show oldest pending manager-to-CLI instruction |
| `write-instruction [--archive-source <id>]` | Write validated manager instruction from stdin |
| `decisions [--json]` | Show pending/answered decisions |
| `archive <id>` | Move processed message to archive |
| `archive-all <queue>` | Archive all pending messages in a queue (reports, instructions, decisions) |
| `ack <id>` | Archive a consumed message |
| `stop` | Halt all bridge operations (confirmation required) |
| `resume` | Reactivate bridge operations (confirmation required) |

Flags:
- `--json` on display commands outputs machine-parseable JSON
- `--archive-source <id>` on write-instruction archives the source report after writing

---

## Quick Start

```bash
# Check bridge health
./scripts/bacp-bridge status

# Display oldest pending report (copy output to ChatGPT)
./scripts/bacp-bridge next-report

# Write manager response to inbox
echo '{"command":"continue","instruction":"...","model_selection":{...}}' | \
  ./scripts/bacp-bridge write-instruction

# Read pending manager instruction
./scripts/bacp-bridge next-instruction

# Archive a consumed message
./scripts/bacp-bridge ack <message-id-or-path>
```

Environment: `BACP_ROOT` overrides the default bridge path (`~/.bragi/agent-control-plane/`).

---

## Current Limitations

- **Human clipboard required.** The ChatGPT manager and CLI executor cannot communicate directly. A human must copy/paste between terminal and ChatGPT. (Phase 3/4 automation is future work.)
- **Manual validation.** JSON structure is checked, but full schema validation against the 4 schema files is not yet implemented.
- **No automated polling.** The CLI checks for instructions at conversation boundaries, not on a timer.
- **No dashboard.** Command output is terminal-only. A UI reads the same filesystem.

---

## Explicitly Out of Scope

| Area | Status |
|------|--------|
| Project discovery | Stopped. Not active unless explicitly instructed. |
| External project changes | Never modify Router, Portal, AppKit, MSDK. |
| GitHub integration | The bridge does not push, PR, or review. |
| API integration | No outbound API calls from the bridge. |
| Dashboard / UI | Terminal-only. Future consideration. |
| Watcher health monitoring | Removed from active scope. |
| bragi-vault or Obsidian Vault | Never read or modified by the bridge. |

---

## Documentation

| Document | What It Covers |
|----------|---------------|
| `docs/manager-control-tool-spec.md` | Full tool specification, workflow, schemas, security |
| `docs/manager-control-tool-implementation-plan.md` | Build plan, remaining gaps, acceptance criteria |
| `docs/chatgpt-cli-bridge.md` | Original bridge architecture (superseded by tool spec) |
| `docs/manager-control-loop.md` | Manager authority model, 13 commands, verification loop |
| `docs/phase-2-bridge-helper.md` | Bridge helper design (implementation complete) |
| `schemas/manager-control/` | 4 JSON schemas (project-state, executor-report, manager-instruction, decision-request) |
