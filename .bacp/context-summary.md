# BACP Context Summary — Compacted 2026-05-03

## Project State

| Field | Value |
|-------|-------|
| **Repository** | Bragi-Agent-Control-Plane |
| **Branch** | `agent/bootstrap-discovery` |
| **All time (UTC)** | 2026-05-03 ~07:00–12:30 |
| **Total commits** | 7 across current session |
| **Committed files** | 17 |
| **Bridge helper** | Phase 2 P0+P1 implemented (`scripts/bacp-bridge`) |
| **Control loop** | Verified — ISSUE→ACKNOWLEDGE→ACT→REPORT→VERIFY→DECIDE |

## Completed Commits

| Commit | Message | Files |
|--------|---------|-------|
| `131f18c` | Initialize Phase 1 bridge setup | 7 (docs, CLAUDE.md, README, gitignore) |
| `149dd6c` | Initialize Phase 1 bridge setup (discovery doc) | 1 |
| `b6d9690` | Document vault and messaging discovery | 1 |
| `c1681c8` | Design manager-control-first architecture | 1 |
| `26e2b73` | Align bridge with manager-control-loop | 6 (4 schemas + 2 docs updated) |
| `3fcd12c` | Verify manager control loop with live instruction | 1 |
| `987a8e8` | Design Phase 2 bridge helper | 1 |
| `12ddd5d` | Implement Phase 2 bridge helper P0 | 1 |
| *(current)* | Implement Phase 2 bridge helper P1 | *(in progress)* |

## Bridge Architecture

```
~/.bragi/agent-control-plane/
├── inbox/manager-to-cli/     → Manager instructions (written by bacp-bridge write-instruction)
├── inbox/human-to-cli/       → Human overrides
├── outbox/cli-to-manager/    → Executor reports (read by bacp-bridge next-report)
├── outbox/cli-to-human/      → Human-bound messages
├── archive/sent/             → Processed outbound
├── archive/received/         → Processed inbound
├── decisions/pending/        → Open decision requests
├── decisions/answered/       → Resolved decisions
├── logs/bridge.log           → Audit trail
└── STOP                      → Kill switch (human only)
```

**Schema files:** `schemas/manager-control/` (4 JSON schemas)
**Bridge helper:** `scripts/bacp-bridge` (Python 3 CLI, stdlib only)

## Implemented Commands

| Command | Phase | Purpose |
|---------|-------|---------|
| `status` | P0 | Show bridge health and queue counts |
| `next-report` | P0 | Display oldest pending executor report |
| `write-instruction` | P0 | Write validated manager instruction to inbox |
| `archive` | P0 | Move processed message to archive |
| `next-instruction` | P1 | Display oldest pending manager instruction |
| `decisions` | P1 | Show pending/answered decisions |
| `stop` | P1 | Activate STOP kill switch (with confirmation) |
| `resume` | P1 | Deactivate STOP kill switch (with confirmation) |
| `ack` | P1 | Archive a consumed instruction/report |

**P0+P1 total:** 9 commands

## Detailed Documents

| Document | What It Covers |
|----------|---------------|
| `docs/discovery-plan.md` | 10-pass discovery scope and methodology |
| `docs/chatgpt-cli-bridge.md` | Bridge architecture, schemas, phased plan |
| `docs/phase-1-bridge-setup.md` | Directory structure, filename convention, setup |
| `docs/manager-control-loop.md` | Manager authority, verification loop, 13 commands |
| `docs/phase-2-bridge-helper.md` | Bridge helper design, CLI commands, validation |
| `docs/control-loop-verification.md` | First live verification of manager→CLI control |
| `docs/discovery-pass-1-vault-and-messaging.md` | Vault structure, message protocol, flag system |
| `docs/discovery-pass-2-project-operating-models.md` | AppKit/Portal/MSDK/Router documentation, contracts |

## Open Risks

1. **Manual transport remains** — human copy/paste between CLI and ChatGPT is still required (Phase 4 solution)
2. **No automated polling** — CLI checks at conversation boundaries, not on timer
3. **Validation is manual (not jsonschema lib)** — catches structure but not deep type checking
4. **Router does not exist** — listed as managed project but has no repo
5. **MSDK uses GitLab CI** — separate CI system from Portal/AppKit (GitHub Actions)

## Next Planned Task

Implement Phase 2 bridge helper P1 commands:
- `next-instruction` — display pending manager instructions
- `decisions` — show pending/answered decisions
- `stop` — activate kill switch with confirmation
- `resume` — deactivate kill switch with confirmation
- `ack` — archive consumed messages after processing

## Do-Not-Repeat Notes

- **Never write to bragi-vault/messages/** — BACP has its own bridge
- **Never modify external projects** (AppKit, Portal, MSDK, Router)
- **Never create/delete STOP** — human only (bridge helper `stop`/`resume` is the approved mechanism)
- **Never use Opus** without explicit manager authorization
- **Never run discovery** without manager instruction
- **Never make outbound API calls** from the bridge (no GitHub, no ChatGPT API)
- **Never auto-archive** messages — explicit archive or ack only
- **Never delete files** — archive by move only, preserve originals
