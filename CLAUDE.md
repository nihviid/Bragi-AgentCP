# CLAUDE.md — Bragi Agent Control Plane

**Current objective:** Build and operate the Manager-Control CLI bridge tool (`bacp-bridge`).
**Do NOT engage in project discovery, vault inspection, or cross-project work unless explicitly instructed by the manager.**

---

## Project Purpose

A local CLI tool (`scripts/bacp-bridge`) that enables a ChatGPT manager to control a CLI executor through a structured filesystem bridge at `~/.bragi/agent-control-plane/`.

The manager decides what to do. The CLI executes. The bridge is the only communication channel.

---

## Active Objective: Manager-Control Tool

**Branch:** `agent/bootstrap-discovery`
**Tool:** `scripts/bacp-bridge` (10 commands)
**Spec:** `docs/manager-control-tool-spec.md`
**Plan:** `docs/manager-control-tool-implementation-plan.md`

The tool is the only active work. Everything else is out of scope unless the manager explicitly instructs otherwise.

---

## Manager/Executor Protocol

The bridge uses these message queues in `~/.bragi/agent-control-plane/`:

```
inbox/manager-to-cli/    ← Manager writes instructions. CLI reads and acts.
outbox/cli-to-manager/   ← CLI writes reports. Manager reads and decides.
archive/sent/            ← Processed outbound messages (preserved forever)
archive/received/        ← Processed inbound messages (preserved forever)
decisions/pending/       ← Open decision requests
decisions/answered/      ← Resolved decisions
logs/bridge.log          ← Append-only audit trail
STOP                     ← Kill switch (human only)
```

The loop:
1. CLI completes task → writes structured report to outbox/
2. Human copies report from `bacp-bridge next-report` → pastes to ChatGPT
3. Manager writes instruction → human pipes via `echo '<json>' | bacp-bridge write-instruction`
4. CLI reads instruction via `bacp-bridge next-instruction`
5. CLI acts → acknowledges via `bacp-bridge ack <id>`
6. CLI reports results → writes new report to outbox/ (goto 1)

---

## Required Reporting Format

At every task boundary (complete, blocked, checkpoint, session end), the CLI MUST write a BACP STATUS REPORT to the bridge:

1. Write structured JSON to `~/.bragi/agent-control-plane/outbox/cli-to-manager/`
2. Use filename convention: `TIMESTAMP~PROJECT~ROLE~PID~NONCE.TYPE.json`
3. Include PROJECT_STATE_SNAPSHOT with: project, task, repo, tests, build, lint, PR, memory, decisions, risks, next
4. Append to `~/.bragi/agent-control-plane/logs/bridge.log`
5. Print the BACP STATUS REPORT at the end of terminal output for human copy/paste

---

## Scope Boundaries

| Permitted | Forbidden |
|-----------|-----------|
| Modify `scripts/bacp-bridge` | Modify Router, Portal, AppKit, MSDK |
| Create/edit `docs/` | Read/write `bragi-vault/` or `Obsidian_Vault/` |
| Read bridge directory files | Create or delete STOP (human only) |
| Run git commands in this repo | Make outbound API calls |
| Respond to manager instructions | Start autonomous discovery |

**Project discovery, watcher health monitoring, vault inspection, cross-project collaboration, GitHub integration, API integration, and dashboard development are ALL out of scope unless the manager explicitly instructs otherwise.**

---

## Key Paths

- Bridge: `~/.bragi/agent-control-plane/`
- Tool: `scripts/bacp-bridge`
- Tool spec: `docs/manager-control-tool-spec.md`
- Implementation plan: `docs/manager-control-tool-implementation-plan.md`
- Schemas: `schemas/manager-control/` (4 JSON schema files)
- Bridge architecture (superseded): `docs/chatgpt-cli-bridge.md`
- Manager control loop: `docs/manager-control-loop.md`
