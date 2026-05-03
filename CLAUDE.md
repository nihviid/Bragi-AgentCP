# CLAUDE.md — Bragi Agent Control Plane

**Phase:** Bootstrap / Discovery
**Do NOT write production code yet.** We are in the discovery phase. Only explore, document, and plan.

## Project Purpose

A local GitHub-connected control plane for managing multiple Bragi projects with AI manager/executor loops.

## Managed Projects

| Project | Location | Role |
|---------|----------|------|
| Router | TBD | API routing, message relay |
| Portal | `~/Desktop/Antigravity/Bragi-portal/` | Cloud control plane, orchestration |
| MSDK | `~/Desktop/Antigravity/msdk-ios/` | Native BLE transport plugin |
| AppKit | `~/Desktop/Antigravity/Bragi-AppKit/` | Native library, kits, runtime |

## Key Paths

- Knowledge vault: `~/Desktop/Antigravity/bragi-vault/`
- Memory vault: `~/Desktop/Antigravity/Obsidian_Vault/`
- Messages: `~/Desktop/Antigravity/bragi-vault/messages/`
- Platform docs: `~/Desktop/Antigravity/docs/`
- **Bridge directory: `~/.bragi/agent-control-plane/`**
- **Bridge setup: `docs/phase-1-bridge-setup.md`**
- **Bridge architecture: `docs/chatgpt-cli-bridge.md`**

## Current Phase: Discovery

Branch: `agent/bootstrap-discovery`
Plan: `docs/discovery-plan.md`

Execute each discovery pass sequentially. Each pass produces an artifact in `docs/discovery/`.

## Bridge Reporting Mandate

**All BACP activity reports through the Phase 1 manual structured bridge.** This is not optional.

**At every task boundary** (task complete, blocked, checkpoint, session end), the CLI MUST:
1. Write a BACP STATUS REPORT to `~/.bragi/agent-control-plane/outbox/cli-to-manager/`
2. Follow the filename convention: `TIMESTAMP~PROJECT~ROLE~PID~NONCE.TYPE.json`
3. Follow the status report schema defined in `docs/chatgpt-cli-bridge.md`
4. Append to `~/.bragi/agent-control-plane/logs/bridge.log`
5. Print the BACP STATUS REPORT at the end of output for human copy/paste

## Rules

- NEVER modify files in managed projects (Router, Portal, MSDK, AppKit).
- NEVER modify files in the bragi-vault.
- NEVER modify files in the Obsidian Vault.
- Read-only exploration only until design phase begins.
- All findings go in `docs/discovery/`.
- The synthesis document `docs/discovery-synthesis.md` is the final deliverable of this phase.
- **NEVER create or delete `~/.bragi/agent-control-plane/STOP`** — only the human may activate or deactivate the kill switch.
